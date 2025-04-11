---
title: Trying AI code review on OpenStack Nova
date: 2025-04-10 14:00:00 +0200
categories: [AI, code-review]
tags: [ai, code-review, nova, openstack]
hidden: true # remove it to publish the post

---

I ran an experiment using AI code review tools on OpenStack Nova patches.

## Goal

Use existing AI code review tools on incoming patches for the OpenStack Nova
open source project and review the validity, deepness, and completeness of
the provided review comments.

## AI tools

I've searched for AI code review tools that are:
* integrated to Github and capable of reacting to incoming pull requests
* freely available or have a trial period

The tools I selected are
* [Sourcery](https://sourcery.ai)
* [CodeRabbit](https://coderabbit.ai)

Setting up both tools does not take more than 5 minutes via their respective
websites. One can select and authorize the bots to access specific Github
repositories.

## Nova

[openstack.org](https://www.openstack.org/):
> OpenStack is an open source cloud computing infrastructure software project
> and is one of the three most active open source projects in the world.


[docs.openstack.org/nova](https://docs.openstack.org/nova/latest/):
> Nova is the OpenStack project that provides a way to provision compute
> instances (aka virtual servers). Nova supports creating virtual machines,
> baremetal servers (through the use of ironic), and has limited support for
> system containers. Nova runs as a set of daemons on top of existing Linux
> servers to provide that service.


Nova consists of more than 300 KLOC open source Python code. It is developed
via a strict code review process where two of the project maintainers need
to approve a patch before it can land. The code review happens via Gerrit
on [review.opendev.org](https://review.opendev.org).

## Patches to review

I selected a relatively small but involved feature consisting of multiple
commits including some preparation/refactoring work, unit testing,
functional testing, and documentation on top of the actual feature
implementation.

The original review is accessible at
[https://review.opendev.org/q/topic:bp/one-time-use-devices](https://review.opendev.org/q/topic:%22bp/one-time-use-devices%22).
The feature went through one major and a couple of minor iterations before
merged. So I selected two states of the patches for review by the
AI tools:

1. The [initial](https://github.com/gibizer/nova/pull/2) version represents the
state of the feature ready for review. This was the state where the Nova
maintainers first deeply reviewed the implementation. After this review, the
patch series went through multiple change and re-review cycles as usual.

2. The [final](https://github.com/gibizer/nova/pull/1) version represents the
last revision of the series that got merged to the upstream repository.

The available AI tools provide easy integration with Github, but not with
any standalone
[Gerrit](https://www.gerritcodereview.com) installation. Fortunately, Nova is
open source, so it is easy to set up a [fork](https://github.com/gibizer/nova)
of the [Nova's code mirror](https://github.com/openstack/nova).
Then the original patches can be fetched from Gerrit and pushed as a PR against
this fork. I did this manually for the trial, but it would not be super hard
to automate it.

As the AI bots are automatically reacting to incoming PRs I only had the task
to review the AI reviews (sic!).

## AI feedback on the initial version

Both tools directly edit the Github PR summary with its own summary of the PR.
This feels very intrusive but probably can be disabled/reconfigured.

Both tools generate a set of sections as part of their output like a summary
for the whole PR then a guide that breaks down the PR into reviewable
parts and finally a set of inline comments.

### Sourcery - PR Summary

The PR summary has multiple inaccuracies. For example:

> Introduce a 'one_time_use' tag for PCI devices that allows devices to remain
in a reserved state after allocation

It is incorrect as the device gets into reserved state (instead of remaining
in it) at allocation. The important effect of the new feature is that the
device remains in reserved state after the **deallocation** preventing
reallocation until the device is externally un-reserved. This is the core of
the "one time use" feature, so missing this point is pretty severe. The summary
is not just incomplete but actively misleading the reader.

The "Enhancement" section is mostly a repetition of the "New Features" section
only the second bullet point adds extra information.

### Sourcery - Reviewer's Guide

The summary here is just a re-formatted version of the PR summary. Nothing new.

The "File-Level Changes" table splits the changes into pretty arbitrary
groups. E.g. it creates a test group then fails to add every file with
test cases and adds test files to other groups.

Also, the description of the groups contains multiple incorrect statements:

1. > Introduces the HW_PCI_ONE_TIME_USE trait for one-time-use PCI devices.

    The tool totally misses the fact that the trait is an external dependency
    that is not introduced by this change, but a separate patch in a separate
    repo and that patch needs to land first before this can function. This
    happens even though this information is right in the commit message of the
    given patch.

2. > Adds a configuration option pci.report_in_placement to enable one-time-use
   > support.

    This is incorrect. The config option is added years before as part of a
    different feature this patch depends on. This patch only uses the
    existing config option.

### Sourcery - Inline comments

There were 4 inline comments so we can look at them one by one:

1. > issue (testing): Add a test case to verify the invalidation logic when no
   > allocations are present.

    This is a valid unit test coverage request.

2. > issue (testing): Add tests for invalid "one_time_use" values.

    This is also valid on the high level. However, looking deeper I found that
    the code that such new unit tests would exercise is an external utility
    [`oslo.utils.strutils.bool_from_string`](https://github.com/openstack/oslo.utils/blob/2f36253cb0cc5b81cfcd8cdf4144caa3beb22d33/oslo_utils/strutils.py#L142)
    that is already well covered with
    [unit tests](https://github.com/openstack/oslo.utils/blob/2f36253cb0cc5b81cfcd8cdf4144caa3beb22d33/oslo_utils/tests/test_strutils.py#L31).
    We should not duplicate such unit tests for all users of that utility.

3. > suggestion (code-quality): Merge nested if conditions (merge-nested-ifs)

    It suggests that instead of two nested ifs use a single if with the merged
    condition of the two original ifs. The generic rule or intention can be
    valid. But the actual proposal on how to execute that is problematic.

    The tool proposes the following changes:
    ```diff
    - if self.tags.get('one_time_use') == 'true':
    -     # Validate that one_time_use=true is not set on devices where we
    -     # cannot support proper reservation protection.
    -     if not CONF.pci.report_in_placement:
    -         raise exception.PciConfigInvalidSpec(
    -             reason=_('one_time_use=true requires '
    -                      'pci.report_in_placement to be enabled'))

    + if self.tags.get('one_time_use') == 'true' and not CONF.pci.report_in_placement:
    +     raise exception.PciConfigInvalidSpec(
    +         reason=_('one_time_use=true requires '
    +                  'pci.report_in_placement to be enabled'))
    ```
    The tool suggests dropping the code comment altogether which is
    wrong. Moreover having a code comment there is a reason why I would not
    suggest the actual merging of the ifs. So applying the suggestion blindly
    is dangerous. Merging the conditions might be acceptable.


4. > issue (code-quality): Use contextlib's suppress method to silence an error
   > (use-contextlib-suppress)

    This suggests a way to catch and drop exceptions with a context manager
    instead of with an `except: pass` construct currently used in the code.
    Technically this is correct, the two ways are equivalent and the suggested
    way is more explicit about the intention. However, the Nova code base never
    uses the suggested style. So this might be a style controversy among the
    maintainers.


### CodeRabbit - PR Summary

This tool provided a correct summary of the feature.

It slightly overemphasized one irrelevant documentation change:

> Updated the documentation glossary with the new "OTU" dictionary entry.


### CodeRabbit - Walkthrough

The summary here is correct but fairly high-level and generic.

The "Changes" table splits the changes into groups based on doc, test, and
implementation but also based on steps taken in the implementation. The
categorization seems correct and complete.

### CodeRabbit - Sequence Diagram

This tool provides multiple call graphs based on the impacted code paths.

Some actors on the diagrams use generalized names that are not used
in the Nova parlance like "Inventory System" instead of "Placement".

Otherwise, the sequence diagrams seem to be correct and feel helpful
to look at the change from a different, than the code diff, perspective.


### CodeRabbit - Inline comments

This tool generated 4 comments marked as "nitpicks" and no real inline
comments. I think it means the tool found no real actionable issue.

Still, we can look at the nitpicks:

1. > ... the code could be simplified by combining the nested if statements.

    Same nested if simplification comment as from Sourcery, but it does not
    fail to preserve the code comment like Sourcery.

2. > contextlib.suppress

    Same suggestion as from Sourcery to use a context manager over an
    `expect: pass` construct. Same comment here that the suggestion is a bit
    more explicit but probably a style question as well.

3. > Consider dropping the .keys() usage.

    Technically correct suggestion but a super nitpick.

4. > improve assertion clarity ... Using a more specific regex pattern would
   > make the test's expected behavior more explicit.

    The suggested test improvement is valid, but also a super nitpick.


## Human review feedback on the initial version

To be able to compare and contrast the AI review feedback with real human
review feedback I summarized my main review comments below. The full
review discussion can be read on
[Gerrit](https://review.opendev.org/q/topic:%22bp/one-time-use-devices%22).

1. I found a bug in the `_invalidate_pci_in_placement_cached_rps` that it
unnecessarily iterates on every inventory of the RP when invalidating it.

2. In a couple of places the code assumed that the inventory generation logic
runs against the existing inventory and modifies that. But that is not true.
The code runs in a way to always re-generate the full inventory and then if
that is different from the existing inventory do an update to Placement.
This means that the patch cannot detect when the `reserved` field is first set
from `0` to `total`. This only caused a minor bug in logging and dead code in
trait handling.

3. I suggested a set of refactorings to move the OTU handling closer to the
pre-existing structure and dataflow of Nova.


## AI feedback on the final version

I will only look at any new or changed review feedback for the final version
from the bots.

### Sourcery

The PR summary is still misleading about why we reserve. But the "Reviewer's
Guide" is improved and correctly states the logic.

The "File-Level Changes" degraded as it now only has a single group instead of
multiple groups, so it is not helpful at all anymore.

There are a couple of new inline comments:

1. > issue (testing): Missing assertion after creating the server
   > It's important to assert that the server is in ACTIVE state after creation
   > to ensure that the device allocation was successful.

     This comment is incorrect the `_create_server` used by the test already
     asserts the state of the server.

2. > suggestion (typo): Consider using consistent casing/hyphenation for
   > "one-time-use" throughout the document. The title uses "One-Time-Use"
   > while the body uses "one time use".

    This is a technically correct doc improvement.

3. > issue (code-quality): Replace interpolated string formatting with f-string

    This is purely a style question. Both ways are totally correct.


### CodeRabbit


The "Summary", "Walkthrough", and "Changes" sections are pretty similar to the
initial PR review.

The "Sequence diagram" is changed to reflect the refactored code. The actor
names are a lot better now. But unfortunately, there are incorrect actions on
the diagram:

> Request allocation of OTU device

The code is not built to react to the allocation requests, it is built on
the inventory generation logic that happens to run both periodically and
at certain events like VM boot. I'm pretty sure that this is a misunderstanding
about the changes in the `dev_spec.py` that "discovering" available devices
with OTU flag.

There is one "Actionable" comment provided:

1. > Improve Test Robustness by Verifying Server State

    This is the same mistake Sourcery made. They both miss the fact that
    `_create_server` already does this assert.


There were additional "nitpick" comments provided:

1. > The enhanced_pci_device_with_spec_tags method should also include the new
   > one_time_use tag, otherwise the tag won't be transferred to the device
   > object.

   This is completely wrong. The final version of the change intentionally
   removed the logic that adds the tag to the PCIDevice object as it implements
   all the logic based on the device_spec information instead.

2. > Improvement to assertion logic enhances test flexibility.
   > ..., there's a potential issue when inv_assertions contains keys not
   > present in rp_inv[rc], which could raise a KeyError.

   The reasoning in the local context is valid. The new assert helper in the
   test would fail with a `KeyError` on nonexisting inventory fields. However,
   as this is test code, in such case the test would fail and therefore the
   issue is caught. The bot suggests that instead of raising a `KeyError` raise
   an `AssertError` via `self.fail()`. I don't see the value in this change due
   to the extra logic it requires.

3. > The note correctly references the prerequisite configuration but uses both
   > "PCI-in-placement" and "pci-tracking-in-placement" which might be
   > confusing. Consider standardizing on one term throughout the
   > documentation.

   This is a valid doc improvement suggestion.

4. > there's a minor issue with an unused variable:

    This is valid. The variable is indeed unused and the code can be
    simplified.


## Conclusion

**The positives**

* Both tools are super easy to set up for Github repositories. Unfortunately
  the same level of integration does not exist for Gerrit today.

* Both tools were able to report some valid comments on the patches.

* Both tools were able to handle multiple commits within a single PR.

* The sequence diagram for CodeRabbit provides a different viewpoint of the
  change under review.

**The negatives**

* Both tools provided multiple invalid comments. Some of them appeared correct
  at first look, but digging deeper it turned out to be unnecessary or even
  harmful.

* Both tools made incorrect statements about what the proposed PR does or does
  not do.

* The valid comments the tools provided were all low value, mostly nitpicks or
  small non-consequential changes. Compared to the human review that discovered
  actual bugs, false assumptions, and suggested structural changes enhancing
  the structure of the code, supporting high-level readability and
  maintainability of the software.

* Both tools were verbose, but Sourcery was a lot more verbose repeating the
  same set of information multiple times in different formats and with
  different wording causing fatigue from the human reviewer.

The only real use I can imagine from the tools is the summarization capability
of a big PR / long patch chain. However, Sourcery made multiple factual
mistakes in their summary and therefore I would not trust it. CodeRabbit has
better summarization capabilities and the sequence diagram appears useful.
However, in the final version, the sequence diagram was also misleading. So I
would be hesitant to blindly trust it on a patch I did not review deeply
first.

CodeRabbit is pretty self-conscious and marks most of its inline comments as
nitpicks and indeed they are mostly nitpicks with occasional useful suggestions
hiding in the noise. Unfortunately the only review from the tool marked
actionable is totally wrong.

Overall I could not trust the reviews from either bots and I needed to manually
verify their statements about the patches. This took significant time even for
a patch that I thoroughly reviewed multiple times in the past. So even though
the bots had some correct suggestions the net value of their contribution was
negative as the correct suggestions were very low value and needed high human
effort to separate the correct suggestions from the incorrect ones.

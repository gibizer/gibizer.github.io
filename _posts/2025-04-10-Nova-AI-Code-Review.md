---
title: Trying AI code review on OpenStack Nova
date: 2025-04-10 14:00:00 +0200
categories: [AI, code-review]
tags: [ai, code-review, nova, openstack]
hidden: true # remove it to publish the post

---

I run an experiment of using AI code review tools on OpenStack Nova patches.

## Goal

Use existing AI code review tools on incoming patches for the OpenStack Nova
open source project and review the validity, deepness and completeness of
the provided review comments.

## AI tools

I've searched for AI code review tools that are:
* integrated to github.com and capable of reacting to incoming pull requests
* freely available or have a trial period

The tools I selected are
* [Sourcery](https://sourcery.ai)
* [CodeRabbit](https://coderabbit.ai)

Setting up both tools does not take more than 5 minutes via their respective
websites. One can select and authorize the bots to access certain github
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


Nova consists of more than 300 KLOC open source python code. It is developed
via a strict code review process where two of the project maintainers need
to approve a patch before it can land. The code review happens via gerrit
on [review.opendev.org](https://review.opendev.org).

## Patches to review

I selected a relatively small but involved feature consisting of multiple
commits including some preparation / refactoring work, unit testing,
functional testing, and documentation top of the actual feature implementation.


The original review is accessible at
[https://review.opendev.org/q/topic:bp/one-time-use-devices]
(https://review.opendev.org/q/topic:%22bp/one-time-use-devices%22).
The feature went through one major and a couple of minor iterations before
merged. So I selected two states of the patches for review by the
AI tools:

1. The [initial](https://github.com/gibizer/nova/pull/2) version represents the
state of the feature ready for review. This was the state where the Nova
maintainers first deeply reviewed the implementation. After this review the
patch series went through multiple change and re-review cycles as usual.


2. The [final](https://github.com/gibizer/nova/pull/1) version represents the
last revision of the series that got merged to the upstream repository.

The available AI tools provide out of the box integration with
[github.com](github.com), but not with any standalone
[gerrit](https://www.gerritcodereview.com) installation. Fortunately Nova is
open source, so it is easy to set up a [fork](https://github.com/gibizer/nova)
of the [Nova's code mirror](https://github.com/openstack/nova).
Then the original patches can be fetched from gerrit and pushed as a PR against
this fork. I did this manually for the trial but it would not be super hard
to automatically do this.

As the AI bots are automatically reacting to incoming PRs I only had the task
to review the AI reviews (sic!).

## Analysis of the review comments

### AI feedback on the initial version

Both tool directly edits the github PR summary with its own summary of the PR.
This feel very intrusive but probably can be disabled / reconfigured.

#### Sourcery

##### PR Summary

The PR summary has multiple inaccuracies. For example:

> Introduce a 'one_time_use' tag for PCI devices that allows devices to remain
in a reserved state after allocation

It is incorrect as the device get into reserved state (instead of remaining in)
at allocation. And the important effect of the new feature is that the device
remains in reserved state after the **deallocation** of the device preventing
the re-allocation of the device until the device is externally un-reserved.
This is the core of the "one time use" feature, so missing this point is pretty
sever. As it is not just incomplete but actively misleading the reader.

The "Enhancement" section is mostly a repetition of the "New Features" section
only the second bullet point adds extra information.

##### Reviewer's Guide

The summary here just a re-formatted version of the tool's PR summary nothing
new. The File-Level Changes table splits the changes in pretty arbitrary
groups. E.g. it creates a test group then fails to add every files with
test cases and adds test files to other groups.
Also the description of the groups contain multiple incorrect statements,
e.g.:

> Introduces the HW_PCI_ONE_TIME_USE trait for one-time-use PCI devices.

The tool totally misses the fact the trait is an external dependency that is
not introduced by this PR but a separate patch in a separate repo and such
patch needs to land first before this can function. Even though this
information is right in the commit message of the given patch and in the PR
summary as well.

> Adds a configuration option pci.report_in_placement to enable one-time-use
> support.

This is incorrect the config option is added years before as part of a
different feature this patch is depending on. This patch only uses the existing
config option.

##### Inline comments



#### CodeRabbit

### Human review feedback on the initial version

### AI feedback on the final version

#### Sourcery

#### CodeRabbit


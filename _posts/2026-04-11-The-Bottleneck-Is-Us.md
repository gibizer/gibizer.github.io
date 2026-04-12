---
title: The Bottleneck Is Us
date: 2026-04-12 19:45:00 +0200
categories: [AI, code-review]
tags: [ai, asdlc, code-review, nova, openstack]
---

Coincidentally, I wrote a [post]({% post_url 2025-04-11-Nova-AI-Code-Review %})
about AI Code Review a year ago. Many things changed since then out
there about technology, about the shape of the hype, but also many things
changed around me and in me. I expect this trend will continue in
hard to predict ways, so here in this post I don't want to focus on what
changed, but rather try to use the noise of those changes to look at what
stayed the same. I assume that those are somewhat more important than what is
constantly changing.

## Managing knowledge

My daily life as a Software Engineer boils down to creating new knowledge and
organizing existing ones in useful ways both within a certain domain, the
product I'm shipping, and also in the general sense about how to do my job
efficiently and sustainably, the craft.

It can be a customer escalation where I'm figuring out what the customer did,
what they expected to happen and what actually happened. Then reflecting that
on what the software is supposed to do and finding where the gaps are in the
understanding of the humans, and gaps between the intended behavior of the
software and the actual one.

Also it can be code review of a new feature where I'm loading context on the
original requirements, building a view on what is the intention of the code
change and how that intention is implemented. Then reflecting on the
gaps between the original requirements and the intention of the change, the
what, and the gaps between the intention of the change and the actual change
in behavior of the software caused by the proposed code modification, the how.

All this work builds understanding in me about what gaps exist, which need to
be closed and how to close them. Then this understanding, my internal context,
needs to be communicated, distilled down to knowledge and sent across to
others. For example by responding to the escalation pointing out the
misunderstanding how the system should be used or replying in the code review
about an edge case the author has missed leading to a potential bug.

In the last year around me everybody started writing prompt libraries, AGENT.md
files, skills, personas, agentic workflows, etc. These are soft instructions to
a tool to achieve predefined goals. This is why we are writing them. But it is
also knowledge. Without sparking a debate about whether the tool builds
understanding out of this knowledgebase or not, I believe humans still do.
Until we remove all the human decision making from software development, the
building of human understanding is important if we want to make good decisions.
Therefore we should keep these instructions understandable, maintainable,
verifiable for and by humans. For example when I write a
how-to-implement-a-new-Nova-feature skill for an LLM tool, I am basically
describing my own understanding of how to do that as a human. When others want
to refine or reuse that skill later they need to understand that knowledge,
the original intention, and the strategy expressing that intention in the text
I wrote in the skill. This is all human-to-human communication and knowledge
management. (...until we remove the last human from the loop.)

Sure this knowledgebase is in dual use. The tool grinds on it too. Therefore it
needs to be "good" for the tool too. I'm not the right person to make
suggestions about how to make it "good" for the tool, at least not yet.
Also there are multiple tools and they are changing rapidly, so I'm skeptical
of the usefulness and shelf life of such suggestions in general. But compared
to tools, humans change a lot slower.

Even though the tools are changing, the principles of managing human knowledge
do not change much. So I will try to use my past experience on how to build
understanding and how to organize knowledge **for humans** in the era of
Agentic Software Development Lifecycle (ASDLC). Whatever that buzzword really
means or will mean 6 months from now.

## Verifying assumptions

One way to build new knowledge is by creating a hypothesis and then trying to
prove or disprove it. I think humans are a lot better at making an assumption
and running with it until somebody else or something else invalidates that
assumption than making good, e.g. testable, hypotheses and doing the legwork
of verifying them before accepting them. So for me it makes sense to point out
hidden, unconscious assumptions to encourage humans to think more about them as
hypotheses and start testing them.

This can be as simple as asking a question in a code review: have you
considered case X? Maybe you assumed case X cannot happen. But have you
actually done the work of proving that it cannot?

But also it can be as complicated as talking about what is a better prompt,
skill, or agentic workflow. For example we assume that we have a big context
window to work with and that window will grow in the future indefinitely.
Therefore we can simply shovel more and more data into that context window of
the LLM and expect better results. I think this is just an assumption, not a
tested hypothesis. Also, I suspect that even if the context window grows
indefinitely in the future (which is unlikely as we are limited by physics
and economics), still LLMs are "smart" due to "attention". If you have more
data in the context window it does not mean you have more overall attention
available to find the important pieces from that data
[1](https://www.morphllm.com/context-rot).

I think that as we are building our ASDLC we are running on a long list of
assumptions. We don't have a well-defined way to make them explicit, or even
better, transform them to hypotheses and then test them. One simple example of
this is when we write new prompts, skills, personas for the LLMs or propose
changes to existing definitions. How can we decide which version of a
definition is better, more useful, produces better results?

I see two practical ways.

One is building an evaluation harness for our ASDLC. I would approach it from
an end-to-end, black-box testing perspective as much as possible as the tools
and interfaces are rapidly changing. The system under test would be a specific
ASDLC workflow, the change under test would be a new skill, persona, prompt, or
change in an existing one. Then the test cases would be in the form of
common tasks we expect to be supported by the affected part of that workflow,
like implementing a specific new feature. And the asserts would be on the level
of ensuring that the task is done and done with good quality. So for a new
feature implementation task that would be a test suite that ensures
functionality, and a code review to assess quality and maintainability of the
generated code. As we have many different workflows and possible tasks
supported by those workflows, building such an evaluation harness is a huge
task. So we have to start as early as possible. ;)

While this harness is being built and in its infancy, I offer a second way as
well.

We can make a probably wrong but temporarily useful assumption that if the
instructions in the ASDLC, e.g. the skill to create a new feature, make sense
to a group of humans that did such feature building before many times, then it
will be useful for the LLM Agent too. So we can apply that human judgement for
creating and improving skills, personas, or prompts, in the form of a review.

So create a group of knowledgeable humans and ask them to judge the skill, or a
change in that skill, from a human perspective. If they agree that it is good,
include it into the knowledgebase, if they have suggestions how to improve it,
then iterate on the proposal. This approach will be based more on feelings
than on facts. We know that human feelings are dangerous to trust in software
(probably in other aspects of life too). I think most of us fell into the trap
before where we felt that a change is good enough and *that* extra test is not
needed. Then found out the hard way that the change was broken exactly *that*
way we ignored in overconfidence. This is why we ask questions in code review
and this is why we want to base our judgement also on facts, passing test
suites more than on assumptions and feelings. These facts will come from the
first approach once that harness is running, but we cannot prevent progress in
the meantime.

## Code review

There are many workflows in an ASDLC. Specification creation, bug triage, code
generation, documentation writing, and reviews. I know I cannot tackle all
these at once. I don't have that much human attention and context window. :)
So I need to select one area, maybe one workflow that seems most influential in
my worklife and focus on that first.

While discussing the push for applying agents in software development, I heard
many times that we are doing this because others, our competitors, out there
are doing this and if they ship faster then we will lose customers. Besides
that this is circular reasoning, our competitors probably say the same, and
this is a positive feedback loop that pushes us towards instability, the
message is clear. Ship faster or else. So I looked at my worklife to see what
is the bottleneck for me and my immediate team, what prevents us from shipping
faster. It only makes sense to optimize the bottleneck if we want to increase
throughput.

I'm one of the maintainers of the sizeable open source project OpenStack.
Similar to other such projects we have a strong review culture. This not only
ensures the quality of the code added to the project, but also creates the
possibility for the maintainers to keep an understanding
of the codebase in their heads and share that with new contributors. After
15+ years we are still in the lucky situation that we have more contributions
coming in than what the maintainers can review in a timely manner. I'm pretty
sure that with LLM-based code generation the amount of code contribution will
continue to grow in the near future. So code review is the bottleneck if our
goal is to ship faster.

(I won't go into details now, but going faster is stupid without going in the
right direction... This is a topic of another post maybe in the future.)

So let's focus on the code review workflow in our ASDLC. How do we establish
and improve that agentic workflow? Code review can be considered as a way of
verifying assumptions. So we can use the two suggestions from the previous
section. Humans can review the review skill in the ASDLC, and we can create an
evaluation framework, a test suite, around the review skill.

Hm "reviewing the review skill", so meta. And a clear source of additional
review load on the maintainers. But at least we are already experts on
reviewing so nothing new here.

Then "a test suite around the review skill". What would that look like?
A test case would consist of a review task and a set of expectations on the
response of the Agent executing the review task by using the review skill under
test.

A review task is a commit or a series of commits, with the
relevant context provided, like the specification, state of the codebase, and
even the previous versions of these commits and the related code review
discussions leading to the current version.

The asserts are pretty complex. They need to cover that
* the Agent found all the known problems in the commits under
  review, but did not hallucinate non-existing problems.
* the comments are understandable and actionable by the author of the commits.
* the Agent attached proper severity for the findings (i.e. nit vs a blocker).
* the language of the comments is appropriate, non-offensive, allows reasonable
  pushback, invites cooperation

I'm not sure how many of these asserts can be automated eventually, but given
the fact that there are already benchmarks out there for Agents I'm hopeful.
Anyhow, we need to run these tests manually first, look for patterns and try
out tools. This is work to be done as well as future posts to write about.

All in all, if we need a good code review skill in our ASDLC, that needs a lot
of test cases and even more human code review in the loop to maintain and
improve the skill and in parallel the test suite for the skill. This is
additional workload, at least initially, that works against our original goal
to improve our speed via our code review capacity on upstream code
contributions.

## Human in the loop

I have a general feeling that the human is the bottleneck in all this. So
if the goal remains to be speed then the assumption that the human is in the
loop will be challenged. If that assumption is removed, e.g. by replacing all
humans above with another Agent, then most of what I wrote above is not
applicable any more. If there are no humans there then we don't have to care
about the human understanding of the review skill. That will be purely for the
Agents. It does not have to be English, it doesn't even have to be text. And
pooof knowledge disappeared in an Agentic black hole.

(Actually black holes are more complicated than that, for the outside observer,
us, the knowledge will be slowly smeared on and fade into the event horizon...)

Maybe, once more, we should think about whether shipping faster is the right
incentive going forward.

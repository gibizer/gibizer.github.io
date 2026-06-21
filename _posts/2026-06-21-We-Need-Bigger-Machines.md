---
title: We Need Bigger Machines
date: 2026-06-21 17:45:00 +0200
categories: [AI, code-review]
tags: [ai, asdlc, code-review]
---

After 20 years, I re-read The Hitchhiker's Guide to the Galaxy books.
It was cozy and nostalgic, exactly what I expected. Then, weeks later, one
morning my overslept brain made a surprising association.

> “All right,” said Deep Thought. “The Answer to the Great Question…”<br>
> “Yes…!”<br>
> “Of Life, the Universe and Everything…” said Deep Thought.<br>
> “Yes…!”<br>
> “Is…” said Deep Thought, and paused.<br>
> “Yes…!”<br>
> “Is…”<br>
> “Yes…!!!…?”<br>
> “Forty-two,” said Deep Thought, with infinite majesty and calm.

...

> “I checked it very thoroughly,” said the computer, “and that quite
> definitely is the answer. I think the problem, to be quite honest with you,
> is that you’ve never actually known what the question is.”

Then, of course, the folks who got the answer needed to build an even bigger
machine to find out what the question *really* is. (They failed. And then later
theorized that the question and the answer cannot even co-exist in the same
universe. Dim prospects.)

But what if they could look at the thought process of the machine
calculating "forty-two"? Would that reveal at least some hints about the *real*
question that is being answered? Sure, it would, but the machine thought for
seven and a half million years, so looking through such a long chain of
reasoning would not be possible for simple, mortal humanoids. So they spent
time and resources, and in the end gained almost nothing. Wouldn't it have been
better to use that time and those resources to try things out by themselves
and see if it helped?

This feels eerily similar to some of my interactions with LLMs. I have a
problem and a vague understanding of what I need to solve it. Then I ask a
question. The machine thinks for a while, sometimes long enough that I just go
on with my life while it churns. Then I get an answer; in most cases, I get
three different possible answers to choose from. And even if one of those
answers seems to be the correct one when I use it, i.e. "It works for me", I
have this feeling that I'm missing something deeper. I don't know how the
answer was found. Therefore *I got no long-term benefit* from solving this
problem. Sure, I solved an immediate problem, maybe I even generated a piece of
revenue for the company. But I did not really learn in the process. So I won't
be able to construct a solution for an adjacent problem on my own next time.
But more importantly, I did not get an insight about the generic nature of the
problem that would help me *avoid* similar problems in the future. So I spent
time and resources, and in the end gained almost nothing.

I still remember studying for exams during my university years. For me, the
most useful way to increase my knowledge was to try to apply what I "learned".
For example, not just read about the ways to solve a particular class of
problems, but actually sit down and try to solve a couple of examples by
myself. Just because the textbook showed me the solution, or even showed me the
steps to reach that solution, I did not gain a deep enough understanding to do
it alone. The actual effort spent on trying, and mostly failing, helped me
really learn. Until I had failed, I did not know what part of the topic was
unclear, where the missing steps were. Trying is an effective tool for moving
from the helpless unknown-unknown territory to the manageable known-unknown
state.

I think this is why companies behind these LLMs desperately try to signal
that coding is a solved problem (by LLMs, obviously). Otherwise it does not
make sense to apply LLMs for coding. If maintaining code is not a fully solved
problem, and I still have to read the generated code and decide if it is good
enough for me to maintain, then it is actually counterproductive to rely on
code generation in general, as I lose the benefit of building lasting
understanding of the codebase. It is a lot easier to maintain something that
you understand.

Sure, one can say: hey, you had to review and eventually accept code before
without writing it yourself. Yes, I did and still do today. But I can rely on
my past experience writing similar code. Also, in the past, I could trust the
author of the patch to understand the change because they went through the
exercise of actually writing the code.

Of course, if in the future I never need to read code again, then I can just
treat code as yet another layer of good abstraction that I don't have to care
about. Similarly to how much I don't care about x86 machine code when I write a
for loop in Python.

We are in a messed up transition period without even knowing if the transition
will ever fully happen. What I know is that these days, I very much still need
to read and review code. So maybe we just need to build bigger machines...

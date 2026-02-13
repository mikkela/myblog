---
title: "Text That Runs"
date: 2026-01-19
draft: false
tags:
  - compilers
  - programming-languages
  - oberon
  - rust
  - systems
description: "A café conversation about compilers, interpreters, and why the first thing you build is not the compiler itself."
---

Programming languages are just insanely fascinating, Lucy said, taking a sip of her coffee.

Lucy, Sancho and Merlin were sitting at their regular café. The napkins were thin, the coffee strong, the table already cluttered with half-finished thoughts.

But how do they actually work? she continued.
How can a program run at all? It’s just text.

Someone has already translated that text into something the machine can understand — using a compiler, Sancho said.

Or rather, into something that can be *misunderstood in a controlled way*. Merlin added.

That sounds dangerous.

All understanding is.

---

## Translating Meaning

So is a compiler just a translator?

No. A translator preserves meaning.
A compiler restructures it.
Merlin pointed out.

And sometimes throws half of it away, if it only makes things slower.
Sancho nodded.

Okay… then what is an interpreter?

An interpreter delays commitment.
The meaning of the program is decided step by step.
No decision is taken in advance.

Merlin took a sip of coffee.

So an interpreter is more flexible?

More honest.

And usually slower.

Lucy exhaled softly.
This is hard.
What would it look like if you drew it?

A napkin appeared. A pencil followed. Lines and boxes began to form.
![Napkin drawing of a compiler and an interpreter](napkin.png)
---

## Time, Memory, and Failure

But programs behave the same way, right?

Only if you ignore time, memory, and failure.

Which the machine never does.

So when would I choose one over the other?

When you know which mistakes you can afford to make early.

Or when you know how many milliseconds you’re willing to lose.

What about errors?
Why can compilers catch them earlier?

Because a compiler assumes the future.
An interpreter waits to see it.

That feels… optimistic.

It’s expensive optimism.

---

## Wanting to Build

Lucy stared at the napkin for a moment.

Okay.
I think I get it — at least a little.
But I don’t just want to understand it.

She paused.

I want to build one.

A compiler?

Yes.
Something real. Something that runs.

That is an excellent way to discover which parts you do not yet understand.

And a reliable way to suffer.

What if we picked something small?
Simple. Clean.

Small languages are rarely simple.
They are merely honest about their complexity.

But some are more polite than others.

Lucy looked at Sancho.
You once mentioned Oberon.

Minimal syntax. Clear semantics.
Designed to be compiled, not argued with.

Oberon is a language that assumes you mean what you write.
No more. No less.
Merlin agreed.

Could we compile it?

Of course.
The interesting question is *to what*.

What about the Oberon RISC processor?

There was a pause.

Now that…
That would be a proper target.

A closed world.
A machine whose assumptions are known.

And we write the compiler in Rust?

Memory safety. Explicitness.
Good tools. No illusions.

A modern language compiling an honest one
for a machine that tells no lies.

---

## Before the Compiler

So where do we start?

With a lexer.
Always with a lexer.

And with a decision.
Will you preserve the language —
or reinterpret it?

Lucy thought for a moment.

I think…
I want to understand it by staying faithful.

Then your compiler will be strict.

And unforgiving.

That’s fine.

She smiled.

Mistakes should hurt early.

Then you have already chosen.
You are building a compiler —
not just a program.

The napkin was full now. The coffee had gone cold.

---

## The Program Before the Compiler

All right.
If we’re really doing this — building a compiler —
what’s the first thing we write?

Not the compiler.

Of course not.

We write the program that *pretends* to be a compiler.

That sounds suspicious.

It is correct.
Every serious system begins as a lie that gradually becomes true.

So… what does that program do?

It starts.
It takes arguments.
It reads files.
It fails gracefully.

That’s it?

That’s already more than most programs manage.

---

## Contracts and Reality

So we begin with the command line?

Yes.
Because that’s how users tell you what they want —
and how they tell you you’re wrong.

The command line is a contract.
It defines what you promise to understand.

Input file, output file…

Maybe flags.
Maybe a target later.
But not today.

Restraint is a form of design.

And the source file —
we just read it as text?

As bytes first.
Text comes later.

Why?

Because files lie.
About encodings. About endings. About size.

Reality always enters through the smallest cracks.

---

## Errors as Narratives

Okay…
But when something goes wrong —
how do we explain *where*?

We count.
Lines. Columns. Offsets.

That sounds tedious.

It is.
Which is why we do it early.

Errors are not messages.
They are narratives.

Narratives?

They tell the user what the compiler believed,
what it encountered,
and why the two cannot coexist.

So every error needs a position?

A span, preferably.
Start and end.

Even before we understand the language?

Especially before.

Meaning is fragile without location.

---

## Failing Well

This feels like a lot of scaffolding.

Good.
Scaffolding keeps you from falling.

And reminds you that the structure is not yet the building.

So the first version of our compiler…

Does nothing useful.

Except start, read a file,
and complain intelligently.

Exactly.

A program that knows how to fail—

Lucy nodded.

I like that.
It feels… respectful.

To the user?

To the future.

They sat quietly for a moment.

—is already honest.

---

## Next

In the next step,
we will teach it to understand structure.

Not yet.

Not yet.

## Links
- [Oberon RISC](https://people.inf.ethz.ch/wirth/FPGA-relatedWork/RISC-Arch.pdf)
- [Oberon RISC Emulator](https://github.com/pdewacht/oberon-risc-emu)
- [Oberon System](https://people.inf.ethz.ch/wirth/ProjectOberon/UsingOberon.pdf)
- [Oberon Language](https://people.inf.ethz.ch/wirth/Oberon/Oberon07.Report.pdf)
- [Project Oberon](https://people.inf.ethz.ch/wirth/ProjectOberon/PO.System.pdf)
- [Rust Programming Language](https://www.rust-lang.org/)
- [Compilers: Principles, Techniques, and Tools](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools) (the "Dragon Book")
- [Crafting Interpreters](https://craftinginterpreters.com/) (a book about building interpreters, which can be a good starting point for understanding compilers as well)
- [Source Code](https://github.com/mikkela/oberon-compiler/tree/texts-that-run)

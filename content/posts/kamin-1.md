+++
title = 'Kamin Interpreters - Introduction'
date = 2024-11-11T09:27:00+01:00
draft = false
genres = ['interpreters', 'programming languages']
tags = ['scala']
+++

Ever since I got my hands on my first computer (a Commodore 64 in the mid-80s), I’ve always loved programming. At 
first, it was because I had no idea what else one would use such a machine for, and later because it became a tool 
for creative expression. Turning ideas and thoughts into a program feels, for me, like a process similar to the 
release others might find in painting, music, or ceramics—though, admittedly, without quite the same fascination 
from others!

A key part of this expression is the programming language itself. If it weren't for the fact that there are already 
too many programming languages, I’d love to add another one to the mix. Realistically, I’ll have to settle for 
studying, learning, and working with those that already exist. ["The Pragmatic Programmer"](https://en.wikipedia.org/wiki/The_Pragmatic_Programmer) recommends learning a new programming language from time to time—not just 
reading about it but actually using it. The one on my list is Scala. I've read a fair bit about it, and now I'm 
trying to get a better feel for it by building a few projects. The first project will be to implement a series of 
interpreters for various programming languages (or rather, selected parts of those languages). These are languages 
defined in Samuel Kamin’s book ["Programming Languages – An Interpreter-Based Approach"](https://kamin.cs.illinois.edu/pubs.html).

In his book, Kamin explores different programming concepts through eight snippets from existing languages. The book 
is from the early 90s, so the languages are, by now, somewhat outdated—but if you can live with that (or even find 
it fun), I think it’ll be an exciting exercise to build interpreters for these languages in Scala. Starting entirely 
from scratch, using nothing but a command-line tool, with the added twist that the interpreter should be able to 
interpret all eight languages. The goal is not only to create each interpreter but also to do it in a way that keeps 
the implementation general and adaptable to the unique aspects of each language.

# The Languages
The languages in Kamin's book are squeezed into the same basic, parentheses-based notation 
(inspired by [LISP](https://en.wikipedia.org/wiki/Lisp_(programming_language))). Every element, 
except the very simplest, is wrapped in a set of parentheses as illustrated below.
```
(if (< x 1) (+ x 2) (+ x 3))
```
This setup makes it easy to write parsers for the languages, but it doesn’t exactly do much for readability. 
You have to be quite familiar with the notation to fully understand what’s actually going on. However, if you’re 
willing to put in the effort, it becomes relatively easy to compare the individual languages. 

The langauges are:
* Basic
* LISP
* APL
* Schema
* SASL
* CLU
* Smalltalk
* Prolog

A palette of programming languages from the late 80s—this should be interesting! Down the rabbit hole we go!

# Basic
We start with Basic, which defines the foundational language that the rest builds upon. 
It consists of a series of constructs, some simple
```
-> 3
3
-> (+ 4 7)
11
-> (set x 4)
4
-> x
4
-> (* x x)
16
-> (print x)
4
4
```
The interpreter will always return the evaluation of the last expression.

I can combine expressions into larger ones
```
-> (set y 5)
5
-> (begin (print x) (print y) (* x y))
4
5
20
-> (if (> y 0) 5 10)
5
-> (while (> y 0)
>   (begin (set x (* x x)) (set y (- y 1))))
0
-> x
128
```
Finally functions can be defined and called
```
-> (define +1 (x) (+ x 1))
+1
-> (+1 4)
5
-> (define mult (x y) (* x y))
mult
-> (mult 5 6)
30
```

Please note, most of the examples are copied from Kamin's book. It also contains other examples.

The variables are bound in an environment that uses a simplified Pascal scope. There are two levels of scope: a global 
scope (which always exists) and a local scope for function calls, where the function's parameters are bound 
in the local scope. If a local scope is available, variables will first be checked against this scope, and then 
against the global scope. This applies to both lookups and updates.

That's enough for this time. The next post will cover the reading and conversion of Basic from text into the 
internal syntax tree.

---
layout: post
title: "The forgetful prompter's guide to building electric cathedrals"
date: 2025-02-07
categories: [Development, AI]
tags: [prompter, architecture, development, zig, ai]
description: "A look at using AI prompters for software architecture, comparing a year-apart rewrite of the same project."
toc: true
---


- [PART 1: What's this about?](#part-1-whats-this-about)
  - [The Console 1 demo application - version 1.](#the-console-1-demo-application---version-1)
  - [Written in Zig - a C-level language.](#written-in-zig---a-c-level-language)
- [PART 2: Comparing features](#part-2-comparing-features)
  - [Features in V1](#features-in-v1)
  - [Features in V2](#features-in-v2)
  - [Differencing the two versions](#differencing-the-two-versions)
- [PART 3: Architecting with a prompter](#part-3-architecting-with-a-prompter)
  - [What changed?](#what-changed)
- [PART 4: Insights and feedback](#part-4-insights-and-feedback)
  - [The issues:](#the-issues)
  - [Insights](#insights)
  - [What comes next?](#what-comes-next)
- [Thank you for reading](#thank-you-for-reading)

Word has it that builders in the 11th century were using color chalks to draw out the 3D structures of their projects on top of ink-written floor plans.

Architects from all over the world would gather at construction sites and look at these color scribbles to assess whether their sky-leaning endeavours would hold up. That's how they pulled hundreds of cathedrals out of the ground - most of which are still standing today.

So can we.

> _«I believed that the most important software […] needed to be built like cathedrals, carefully crafted by individual wizards or small bands of mages working in splendid isolation, with no beta to be released before its time.»_

Eric S. Raymond, The Cathedral and the Bazaar


# PART 1: What's this about?

## The Console 1 demo application - version 1.

TLDR: I wrote an extension for my favorite music application. It provides deep integration of Softube's Console1 controller.

The controller is basically a box covered with knobs and buttons which mimics a mixing studio's console.

I wrote the V1 of the project from January to March 2024 (a year ago), I presented the draft project circa 4 months ago, and wrote about it in [this post]().


## Written in Zig - a C-level language.

Written during spring 2024, without the help of an AI prompter.

No models were trained on Zig at the time, so using a prompter was not an option.

The app was good, but totally incomplete.

Time for V2 - a year later.


# PART 2: Comparing features

_Where I draw comparisons between V1 and V2, and the amount of work it took._

## Features in V1

- control surface
- 1 input mode
- user settings read/saved on disk
- user mappings read/saved on disk
- Ini custom parser

Cloc's output:
```
      29 text files.
      27 unique files.
       2 files ignored.

Language       code
C/C++ Header   6671
Zig            4771
C++             298
Lua              67
JSON             15
Markdown          3
SUM:          11882
```


## Features in V2

- control surface
- 5 input modes
- logger
- state machine with custom actions
- GUI loop using custom wrapper
- mapping panel
- fx ctrl panel
- settings panel
- debug panel
- user settings from disk
- user mappings from disk
- Ini custom parser
- Fx parser (read from disk and categorize)
- Fx browser
- custom GUI widgets
- global data storage module
- Theme parser
- Tests
- Theme tweaker
- 2 window styles

```
Cloc's output:
      48 text files.
      46 unique files.
       3 files ignored.

Language      code
Zig           8629
C/C++ Header  6671
C++            298
Markdown        95
Lua             67
JSON            15
SUM:         15832
```

Why are we talking about this?


## Differencing the two versions

- V1: 100 days of work, 4771 LOC, 5 features
- V2: 7 days of work, 8629 LOC, 17 features

By the day count: 14.3x faster

By the LOC count: 1.8x faster

By the feature count: 3.4x more features

Total combined speedup: 14.3 * 1.8 * 3.4 = you get the picture.

These are terrible metrics. But they tell us about the productivity gain: an order of magnitude.

What's the key factor in this difference?


# PART 3: Architecting with a prompter

Using the prompter for auto-completion is one thing. I had clear ideas about the project, but needed some recommendations on how to architect it.
I drafted a design document, fed it into the prompter's context, and started the discussion:

> _\<Design document inserted at beginning of discussion\>_
>
> I: Here's a draft design outline for my console1 extension for reaper. Provide feedback on the document and suggest data structures which might best match the needs of my project.
>
> Prompter: Based on your design document, I can offer this analysis and suggestions for data structures.
> _State machine definition,_
> _data structure definitions_
> Let me outline some key considerations:
> _point 1: bla bla bla,_
> _point 2: do this do that,_
> _point 3: yaddi yadda,_
>
> I: About point 1: How can I do X?
> Also, regarding point 2: Could you explain X aspect?
> For point 3: Here's what I'm doing, what's the canonical approach for this?
> By the way, you need to know X, Y and Z about how the module works.
>
> Prompter: _<corrects previous suggestions>_ Here's how you can build the state machine:
> _code example_
>
> _Tons more back and forth, discussing general approach and data structures._
>
> I: Let's talk about a game plan worth following to put together the project. I reckon the order of the steps ought to be:
> _proceeds to a bulleted list of steps_
>
> Prompter: Let me suggest some refinements and dependencies:
>
> _back and forth ensues, until the game plan lands into the design doc._
> _Implementation begins._
>
> _From there, it's a ride on the highway._

## What changed?

I involved the prompter from the initial design phase of the project.

I had a lot of unknowns and gaps in my knowledge which constrained my ability to make the right decisions - and designing a 15k-LOC application takes a _lot_ of decisions.

Point is: the prompter's been most likely trained on thousands of open-source code bases. If I ask it the right questions, I can yield some of the insights that drove their designs.

So what's changed is that I'm not using the prompter as an auto-complete: instead of asking it to finish my sentences - which is akin to asking a machine to read my mind - I used it as a talking encyclopedia.


# PART 4: Insights and feedback

## The issues:

- Size of context is the name of the game - for now.

Unfortunately, prompters cannot just be fed your whole code base and be asked to fix it. There's limitations on how much pre-requisite information you can provide them. This implies that, in order to yield pertinent answers, you must keep a tight leash on how much the machine needs to know.

If you don't provide enough context, the prompter can't answer pertinently. If you provide too much context, the prompter can't answer at all.

Maybe two years from now we'll have a 2M token-big context, and team-wide collaborative contexts. My gut feeling is that getting managers to adapt is going to be _much more_ of a challenge than getting a big context.

- Inline editing is hard, just like keeping separation of concerns is hard.

Prompters can make changes in-line, but it's tough to get them to write good local changes when all the app's pieces are touching each other.

- The deeper into the details, the more sloppy the prompter's output.

- Inaccurate questions yield uneven answers when I re-run the prompt.


- Accurate questions yield similar answers when I re-run the prompt.


- One-shot problems which are very custom or have not been solved before often do terrible.


- Exam-style questions yield the best output - the prompter is nothing but a talking encyclopedia, after all.


## Insights

- Non-assisted code is already the world of yesterday. Saying «AI is coming» is also the world of yesterday.

- Better-written prompts = better output

- Write to clarify your ideas, and get feedback on your writing. The quality of the feedback is a proxy for the quality of your writing.

- Best guidance and ideas come from tech-agnostic discussions. Talk first, without code.

- This thing has seen code bases and forum answers I haven't. It's got general culture I don't.

- Design docs come first. Then the list of steps, modules, features. Then a game plan, then the implementation.

- 90% of the process is rubber-ducking.

- Live by the tests, proof-read the code.

- Accurate questions make accurate answers.

- Game plan: design a path with as few dark corners as possible.

- 10x programmers don't write faster. They just only write once. The goal is to get it right on the first try.

- How much closer are you to getting it right on the first try now that you have a plan, as opposed to when you did not?

- Redoing the work is cheap with a prompter. You can iterate.

## What comes next?

- Now you can really go top-down.

- Prompters are almost in the past already. Agents are here.

- Some time from now: 1M token-context and a file watch mode to keep the context fresh.

- Open the door to collaborative contexts - e.g. in the form of an `Architecture.md`


# Thank you for reading

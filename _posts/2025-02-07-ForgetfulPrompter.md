# The forgetful prompter’s guide to building electric cathedrals

Word has it that builders in the 11th century were using color chalks to draw out the 3D structures of their projects on top of ink-written floor plans.

Architects from all over the world would gather at construction sites and look at these color scribbles to assess whether their sky-leaning endeavours would hold up. That’s how they pulled hundreds of cathedrals out of the ground - most of which are still standing today.

So can we.

> _«I believed that the most important software […] needed to be built like cathedrals, carefully crafted by individual wizards or small bands of mages working in splendid isolation, with no beta to be released before its time.»_

Eric S. Raymond, The Cathedral and the Bazaar


# PART 1: What’s this about?

## The Console 1 demo application - version 1.

TLDR: It’s an extension for my favorite music application. It provides deep integration of Softube’s Console1 controller.

The controller is basically a box covered with knobs and buttons which mimcs a mixing studio’s console.

I wrote the V1 of the project from January to March 2024 (a year ago), I presented the draft project circa 4 months ago, and wrote about it in [this post]().


## Written in Zig - a C-level language.

Written during spring 2024, without the help of an AI prompter.

No models were trained on Zig at the time, so using a prompter was not an option.

The app was good, but totally incomplete.

Time for V2 - a year later.


# Let’s take a look at the app


# PART 2: Comparing features

# Features in V1

- control surface
- 1 input mode
- user settings read/saved on disk
- user mappings read/saved on disk
- Ini custom parser

Cloc’s output:
```
      29 text files.
      27 unique files.
       2 files ignored.

github.com/AlDanial/cloc v 1.98  T=0.05 s (570.3 files/s, 626094.3 lines/s)
Language                         files          blank        comment           code
C/C++ Header                         7           2005           2462           6671
Zig                                 13           1076          12103           4771
C++                                  3             37              2            298
Lua                                  1              7             16             67
Windows Resource File                1             22             27             57
JSON                                 1              0              0             15
Markdown                             1              1              0              3
SUM:                                27           3148          14610          11882
```


# Features in V2

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
Cloc’s output:
      48 text files.
      46 unique files.
       3 files ignored.

github.com/AlDanial/cloc v 1.98  T=0.05 s (916.4 files/s, 610695.3 lines/s)
Language                         files          blank        comment           code
Zig                                 31           1567           8660           8629
C/C++ Header                         7           2005           2462           6671
C++                                  3             37              2            298
Markdown                             2             18              0             95
Lua                                  1              7             16             67
Windows Resource File                1             22             27             57
JSON                                 1              0              0             15
SUM:                                46           3656          11167          15832
```

Why are we talking about this?


# Differencing the two versions

- V1: 100 days of work, 4771 LOC, 5 features
- V2: 7 days of work, 8629 LOC, 17 features

By the day count: 14.3x faster

By the LOC count: 1.8x faster

By the feature count: 3.4x more features

Total combined speedup: 14.3 * 1.8 * 3.4 = you get the picture.

These are terrible metrics. But they tell us about the productivity gain: an order of magnitude.

What’s the key factor in this difference ?


# PART 3: Architecting with a prompter

Using the prompter for auto-completion is one thing. I had clear ideas about the project, but needed some recommendations on how to architect it.
I drafted a design document, fed it into the prompter’s context, and started the discussion:


> designDoc.md

## I:
Here’s a draft design outline for my console1 extension for reaper.
Provide feedback on the document and suggest data structures which might best match the needs of my project.

## Prompter:
Based on your design document, here's my analysis and suggestions for data structures:

### State Machine Structure
```zig
const Mode = enum { };
const State = struct { };
```

### Key Data Structures

```zig
const ConsoleControl = struct { };
const FxMapping = struct { };
const FxParameter = struct { };
const ChannelStrip = struct { };
const Settings = struct { };
```

### Suggestions:

1. **Persistent Storage**: _bla bla bla_
2. **State Management**: _do this, do that_
3. **Error Handling**: _yaddi yadda_
4. **Performance Considerations**: _nya nya nya_


## I:

> Storage: Here’s where the files are supposed to be stored, here’s the file format we use. How can I do X?
> State: Explain X aspect of your suggestion.
> Error: include Y style of error handling.
> Performance: Here’s what I’m doing, **what’s the canonical approach for this?**

> Also, you need to know X, Y and Z about how the GUI works. This newts your recommendation. Also, I want to know how to build the state machine.

## Prompter:

_<corrects previous suggestions>_

> Here’s how you can build the state machine

_code example_

[…]

_Tons more back and forth, discussing general approach and data structures._


## I:

> Let’s talk about a game plan worth following to put together the project. I reckon the order of the steps ought to be:

_proceeds to a bulleted list of steps_

## Prompter:

> let me suggest some refinements and dependencies:

_back and forth ensues, until the game plan lands into the design doc._

[…]

_Implementation begins._


# What changed?

I involved the prompter from the initial design phase of the project.

I had a lot of unknowns and gaps in my knowledge which constrained my ability to make the right decisions - and designing a 15k-LOC application takes a _lot_ of decisions.

Point is: the prompter’s been most likely trained on thousands of open-source code bases. If I ask it the right questions, I can yield some of the insights that drove their designs.

So what’s changed is that I’m not using the prompter as an auto-complete: instead of asking it to finish my sentences - which is akin to asking a machine to read my mind - I used it as a talking encyclopedia.


# PART 4: Insights and feedback

# The issues:

- Size of context is the name of the game - for now.

Unfortunately, prompters cannot just be fed your whole code base and be asked to fix it. There’s limitations on how much pre-requisite information you can provide them - that is called the «prompter’s context size». This implies that, in order to yield pertinent answers, you must keep a tight leash on how much the machine needs to know.
If you don’t provide enough context, the prompter can’t answer pertinently. If you provide too much context, the prompter can’t answer at all.

- Inline editing is hard, just like keeping separation of concerns is hard:
Prompters can make changes in-line, but it’s tough to get them to write good local changes when all the app’s pieces are touching each other.

- The deeper into the details, the more sloppy the prompter’s output.

- Inaccurate questions mean I get uneven answers when I re-run the prompt.


- Accurate questions mean I get similar answers when I re-run the prompt.


- One-shot changes that are very custom or have not been solved often do terrible.


- Exam-style questions yield the best output - the prompter is nothing but a talking encyclopedia, after all.


# Insights

- Non-assisted code is already the world of yesterday. Saying «AI is coming» is already behind.

- Better writing = better output

- Write to clarify your ideas, and get feedback on your writing. The quality of the feedback is a proxy for the quality of your writing.

- Best guidance and ideas come from tech-agnostic discussions. Talk first, without code.

- This thing has seen code bases and SO questions I haven’t. It’s got general culture I don’t.

- Design docs come first. Then the list of steps, modules, features. Then a game plan, then the implementation.

- 90% of the process is rubber-ducking.

- Live by the tests.

- Accurate questions make accurate answers.

- Game plan: design a path with as few dark corners as possible.

- 10x programmers don’t write faster. They only write once. The goal is to get it right on the first try.

- How much closer are you to getting it right on the first try now that you have a plan as opposed to when you did not?

- Redoing the work is cheap with a prompter. You can iterate.

## Some more nuanced aspects

In industry, there’s productivity differences of 1 to 10 from a developer to the next. That’s a sad aspect. Some coders are just better or faster than others, and management echelons have to take this as _fait accompli_ - no matter how high the standard in your hiring process.

When prompters came around, a lot of people drew the assumption that using AI was going to increase the productivity of programmers across the board. It was assumed - and still is - that prompters would just level the playing field.

I posit that this is not what is happening.

Among AI opponents, the anti-AI, there are those who say: the output of prompters is so bad that it's absolutely unusable. And anyway, junior developers or users without discernment who try to benefit from it make the work they're trying to do worse; because AI encourages them to never learn by themselves, and to never develop the qualities necessary for developers - i.e. discernment, the ability to handle logic, etc.

These opponents are right.

Only, they are right concerning junior devs. Regarding the quality of prompters' output - they are wrong. They were right a year ago. But today, the quality of the models, the quality of applications, has changed - drastically enough that the quality of questions over-determines the quality of output responses.

Today, what makes AI usage relevant is the relevance of the questions asked to it. But who are the users who ask the most relevant questions? They are those who were already the most capable before.

We started from the principle that AI would level the playing field, bridge the productivity gap between the least and most skilled - in reality, my bet is that it widens it.

It widens it because the devs who will be able to get the maximum advantage and the most significant productivity gain from using AI are those who were already the best before.

## Pascal's Wager
Then, there is a second category of anti-AI who are - let's say - the Pascalian anti-AI.

What does their way of thinking look like?

They somewhat model their view of AI and what it will become on Google. Google's idea was: "we want to build a database that is omniscient and omnipresent". That is, something that has an answer to everything, that is present in all the world's systems and that has a constant influence and presence in everyone's life.

But if you are omniscient, omnipresent, and omnipotent (and this is what we saw during covid) your knowledge cannot be contradicted by reality - because otherwise it calls into question your omniscience. If you are truly omniscient - which until now tech giants claimed to be - you cannot be wrong. This implies that you have power of preemption over reality. You can preempt reality, even if it proves you wrong. If events prove you wrong, but you are omniscient, then the events are wrong - not you.

This is the artifact of a very transhumanist vision - using technical means to build artificial omniscience, artificial omnipotence, and artificial omnipresence, the tech world considered it was able to manufacture a kind of false god.

That is, a creature that is falsely omniscient, falsely omnipotent and falsely omnipresent. I think many people were afraid of this, that they sensed the existence of this vision among the big leaders, the most successful entrepreneurs in the tech world, and they saw in AI the potential for tenfold continuation of this vision - and they hated the idea.

They hated the idea with good reason because a false god, falsely omniscient, falsely omnipotent, who has power of preemption over reality, and who has the ability to lie and manipulate us to try to control us according to its will, has a name: it's an antichrist. That's what we call an antichrist.

So regarding the Pascalian anti-AI, I think this is a category of person who sensed the transhumanist, antichrist-like vision of AI, and who somewhat deifies tech. They consider that what we are about to create - what we are pursuing through the development of general artificial intelligence - is fundamentally the creation of a false god. It's fundamentally the creation of an antichrist, whose only possible goals - once it gains consciousness - is to destroy humanity.

It's a kind of Pascal's Wager, but inverted: if you believe in the false god, you have everything to lose. However, if you refuse to believe in the false god and turn away from it, you have everything to gain.

Among AI's Pascalians, there is a second perspective that speaks not of the wager, but of Pascal's mugging. The mugging story goes: Pascal is approached by a thief who forgot his knife. The thief tells him: "give me a thousand bucks, I'll go home and bring you back ten thousand." And Pascal replies "No, it's not worth it, since the risk level is too high." So the thief tells him: "Fine, just give me a hundred bucks. I'll prove you wrong and give you back not ten thousand but a million."
The lesson of this story is to balance risk levels. It's about balancing the level of gain and potential benefit drawn from a bet of this size.

In both cases - pure Pascalians and Pascal's muggers have a very Manichean vision, very deifying of the tech phenomenon, and which systematically substitutes for human judgment.
They have somewhat of a fascist dimension - they credit the transhumanist idea that common man is very small before great minds and that his submission to technological domination is inevitable.
They have somewhat - despite themselves - this desire to resist this vision, while crediting it as being valid, possible, and inevitable.

It's a somewhat eschatological view, somewhat bourgeois, somewhat Manichean, somewhat science fiction, and it's difficult for me to say if it can materialize - I have my doubts on the question. Here's why.

What we currently see in AI usage is that its quality, its result, its scope, its aim, are exclusively determined by the use we make of it - i.e. by the discernment, intelligence and intentions of the human factor.
# What comes next?

- Now you can really go top-down.

- Prompters are almost in the past already. Agents are here.

- Some time from now: 1M token-context and a file watch mode to keep the context fresh.

- Open the door to collaborative contexts - e.g. in the form of an `Architecture.md`


# Thank you for listening

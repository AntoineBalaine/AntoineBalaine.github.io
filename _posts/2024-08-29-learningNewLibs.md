---
layout: post
title: "The opiniologue's guide to keeping sane when learning a new tech"
date: 2024-08-29
categories: [Development, Learning]
tags: [learning-process, programming, methodology, best-practices]
description: "A practical guide to learning new technologies efficiently, using project complexity matching and commit counting as progress metrics."
toc: true
---

1. [Disclaimer](#disclaimer)
2. [The problem](#the-problem)
3. [The lean mean learning machine only picks battles it can win](#the-lean-mean-learning-machine-only-picks-battles-it-can-win)
4. [The copyist monk approach](#the-copyist-monk-approach)
5. [Some merry-go-round projects](#some-merry-go-round-projects)
6. [My merry go-round project: writing a small parser](#my-merry-go-round-project-writing-a-small-parser-for-a-markdown-format)
7. [Try to match the size of the project to your level of proficiency](#try-to-match-the-size-of-the-project-to-your-level-of-proficiency)
8. [The commit count](#the-commit-count)
9. [Commit counts are vanity](#commit-counts-are-vanity-theyre-only-an-opiniologues-take)
10. [Mind-experiment: reaching fluency with two unknowns](#mind-experiment-lets-think-about-what-its-like-to-reach-fluency-with-two-unknowns)
11. [This line of thinking is not a system](#this-line-of-thinking-is-not-a-system)

Notes from an internal presentation I did at my company.

The initial announcement carried the following bombastic teaser:

> In this presentation, I’ll be presenting the world with a new branch of engineering that promises to encompass all the fields of human knowledge, from human sciences to natural sciences, to math and beyond: opiniology.
>
> In this age of fast-paced change, you imperatively need a go-to roadmap to discern and leverage upcoming transformative technologies.
>
> The opiniologue’s guide will provide you with a clear 4-step process to learning a new tech without feeling like a head-less chicken seeking its way out of a mini golf.

# Disclaimer

Learning is personal.

I am but an obnoxious french opiniologue. Any of the following is thus only personal perspective. Don’t take it as gospel.

> _«Only fools don’t change opinions. It’s my opinion, and I don’t see why I should change it.»_


# The problem
To learn a new language, you need to write a project.
To write a new project, you need to know the language.

The project and the language are 2 unknowns, and you can only afford to have one of them.


# The lean mean learning machine only picks battles it can win

Military industries only produce tank designs which’s specs guarantee 95% accuracy within their shooting range.

When the battle starts, if targets are outside of the tanks’ range, the tanks stay in the garage.

### So should you.
_Note: I promise this is the last «should» of this presentation._

Anything below 95% certainty that you’re going to hit the bull’s eye is a waste of your time.

Whichever new tech you’re trying to approach, the curriculum that you’re planning to follow needs to be perfectly clear.

Having one unknown (i.e. the language or the framework) is inevitable. Having two is much too expensive.
You can reduce the unknowns:
- write a program you already know in a new language, or
- write a new program in a language you already know.


# The copyist monk approach:

Ready a small repertoire of programs, and port them to your new tech.

### If you have zero repertoire, learn by copying other people’s work.

All you need is 1 project in your repertoire.

Learn programs which address problems that appear everywhere.
The skills that you acquire by implementing those programs give you a bag of tricks:

- accessing files from disk,
- mapping data,
- handling strings,
- writing tests,
- structuring your project,
- installing dependencies,
- building,
- etc.


# Some merry-go-round projects:
1. a parser
2. a linked list or hash table
3. an http library
4. an underscore.js (functional programming)
5. a FE/BE app - such as the «realworld» app reference
6. a ray-tracer
7. a database
8. a static site generator


# My merry go-round project: writing a small parser for a markdown format

- Started off with the dream of writing a language server for the ABC music format
```yaml
X: 1
T: My angel-sounding song
   [ceg]  [cfa]  [ceg]
w: aaah - aaah - aah
```

1. write a parser
  - read the book _crafting interpreters_ for the basics
  - use a reference implementation of the book’s toy language
2. write a language server
  - use microsoft’s docs [FAILED miserably]
  - use a reference implementation for a complex language like bash [FAILED also miserably]
  - use a reference implementation for a toy language [success!]

Crafting Interpreter’s Lox scanner, in typescript, pulled from someone’s github repo:
```typescript
export default class Scanner { // SawyerHood/lox-typescript
  private source: string = ''
  private tokens: Token[] = []
  private start: number = 0
  private current: number = 0
  private line: number = 1

  constructor(source: string) {
    this.source = source
  }

  scanTokens(): Token[] {
    while (!this.isAtEnd()) {
      this.start = this.current
      this.scanToken()
    }
    this.tokens.push({type: TokenType.EOF, lexeme: '', literal: null, line: this.line})
    return this.tokens
  }

  private scanToken() {
    const c = this.advance()
    switch (c) {
      case '(':
        this.addToken(TokenType.LEFT_PAREN)
        break
      case ')':
        this.addToken(TokenType.RIGHT_PAREN)
        break
    }
```


My version - the scanner for the ABC music markdown
```typescript
export class Scanner {
  private source: string;
  private tokens: Array<Token> = new Array();
  private start = 0;
  private current = 0;
  private line = 0;
  constructor(source: string, errorReporter?: AbcErrorReporter) {
    this.source = String.raw`${source}`;
  }

  scanTokens = (): Array<Token> => {
    while (!this.isAtEnd()) {
      this.start = this.current;
      this.scanToken();
    }
    this.tokens.push(
      new Token(TokenType.EOF, "\n", null, this.line, this.start)
    );
    return this.tokens;
  };
  private scanToken() {
    let c = this.advance();
    switch (c) {
      case "\\":
        if (this.peek() === "\n" && !this.isAtEnd()) {
          this.advance(); this.addToken(TokenType.ANTISLASH_EOL); this.line++;
        } else {
          this.advance(); this.addToken(TokenType.ESCAPED_CHAR);
        }
        break;
      case "&":
        this.addToken(TokenType.AMPERSAND); break;
```


The language server project:
```tree
jslox/src (fjakobs/jslox)             AbcLsp/src (my version)
├── extension.ts                      ├── extension.ts
├── jslox                             ├── extensionCommands.ts
├── lsp-client.ts                     ├── server
├── server                            │   ├── AbcDocument.ts
│   ├── LoxDocument.ts                │   ├── AbcLspServer.ts
│   ├── LoxLspServer.ts               │   ├── completions.ts
│   ├── SemanticTokenAnalyzer.ts      │   ├── server.ts
│   └── server.ts                     │   └── server_helpers.ts
└── test                              └── test
    ├── runTest.ts                        ├── runTest.ts
    └── suite                             └── suite
        ├── extension.test.ts                 ├── extension.test.ts
        └── index.ts                          └── index.ts
```


# Try to match the size of the project to your level of proficiency with the new tech

Assess: where am I in my progression?
1. pre-beginner
2. beginner
3. intermediate
4. fluent

Assessing this is fairly nebulous:
Some past projects of yours might have done a good job with the tech, but totally overlooked one of its big aspects.

You can’t assess your overall in the blind, but you can assess the peak you have reached.

You can objectivate your peak using a stupid metric:


# the commit count

How many commits does it take to implement one of the programs from my repertoire?
How many unknowns are there going to be along the way?

With 0 unknowns, there’s 0 challenge. With 2 unknowns, it’s too expensive to complete.

### 1. pre-beginner: exercises repo, walk-through tutorials, <= 10 commits to complete
  - 1 unknown, but you can look up the answers at every step
  - overview tutorial, base docs, learnXInYMinutes
  - syntax basics repo: rustlings (rust) / ziglings (zig) / clonings (clojure) / golings (go)
### 2. beginner: base functionality repo, <= 20 commits to complete
  - copy an existing project or port an existing project to one of your languages
  - todo app
  - ini|json parser
### 3. intermediate: <= 80 commits to complete
  - realworld app (port it to a known language, or copy a reference in your new language.)
### 4. fluent: > 100 commits

At each step of the process, you decrease how much you rely on existing examples, or on the support of an online community.

These commit counts are rough patterns that I noticed in my process. Can they be trusted?


# Commit counts are vanity. They’re only an opiniologue’s take.

### Commit count is a Big Mac index. Nothing more.

I posit that once you have completed an intermediate project, you have made it to fluent-speaker land.

By my big-mac index, that’s 10 (pre-beginner) + 20 (beginner) + 80 (intermediate) = 110 commits.

### Counting commits is a vanity metric of how much you produce, not how long it takes.
If you’re averaging 5 commits day, you can do a rough estimate of time, but that’s not going to be accurate.

### My own index does not matter: what matters is you, and experimenting with what works for you.

You could use any other metric - provided that the number of unknowns at play does not falsify your findings.


# Mind-experiment: let’s think about what it’s like to reach fluency with two unknowns

I assume that reaching fluent-speaker land takes 110 commits for a new tech (framework, lib, etc.), and 110 commits for a language.

### If you start with two unknowns, I posit that they compound at every step of the process:

Learning a framework in an unknown language, every line of code takes figuring out an unknown in the language.
I posit that this results in 10 commits worth of framework * 10 commits worth of language.
As a result, writing 10 commits of project, requires 10 * 10 commits of work.

If I separate the process, I do 10 commits worth of language _and then_ 10 commits worth of framework.

### This is not accurate arithmetic by any means - it’s opiniology.

But it does give me a ballpark: figuring out two unknowns simultaneously is exponentially more difficult than one at a time.

About whether it’s worth applying my learning stages to yourself:


# This line of thinking is not a system

Going about thinking «I’m at beginner 2.2-a» and «I completed x commits, thus I’m at peak level x» makes you focus on the wrong thing.

This is _not_ a framework, just a helper to match your next project’s size with your current level of proficiency.

### A wrong match threatens your entire learning journey
If you skip a step, i.e. try an advanced project before acquiring the basics, the overwhelming cost of the effort is likely to make you quit.

If you abandon your learning track before reaching fluency, the new tech won’t land in your toolbox.
At this point, you’ll be left with an incomplete experiment, wondering what it would have taken to succeed.

Nobody likes taking anything home other than victory. Pick battles you’re sure to win.

### «No match» also threatens your journey
Running out of ideas on what to study next is also a dangerous place to be. You could again find yourself dropping out just because you don’t know where to go.

That’s especially the case when big codebases in your new tech still look alien.
In this situation: refer back to your repertoire of projects. They’re here to help.

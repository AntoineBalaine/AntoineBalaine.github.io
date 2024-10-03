---
layout: post
title:  "[code] The neurotic astronaut’s guide to assertions"
date:   2024-09-02 16:18:36 -0400
categories: assertions code
---

Notes from another presentation I did at my company.

The following is a dumbed-down version of [Tigerbeetle’s «it takes two to contract»](https://tigerbeetle.com/blog/2023-12-27-it-takes-two-to-contract). 
Same structure - just fewer words, for my short-attention-span friends.


# The neurotic astronaut’s guide to assertions and contract programming

How to avenge your frustration by punishing this unwieldy software.

---

# The problem:

Every 10 to 100 lines of code, there’s a bug. 

Writing computer language is not very prone to stream of thought.

AI promises to solve all our problems, in exchange for living in code-dystopia. I’ll pass.

# The goal: Write code with no bugs!
I just want to write things once, give the app a track run, and go back to gardening.

---

# Some strategies:

1. re-usable components - the less code, the less bugs I have
2. test, test and test - grind those bugs into oblivion
3. isolate the components and test - combine the two previous ones! À la Storybook
4. data types - they increase the grind, but they make you dodge 60% of the bullets.
5. in-line documentation - great for reading, but not great for maintenance

Obviously, these three don’t cover the whole territory:
- How can we work on the contents of the data?
- How can we securize the state of the app?
- How can we make sure that the transforms are correct?

---

# Enter assertions

Let’s fail fast: if my expectations about the state of some data are not met, let’s blow up the app.
```typescript
function assertIsMoreThan (value: unknown, reference: number): value is number {
  if (typeof value !== "number") throw new Error("Not a number");
  if (value <= reference) throw new Error("invalid value");
}
```
The types tell us something, but the real picture comes from `value <= reference`

If you were planning on adding a doc comment instead of blowing up the app, you were going to miss on a chance to be proven wrong.

> _«But, but, but! Breaking the game every time I’m proven wrong sounds like childish neurosis!_

> _— Yes. Embrace the chaos.»_

---

# How do I fit this in the code? How is this a strategy?

```zig
var someGlobal: ?string = null;

fn Main(){
  openComponent();
  whileOpen();
  closeComponent();
}

fn openComponent() {
  assert(someGlobal == null); 
  someGlobal = allocate("hello world");
  assert(someGlobal != null);
}

fn whileOpen() {
  assert(someGlobal != null);
  drawText(someGlobal);
}

fn closeComponent(){
  assert(someGlobal != null);
  deallocate(someGlobal);
  assert(someGlobal == null);
}
```

---

Reading this, was it easy to figure out what `allocate()` and `deallocate()` do?

If it doesn’t do what I expect, how long is it going to take for me to find out?

Does it look like a doc-comment would have been a better option?

Does it look like a unit-test would have been a better option?

---

# Some syntactic sugar in Go and Zig for this

Using `defer`, which runs at the end of a scope:
```zig
fn closeComponent(){
  assert(someGlobal != null);
  defer assert(someGlobal == null);
  deallocate(someGlobal);
}
```
This is not just about littering code as a means to feel better.

1. It provides information to the reader
2. It clearly states what the invariants are in this situation
3. It sets clear expectations about the result of a data transform
4. It alleviates the need to formally prove that the code behaves expectedly
5. It alleviates (some of the) need to unit test

---

# That’s not what a contract does

How about having the assertions on both sides of the function call?

```zig
fn Main(){
  assert(someGlobal != null);
  closeComponent();
  assert(someGlobal == null);
}
```
Now, that’s a contract: 2 parties are asking a question, and they expect to agree.
If I refactor this code - say I move some sections to a distant place in the codebase - the assertions are still there.
Everywhere.

> _«But, but, but ! That’s code duplication !_

> _— Is it really ?_

> _— Who uses this level of contorsion anyway ?»_

---

# NASA does this.

NASA’s Power of 10: Rules for Developing Safety-Critical Code

1. Restrict all code to very simple control flow constructs 
  — do not use `goto` statements, `setjmp` or `longjmp` constructs, or direct or indirect recursion.
2. Give all loops a fixed upper bound.
3. Do not use dynamic memory allocation after initialization.
4. No function should be longer than what can be printed on a single sheet of paper in a standard format with one line per statement and one line per declaration.
5. The code's assertion density should average to minimally two assertions per function.
6. Declare all data objects at the smallest possible level of scope.
7. Each calling function must check the return value of non-void functions, and each called function must check the validity of all parameters provided by the caller.
8. The use of the preprocessor must be limited to the inclusion of header files and simple macro definitions.
9. Limit pointer use to a single dereference, and do not use function pointers.
10. Compile with all possible warnings active; all warnings should then be addressed before the release of the software.

---

# We all want to build a spaceship

Let’s just neurotically break it from time to time.

Thank you, code astronauts.


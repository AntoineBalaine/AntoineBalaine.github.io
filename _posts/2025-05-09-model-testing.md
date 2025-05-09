---
layout: post
title: "Devlog: ZlowCheck’s model testing"
date: 2025-05-09
categories: [devlog, code]
tags: [property-based testing, zig, PRNG, devlog]
description: "Property-based testing 2: model tests"
toc: true
---


# A word about model testing:
You want to test your awesome app, but you want to test it _while it’s moving_, instead of when it’s at a stop. You need to verify that, no matter what sequence of interactions your users might have with your program, it keeps on behaving expectedly.

So you decide to implement a simplified model version of it. The model version doesn’t implement the i/o routines, the complicated side effects, the complex initialization, none of the hard stuff. It only keeps track of the stuff that’s worth checking at runtime. 

Because you’re crazy contorted, you decide that you want to check *all* the possible interactions that your user can have. So you write an exhaustive list of all the user-actions. Then, for each of these actions, you write a function that executes the action against the model and your app, and checks that both of them agree on the outcome. 

# How ZlowCheck does slow-food

Ok, I implemented model-based testing in my [ZlowCheck library](https://github.com/AntoineBalaine/zlowcheck).

Here’s a long sample:

```zig
test assertStateful {
// Define commands
const IncrementCommand = struct {
    pub const name = "increment";
    pub fn checkPrecondition(self: *@This(), model: *ModelType) bool {}
    pub fn onModelOnly(self: *@This(), model: *ModelType) void {}
    pub fn onPair(self: *@This(), model: *ModelType, sut: *SystemType) !bool {}
};

const DecrementCommand = struct {
    pub const name = "decrement";
    pub fn checkPrecondition(self: *@This(), model: *ModelType) bool {}
    pub fn onModelOnly(self: *@This(), model: *ModelType) void {}
    pub fn onPair(self: *@This(), model: *ModelType, sut: *SystemType) !bool {}
};

var inc_cmd = IncrementCommand{};
var dec_cmd = DecrementCommand{};
// Create commands array
const commands = [_]Command(ModelType, SystemType){
    Command(ModelType, SystemType).init(&inc_cmd),
    Command(ModelType, SystemType).init(&dec_cmd),
};

var random_bytes = [_]u8{
    0x8B, 0xBE, 0x02, 0x2C, // etc.
};
var prng = FinitePrng.init(&random_bytes);
var random = prng.random();

// Create command sequence
const CommandListType = CommandList(ModelType, SystemType);
var cmd_seq = try CommandListType.init(
    &commands,
    &random,
    .{ .max_runs = 50 },
    std.testing.allocator,
);
defer cmd_seq.deinit(std.testing.allocator);

// Initialize model and SystemType
var model = ModelType{};
var sut = SystemType{};

// Run the test
const failure_opt = try assertStateful(
    ModelType,
    SystemType,
    &cmd_seq,
    &model,
    &sut,
    .{ .verbose = false },
);
try std.testing.expect(failure_opt == null);
}
```

There’s a few opinionated choices which might turn out to be cursed. Not sure, yet. Let me explain:

`assertStateful` is a model test runner that expects a commands list which implements an iterator interface.

At every call to `commandListIterator.next()`, the iterator decides which command from the CommandList is going to be run next. Each of these decisions is recorded by the iterator under the hood. The way the decision is encoded is: 
```zig
const Entry = struct {
/// offset into prng’s byte stream, which served as seed for the decision
  byte_pos: u32, 
/// which command was selected, represented as an non-exhaustive enum
  cmd_idx: Idx,  
}
```

Once the runner encounters a failure, it proceeds to binary-search for a shorter failing sequence. Finding a shorter version requires re-playing through segments of the sequence, to see if they reproduce the failure. This can be done using each decision’s `byte_pos`. That’s the main advantage of using the finite-prng: if I move the cursor into the underlying byte stream, I can replay through all the decisions without needing to store the data they output. The `byte_pos` of each entry can be used to re-generate any of the decisions in the sequence - in fact, I could very well _not_ record the `cmd_idx`, and just re-generate it at every replay.

```
Decision1 {byte_pos: 0} … Decision2 {byte_pos: 4} … Decision3 {byte_pos: 9} … FAILURE
                            ^
                            starting the replay from here yields an identical reproduction
```

The advantage of this is: let’s say some of the commands wrap a `Generator(T)` which also relies on the finite-prng. This means that whatever they generated will _also_ be replayed:

```
Decision1{} … Generator(T).generate() … Decision2{} … Generator(U).generate()
                                            ^
                                            starting the replay from here will also 
                                            yield the same value from Generator(U).generate()
```
This is done using an alternate method from the iterator: `iterator.nextReplay()`. The only difference with the canonical `iterator.next()` is that `nextReplay` doesn’t record the decisions.

# So what’s the matter?

The reason why this might be cursed: in ZlowCheck, some Generators for specific types have to allocate data - i.e. Generators for slices and arrays. I still haven’t figured out how the lifetime of that data is supposed to play out. For now, I reckon it’s going to be up to the library consumer to worry about that.


This makes the code super easy to reason about, though:
```zig
// assertStateful()
var iterator = command_sequence.iterator(null);

// keep on picking commands
while (try iterator.next()) |*cmd| {
  // apply the cmd to the model/System pair
  if (try cmd.onPair(model, sut)) continue;

  // Create a failure result
  var result: StatefulFailure = .init(iterator.index);

  // Try to shrink the command sequence
  const shrunk_position = try shrink(
    command_sequence,
    model,
    sut,
  );
}
```

And then, I can pass the start offset of the replay to a new version of the iterator:

```zig
// replayChunk()
model.reset();
sut.reset();

// pass the chunk
// which contains the start and end positions we want to replay
var iterator = command_sequence.iterator(chunk);

// Run through the commands in this chunk
while (try iterator.nextReplay()) |cmd| {
    // Yep, it’s the same…
    if (!try cmd.onPair(model, sut)) {
        // Found a failure - this chunk reproduces the issue
        return true;
    }
}

// No failure found with this chunk
return false;

```

---
layout: post
title:  "[code] Zig’s unions, interfaces, and multi-arrays: the shoot-out"
date:   2024-10-03 09:09:36 -0400
categories: Zig code
---
# Table of Contents
- [Introduction](#introduction)
- [Unions](#unions)
- [Interfaces](#interfaces)
- [Multi-Arrays](#multi-arrays)
- [How to pick](#which-of-the-three-should-i-pick)
- [References](#references)

# Introduction
In the process of trying to switch my project to data-oriented design, there’s been the delicate phase of choosing the data structures.
My initial reaction to the unions had been very excited, with their `inline` capabilities.
The difficulty I had stemmed from that my project required two sets of unions to be nested into each other:
```zig
const Param = union {
  BasicParam: struct {id: u8, name: [:0]const u8},
  ControlParam: union {
    Button, Knob, Slider, Drag
  }
}
```
As my project grew, it progressively became more difficult to wrangle this level of nesting while adding more features.
I was thirsty for something faster and easier to use.
Enter the discussion about interfaces. If you haven’t read the post by [open my mind](https://www.openmymind.net/Zig-Interfaces/), go do that. It’s great.

# Unions
This is a statically typed environment, there’s one piece we’d like to use everywhere, but it must be represented by multiple types. In the case of a union, that’s easy.
```zig
union {
  button, knob, slider, drag
}
```

*Pros*: easy to use, easy interface, 0 maintenance.

*Cons*: they take up the memory space of the largest of their variants. Is their performance worse? Not sure - yet…

# Interfaces
As an alternative, the interfaces are great. You stick the data behind an erased pointer, and you then don’t have to be wrestling the type system to accommodate all the variants of the data. It’s pretty neat:
```zig
const Control = struct {
    ptr: *anyopaque,
    draw: *const fn (self: *anyopaque) anyerror!void,
    deinit: *const fn (self: *anyopaque, allocator: Allocator) void,
    pub fn init(ptr: anytype) Control {} // init() implementation here
```
The opaque pointer is called a layer of indirection: just misleading the type system, hiding what it doesn’t need to know so that we can fit multiple shapes of data in a single spot.
Upon instantiation, you can shove your actual type, the init() takes care of creating the pointer. Then whenever you try to call methods of the interface, such as draw(), the interface removes the indirection, casts its content back to its original type, and calls the corresponding method.

*Pros*: easy to use, fits everywhere.

*Cons*: Storing the data behind the indirection must be on the heap. Lord knows where it’s going to be placed in memory.

# Multi-Arrays

The interfaces are neat, but if you have a union and you have a limited number of variants, you have another option from the unions: splitting your unions into separate tables.
Here’s the quote from the ziggit discussion:

> […] if your use-case allows using union-switch approach, and if there are many objects you want to call [a method] with, you may also consider data-oriented approach where you keep separate arrays for each type.

```zig
const pets: []Pet = ... // a lot of pets
for (pets) |pet| {
    pet.speak();
}
```

> you may consider this instead

```zig
const dogs: []Dog = ... // all dogs are here
const cats: []Cat = ... // all cats are here
for (dogs) |dog| {
    dog.bark();
}
for (cats) |cat| {
    cat.meow();
}
```

> […] if you are able to list all your types in the union (i.e., all types are known at compile time), then you are able to list all arrays (cats and dogs) as well. You may define a struct that holds the lists.

# Which of the three should I pick?
Well, I like all three of these approaches.
The third one can work because I have a known number of variants to work with.
Let’s do a shoot-out to compare.

My benchmark implements 400 instances of the data structure, iterates 10M times over it, and calls a method with its data.
First, the nested union:
Here, `Control`’s a union that contains a single knob variant. I just want to find out what overhead I’m incurring with accesses:
```zig
const PRM = union(enum) { Base: Base, Control: Control };
for (0..400) |i| {
    const knob = try allocator.create(Knob);
    knob.* = Knob{ .id = @as(c_int, @intCast(i)), .name = try allocator.dupeZ(u8, "knob") };
    try db.prmList.append(allocator, PRM{ .Control = Control.init(knob) });
}

for (0..10_000_000) |_| {
    for (db.prmList.items) |*item| {
        switch (item.*) {
            PRM.Base => {},
            PRM.Control => |*control| {
                switch (control.*) {
                    .Knob => |*knob| {
                        try knob.draw();
                    },
                }
            },
        }
    }
}
```
Using an interface, same spiel:
```zig
const Control = struct { // my indirection layer
    ptr: *anyopaque,
    draw: *const fn (self: *anyopaque) anyerror!void,
    deinit: *const fn (self: *anyopaque, allocator: Allocator) void,
    pub fn init(ptr: anytype) Control {}};
for (0..400) |i| {
    const knob = try allocator.create(Knob);
    knob.* = Knob{ .id = @as(c_int, @intCast(i)), .name = try allocator.dupeZ(u8, "knob") };
    try db.controls.append(allocator, Control.init(knob));
}

const start = std.time.milliTimestamp();
for (0..10_000_000) |_| {
    for (db.controls.items) |control| {
        try control.draw(control.ptr);
    }
}
```
And the table-based version:
```zig
for (0..400) |i| {
    const knob = KnobECS{ .id = @as(c_int, @intCast(i)), .name = try allocator.dupeZ(u8, "knob") };
    try db.Knob.append(allocator, knob);
    const details: Details = .{ .largestep = 0.1, .smallstep = 0.01 };
    try db.details.append(allocator, details);
}

const start = std.time.milliTimestamp();
for (0..10_000_000) |_| {
    for (db.Knob.items, 0..) |*knob, idx| {
        try knob.draw(&db.details.items[idx]);
    }
}
```

And the results are in:
```
UNION elapsed time: 11594, average: 0.000028985
db: 24 bytes, union: 48000 bytes, total: 48024 bytes

INTERFACE Elapsed time: 23891, average: 0.0000597275
db: 24 bytes, knobs: 44800 bytes, interface: 9600 bytes, total: 54424 bytes

DOD elapsed time: 11423, average: 0.0000285575
db: 112 bytes, knobs: 9600 bytes, details: 35200 bytes, total: 44912 bytes
```

I’m surprised. I’d initially expected the unions to perform much worse.
Obviously, the benchmark is very superficial, and there’s probably some things which I don’t know about. It does give an insight though: the interfaces seem to be slower and more memory-hungry…

# References:
[Ziggit - «Switch based polymorphism vs Pointer based. Which is more efficient?»](https://ziggit.dev/t/switch-based-polymorphism-vs-pointer-based-which-is-more-efficient/5695)

[Open my mind’s «Zig Interfaces»](https://www.openmymind.net/Zig-Interfaces/)

[Hexops - «Let’s build an entity component system»](https://devlog.hexops.com/2022/lets-build-ecs-part-2-databases/)

[Casey Muratori - «Clean Code, Horrible Performance»](https://www.computerenhance.com/p/clean-code-horrible-performance)

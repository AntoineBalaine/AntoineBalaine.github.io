---
layout: post
title: "TreeList 4: benchmarking"
date: 2025-06-02
categories: [devlog, code]
tags: [array-pool, zig, devlog]
description: "Comparing two implementations of the treelist"
toc: true
---

Here’s a project in which I’m implementing an in-memory [database of trees](https://github.com/AntoineBalaine/treelist).

The goal of the database is to house some pre-made GUI layouts for audio FX. Each layout is a tree of sub-components. Components typically carry data like x/y coordinates, font sizes, colors, etc. 
Each FX on the user’s computer can have a layout, which allows for quick recall to display in an fx-rack host (think of an fx rack like Ableton’s).

# Benchmarking the TreeList
- [How does it fare?](#how-does-it-fare)
- [Maybe this could have been simpler?](#maybe-this-could-have-been-simpler)
- [Benchmarking](#benchmarking)
  - [Setup and Configuration](#benchmarking)
  - [Results: Insertions](#benchmarking)
  - [Results: Traversals](#benchmarking)
  - [Conclusion](#benchmarking)

## How does it fare?

Tree nodes are of heterogeneous types. I want to have as good a cache locality as possible, so I want each node type to be stored in a dedicated table. This allows accessing nodes via indexes, instead of using pointers.

That’s my initial implementation: the TreeList. The Treelist accepts a struct of types at compile time, and creates a list of tables for each type in the struct. From there, node addressing can be done using direct indexing into the list of tables. 

The concept is very simple, though the implementation definitely gave me a few grey hairs.

## Maybe this could have been simpler?

There’s another possible approach, instead of using a list of tables: you can use a multi-array list, which stores tagged unions. This makes for a much simpler implementation, though initially I’d dissed it because there’s difference ratios in memory sizes of my node types of 1 to 10. I felt stupid knowing that the smaller variants would end up using 10 times as much memory just because of how unions work.

Big advantage: 
- the logic is much simpler, 
- there’s a single array under the hood, which should benefit cache locality

Big disadvantage: 
- the multi-array list’s API can’t retrieve pointers, so you need to implement a dedicated `update` API.
- your variants now all occupy the same space in memory. Not a big deal, unless your variants are wildly different in size.


## Benchmarking

Since this is 2025 and we’re all affected by the «fear of missing out» syndrome, I decided to investigate what such a scenario would _actually_ cost in terms of perf. So I set out to implement a second version of my TreeList, backed by a multi-array list instead of a pool of tables.

When it comes to the tree algos (traversals, appends, etc.), they’re the same.

In order to compare the versions, I created a program which can take cli arguments to trigger the use of either of the data structs. The programs will then be loaded in [p.o.o.p.](https://github.com/andrewrk/poop) or [hyperfine](https://github.com/sharkdp/hyperfine) for comparison.   

The basic execution run looks as follows:
```zig
const BenchmarkConfig = struct {
  num_trees: usize = 1000,
  nodes_per_tree: usize = 100_000,
  // […]
};
```

Obviously, I’ve picked the worst possible usage scenario for the multiarray list, here:

```zig
const MA_MediumNode = struct {
    value: i32,
    extra_data: [10]i32 = [_]i32{0} ** 10,
    child: ?u32 = null,
    sibling: ?u32 = null,
    parent: ?u32 = null,
};

const MA_LargeNode = struct {
    value: i32,
    // 10x larger than medium
    extra_data: [100]i32 = [_]i32{0} ** 100, 
    child: ?u32 = null,
    sibling: ?u32 = null,
    parent: ?u32 = null,
};
```
This is the worst possible scenario because here the smaller nodes are taking up as much memory as the larger ones - even though the larger ones are 10 times as large. This means that every time you want to traverse a node, you must load up the largest possible memory occupation of all the variants into the CPU. And more memory generally means slower loads. 

To be noted that the MultiArrayList gets a tiny leg up on the array-pool here, since the latter’s `Location`s must take up a `u64` instead of a `u32` to perform indexing. That’s because, in the array-pool’s `Location`, the first 32 bits are used to identify into which of the tables we need to index - whereas the multi-array list only uses one table under the hood.
```zig
const TL_SmallNode = struct {
    value: i32,
    child: ?u64 = null,
    sibling: ?u64 = null,
    parent: ?u64 = null,
};
```

Obviously, you could work your structs so that they align with the size of cache lines. This is not the case here, which makes things worse. When it comes to backtracking through the tree with my state-machine approach, the multi-array list’s tagged unions quickly become expensive. 

The run gives me: 
```sh
hyperfine "./zig-out/bin/benchmark --impl=treelist" "./zig-out/bin/benchmark --impl=multiarray" 
Benchmark 1: ./zig-out/bin/benchmark --impl=treelist
  Time (mean ± σ):      1.879 s ±  0.006 s    [User: 1.757 s, System: 0.122 s]
  Range (min … max):    1.872 s …  1.890 s    10 runs
 
Benchmark 2: ./zig-out/bin/benchmark --impl=multiarray
  Time (mean ± σ):      6.754 s ±  0.078 s    [User: 5.296 s, System: 1.177 s]
  Range (min … max):    6.664 s …  6.897 s    10 runs
 
Summary
  ./zig-out/bin/benchmark --impl=treelist ran
    3.59 ± 0.04 times faster than ./zig-out/bin/benchmark --impl=multiarray
```
But that’s just for running insertions.

If I add 10 traversals of all the trees in the database on top of that (_«10 full-table scans»_ in database lingo), we get a much bleaker picture:
```sh
hyperfine "./zig-out/bin/benchmark --impl=treelist" "./zig-out/bin/benchmark --impl=multiarray"
Benchmark 1: ./zig-out/bin/benchmark --impl=treelist
  Time (mean ± σ):      1.915 s ±  0.009 s    [User: 1.766 s, System: 0.149 s]
  Range (min … max):    1.907 s …  1.935 s    10 runs
 
Benchmark 2: ./zig-out/bin/benchmark --impl=multiarray
  Time (mean ± σ):     19.726 s ±  0.191 s    [User: 18.596 s, System: 1.014 s]
  Range (min … max):   19.520 s … 20.109 s    10 runs
 
Summary
  ./zig-out/bin/benchmark --impl=treelist ran
   10.30 ± 0.11 times faster than ./zig-out/bin/benchmark --impl=multiarray
```

I’m forcing the disadvantage so much that it’s almost unfair. Yet, the point is made: not everything that shines is gold - the cache locality effects promised by a multi-array list can quickly be destroyed if your structs are too heavy.



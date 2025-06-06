---
layout: post
title: "TreeList 3: how do I traverse?"
date: 2025-06-01
categories: [devlog, code]
tags: [array-pool, zig, devlog]
description: "A look into traversing trees in idiomatic zig"
toc: true
---

In the previous entries of this series, we looked at the fundamentals of a [database of GUI layout trees](https://github.com/AntoineBalaine/treelist).

In this entry, we’ll explore some of the decisions needed when designing the logic for traversing the trees.

This post is somewhat more focused on the tree algos. The subject has caused so much ink to flow, it’s practically become a meme, and I reckon it might be less interesting to the avid Zig practitionner. 

Nevertheless, I’m including this episode into the series, if only because it shows how indexing into tables can be done idiomatically.

# Traversal, with some grief.

- [Traversing using a state machine?](#traversing-using-a-state-machine)
- [Pre/Post order traversals, inverting the tree](#prepost-order-traversals-inverting-the-tree)
- [Morris traversal](#morris-traversal)
- [Sharing the stack across tree traversals](#sharing-the-stack-across-tree-traversals)
- [Iterator to the rescue](#iterator-to-the-rescue)
- [What about removing entries from one of the trees?](#what-about-removing-entries-from-one-of-the-trees)
- [I want to backtrack without a stack, though](#i-want-to-backtrack-without-a-stack-though)

## Traversing using a state machine?

For traversing, you can use a [state-machine-based approach](http://plasmasturm.org/log/453/), which keeps track of the current position into the tree by maintaining a stack of the previous nodes:

```
prevList[]
switch (prev_node){
    parent => goto left node,
    left => goto right node,
    right => go to parent
}
```

The problem is the order in which the elements need to be rendered: when it comes to [imgui](https://github.com/ocornut/imgui/) (the GUI library for this project), since it uses a draw stack, we want the background elements printed first, and the foreground elements printed last.

I tend to picture the draw process using breadth-first, since it maps to the way my n-ary tree represents the hierarchy: root contains a list of children which are really the background elements (colors, subpanels or subwindows), each of which might contain more interactive nodes (buttons, sliders, etc.).

```
// n-ary breadth-first drawing
      A
   /  |  \
  b   c   d
 /|\  |  /|
e f g h i j
```

Following this structure for the drawing requirements means that we must traverse depth-first:
```
left-child, right-sibling
     A
   /
  b----->c -->d
 /       |   /
e->f->g  h  i->j
```

## Pre/Post order traversals, inverting the tree

In my widget trees, a root only has children - typically the root might be a window or a top-level UI container.

If you iterate sibling-first, you can go to the bottom of the tree, and traverse it all the way back up - it is an inverted traversal, so you could do post-order traversal. If I traverse using a state machine, all I need to do is visit right nodes first:

```
switch(prev)
    parent -> goto right node
    right -> goto left node
    left -> go to parent
```

- without including the backtracks: `a b c d i j h e f g`
- with backtracks: `a b c d i j i d c h c b e f g f e b a`

NB: Incidentally, you _could_ also invert your binary tree with this: if you go bottom up and sibling first, you can invert the sibling/child locations all the way up to the root. There you go, you’ve now satisfied the coding interview meme…

This state-machine approach is very easy to grasp. It still doesn’t give me breadth-first, but that doesn’t matter so much for my gui tree: the entire idea is that I’m storing the data, separate from the drawing logic. Higher nodes represent container widgets, and lower ones represent input, decoration, and interactive widgets.

If I do pre-order traversal, depth-first, I’m still getting the containers drawn first, and their children drawn later - which is what we want when drawing elements in an immediate-mode GUI stack.

In my GUI tree, pre-order traversal visits containers before their contents, which matches the natural drawing order (draw the container, then draw what's inside it).

## Morris traversal

You could use the Morris algorithm: every time you reach a terminal in the tree, you update its sibling pointer (right-node pointer) so that it points to the next sibling of its parent. This way, you can traverse the tree without a stack, without recursion, and you do not need to backtrack when you reach a terminal node.

```
left-child, right-sibling
     A
   /
  b----->c -->d
 /       |   /
e->f->g  h  i->j
```

Here, `g` points to `c`, `h` points to `d`, so traversal order will be: `a b e f g c h d i j`.

To make things even cuter, I can mark whether a link points to a parent’s sibling using tagged unions:

```zig

pub const TreeList = struct {
    const Self = @This();

    // Link type with tags
    const Link = union(enum) {
        normal: Location, // Regular child or sibling link
        to_parent: Location, // Link back to parent
        to_parent_sibling: Location, // Link to parent's next sibling
    };

    // Node structure with tagged links
    const Node = struct {
        value: u32,
        child: ?Link = null,
        sibling: ?Link = null,
    };
```

This is however only possible because you’re traversing depth first.

- if I design it as `left-sibling, right-child`, it’s a little more difficult to picture:

```
// left-sibling, right-child
        A
      /
      b -> e
    /      |
   c->h    f
  /        |
 d->i      g
    |
    j
```

Here, `g` points to `c`, `h` points to `d`. For removals, you can reconstitute the parent context using a stack - though that’s not necessary for traversal.

That’s the Morris algo, ultimately. The child nodes’ left and right pointers (in our case, index-based `Locations`…) are being used to reference the predecessor node and successor node.
The logic can be represented as:

```
switch(prev)
    parent -> goto right node
    right -> goto left node
    left -> go to next parent sibling
```

If I try to rewrite this using a better-worded approach:

```
switch(prev)
  parent -> goto child
  child -> goto sibling
  sibling -> goto parent’s sibling
```


This leads to a reasonable traversal function:

```zig
pub fn traverse(self: *TreeList, root_idx: usize, visit: *const fn (*Node) void) !void {
    var current_idx = root_idx;

    while (true) {
        const node = &self.nodes.items[current_idx];
        // visitor pattern here
        visit(node);

        // Navigation logic using tagged links
        if (node.child) |child| {
            // Always go to child first if available
            switch (child) {
                .normal => |idx| current_idx = idx,
                else => unreachable, // Child should always be a normal link
            }
        } else if (node.sibling) |sibling| {
            // Then try sibling
            switch (sibling) {
                .normal => |idx| current_idx = idx,
                .to_parent => |idx| {
                    // Go back to parent
                    current_idx = idx;

                    // Skip visiting the parent again - we've already visited it
                    if (self.nodes.items[current_idx].sibling) |parent_sibling| {
                        switch (parent_sibling) {
                            .normal => |sibling_idx| current_idx = sibling_idx,
                            else => break, // End traversal if no normal sibling
                        }
                    } else {
                        break; // End traversal if no sibling
                    }
                },
                .to_parent_sibling => |idx| {
                    // Special sentinel value indicating end of traversal
                    // I hate encoding behaviour in data, though.
                    if (idx == std.math.maxInt(usize)) {
                        break;
                    }
                    // Go to parent's sibling
                    current_idx = idx;
                },
            }
        } else {
            // No more navigation options
            break;
        }
    }
}
```

But that’s an in-place change, which means that I’m going to have to change all those non-normal links back to normal at _some_ point, if I want to be implementing undo functionality - which would have to be done using structural sharing.

This is where my library looses in general-purposeness: I need the undo for my FX layouts. I want my app’s user to be able to create his own fx layouts using a GUI. This means that if he makes a mistake, he has to be able to undo at some point. So the tree is going to need to be represented across undo versions, and that is done with structural sharing.

The problem with structural sharing, is that those non-normal links would all instantly be invalid.

The Morris algo expects to be making these in-place changes only temporarily, but this feels redundant: if I make the changes temporarily, I’m going to have to need a stack-based traversal at some point anyway - which means two traversal implementations instead of one.

Yet again, a good old case of _the better is the enemy of the good_.

## Sharing the stack across tree traversals

This does not have to be a stopgap, though: I can share the stack for the whole tree list, and store it top-level.

```zig
pub fn TreeList(comptime node_types: anytype) type {
  return struct {
      const Self = @This();
      // Storage for each node type
      storage: Storage = undefined,
      // String pool for interning strings
      string_pool: StringPool = .empty,
      // Map from string refs to root nodes
      roots: std.AutoHashMapUnmanaged(StringPool.StringRef, Location(TableEnum)) = .{},
      traversal_stack: std.ArrayListUnmanaged(Location(TableEnum)) = .{},
      max_tree_height: usize = 0,
```

I can just reset its memory between traversals, without freeing it - then it’s just ready for re-use.

I care that traversals in the GUI be confidently done without allocations mid-loop, so I’m adding some extra sugar on top: the treelist will measure the max height every time a node is added to a tree:

```zig

pub fn addChild(
    self: *Self,
    parent_loc: Location(TableEnum),
    child_loc: Location(TableEnum),
    root_loc: Location(TableEnum),
    allocator: std.mem.Allocator,
) !void {
    const parent = self.getNode(parent_loc).?;

    child.sibling = parent_node.child;
    parent.child = child_loc;

    const height = try self.getTreeHeight(root_loc, allocator);

    // Update max height if needed
    if (height > self.max_tree_height) {
        self.max_tree_height = height;
        try self.traversal_stack.ensureTotalCapacity(allocator, self.max_tree_height + 1);
    }
}
```

This should not even be part of the `addChild()` function, though. This design sucks.

## Iterator to the rescue

If I switch to an iterator, I can consume it right upon instantiation, I can give a max depth, and it dodges the allocation.

```zig
/// Iterator for traversing the tree without allocations
pub const Iterator = struct {
    tree_list: *Self,
    current: ?Loc,
    // Fixed-size stack to avoid allocations
    stack: [MAX_TREE_HEIGHT]Loc = undefined,
    stack_len: usize = 0,

    /// Create a new iterator starting at a given root
    pub fn init(tree_list: *Self, root: Loc) Iterator {
        return .{
            .tree_list = tree_list,
            .current = root,
            .stack_len = 0,
        };
    }

    /// Get the next node in depth-first traversal (child first, then sibling)
    pub fn next(self: *Iterator) ?PtrUnion {
        const current = self.current orelse return null;

        // Get the current node pointer
        const node_ptr = self.tree_list.getNodePtr(current) orelse return null;

        // Prepare to move to the next node
        self.moveToNext(current, node_ptr);

        return node_ptr;
    }
  // […]
}
```

There’s an unknown about the max depth of the tree - is my `max_tree_height` going to fly ?

## What about removing entries from one of the trees?

Whenever I decide to remove a node, there’s only two ways to operate a removal: «swap remove» or «ordered remove».

«Ordered remove» means I move forward all the nodes after the current removal. For each of those nodes, if they have a parent, now the parent’s pointer is false. Also, if they carry children or siblings, those might have gotten moved to. So this falsifies the whole database.

«Swap remove» (swap with the last element in the backing array) falsifies only one pointer, but still falsifies it: fixing the falsification requires knowing where the parent is, and updating its pointer.

In order to do this, the node would have to store the location of its parent. This adds the extra overhead of storing an `Location` into the nodes themselves.

I could instead delay removals by marking tombstones into the arrays - but this means that tables would have to be compacted sooner or later, which brings about the same problem.

There’s other options like generational indices which are very ECS-like: you don’t swap the removed nodes, you just increment their generation, and that gives indications about whether or not a slot is available. That’s a viable approach when you need all the flexbility, but it’s pure overhead when you are working a known and limited set of types from the beginning.

## I want to backtrack without a stack, though

OK, fine, let’s store the parent-location inside the node:

```zig
// I keep this only for reference.
pub fn NodeInterface(comptime TableEnum: type) type {
    const Loc = Location(TableEnum);
    return struct {
        child: ?Loc = null,
        sibling: ?Loc = null,
        parent: ?Loc = null,
    };
}
```

Now I can traverse the tree up and down from any entry point without having to worry. I stick to my state-machine approach:

- if a node is a child, the `parent` field points to its parent.
- if a node is a sibling, the `parent` field points to its older-sibling.
- I can know whether a node is a sibling or child by comparing the parent’s `sibling` field.

This makes it possible to backtrack the whole tree. It also makes it possible to traverse siblings-first, without a stack.

With storing the parent location, both the sibling and the child know where the parent is. Now, depending on the order of traversal, the backtracking can decide whether we need to continue up or whether we need to backtrack to the other node of the parent:

```python
fn nextChild()
  visitChild()
  visitSibling()
  ascendToParentSibling(current_loc, current_node)


fn ascendToParentSibling(current_loc, current_node)
  parent = getParent(current_node.parent)
  if (parent.child.loc==current_loc)
    goto parent.sibling
    return
  if (parent.sibling.loc == current_loc)
    nextParentSibling(parent_loc, parent)

fn nextSiblingFirst()
  visitSibling()
  visitChild()
  nextParentChild(current_loc, current_node)

fn nextParentChild()
  parent = getParent(current_node.parent)
  if (parent.sibling.loc == current_loc)
    goto parent.child
    return
  if (parent.child.loc == current_loc)
    nextParentChild(parent_loc, parent_node)
```

That’s a lot of conditions to check, and your traversal is going to be slow - but it dodges the stack, and the logic is very simple.

If you have situations where writes are going to be rare enough and can afford to modify `Location`s in place, you could switch your traversals to using the Morris algo.

In the next entry in this series, we’ll be looking at benchmarks.


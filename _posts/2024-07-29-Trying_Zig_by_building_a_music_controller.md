---
layout: post
title: "Trying Zig by building a music controller"
date: 2024-07-29
categories: [Development, Audio]
tags: [zig, daw, midi, reaper, console1, programming]
description: "A deep dive into building a DAW extension in Zig for the Softube Console1 controller, covering build systems, MIDI implementation, and architecture decisions."
toc: true
---

[Why, oh why?](#why-oh-why)
   - [DAWs](#daws)
   - [The controller](#the-controller)
   - [The goal](#the-goal)

[Part 1 - The Build](#part-1---the-build)
   - [The C wrapper: forwarding C++ calls to Zig](#the-c-wrapper-forwarding-c-calls-to-zig)
   - [The Zig wrapper: forwarding Zig calls to C++](#the-zig-wrapper-forwarding-zig-calls-to-c)
   - [Building C++ from Zig](#building-c-from-zig)

[Part 2 - The App](#part-2---the-app)
   - [Architecting the app](#architecting-the-app)
   - [The controller's design, its midi implementation](#the-controllers-design-its-midi-implementation)
   - [Midi messages](#midi-messages)

[Part 3: Lifecycle of the extension](#part-3-lifecycle-of-the-extension)
   - [The config file format](#the-config-file-format)
   - [The config's global state](#the-configs-global-state)
   - [Side note on memory allocations](#side-note-on-memory-allocations)
   - [How fast should my app run?](#how-fast-should-my-app-run)
   - [Coding style guidelines](#coding-style-guidelines)

# Why, oh why?

I decided to build an integration for my Console1 midi controller. I’d been planning on making a record with a friend over the summer, I had the controller laying around, and figured that it’d be fun to use it to mix the album.

### DAWs:
Music softwares for recording are called DAWs - Digital Audio Workstations. You can compose, record, mix and master records with them.

### The controller:
As a product, Softube’s Console1 is a midi controller that comes with a DAW plug-in. The plug-in passes messages between the hardware and the DAW, and allows you to use virtual reproductions of recording studio consoles. It’s meant to be emulate the workflow of a record-mixing studio, in a budget and real-estate friendly way.

Since the software side uses a DAW plug-in, there’s a lot that can be done with it, but I found that it also imposed some big limitations. Plugins rely on a pre-set API. This API is great, but it can’t do everything. It has access to the audio threads of the host, it can have a GUI thread, and it has limited access to internal information of the host.

### The goal:
What I wanted was to control _all_ the internals of the host via the hardware controller. That’s also been a frequent complaint in audio communities about Console1: while the hardware was great, users found that the plug-in API didn’t allow for the deep integration they wanted. That’s the kind of thing that can make or break your dance with the muse when you’re trying to be creative.

In order to circumvent those limitations, I decided to re-implement some of the functionalities of the plug-in, with a few extra features. This involves creating the software layer between the host DAW and the hardware controller. This layer will come in the shape of a DAW extension, it tells both parties what to do, through a host-specific API. Since this API is vendor-specific, the project is very niche: you get expanded functionality, but only for one host vendor.

My DAW host of choice is called Reaper - there’s a lot of reasons that went into choosing it, the main one being that Reaper actually *does* have a programmable interface - most of its competitor vendors do not.

 Speaking of API. let’s look at how to hook into the host API. My host has a C++ SDK with 4000+ functions. I wanted to use zig instead of C++ for this project, so the first step was to write a wrapper for those of the API functions which I planned on using.

# Part 1 - The Build

## The C wrapper: forwarding C++ calls to Zig

### Issue n°1: C++ doesn’t have an ABI.
So, the SDK can’t interface directly with Zig. It has to do so through C. In Zig, interfacing with the C ABI is first class. So, to create the wrapper, I have to create some C++ header files that contain `extern "C"` blocks - those are meant to be read as C programs - and those `c` blocks can be read from zig.
```rust
// read_my_c.zig
const C_functions = @cImport("my_c_header_file.h");
```
### Issue n°2: `extern "C"` can’t export C++ classes
A lot of C++ data structures don’t exist in C. They’re unknown in that land. So, the unknown data structs which I plan to include in the `extern "c"` block need to be wrapped.
To do so, they can be represented  as void pointers in the header files (`void *some_cpp_class`). That’s easy. The pointers work in both C and C++, and the `void` is a little white lie to the C type system.
```cpp
// c_wrapper_header.h
extern "C"{
	typedef void *my_void_pointer;
	void wrapper_someClassMethod(my_void_pointer instance);
}
```
In the corresponding implementation file, I can cast the `void *` to the class it wraps, and call the methods.
```cpp
// c_wrapper_implementation.cpp
void wrapper_someClassMethod(my_void_pointer instance) {
  static_cast<MyCppClass *>(instance)->someClassMethod();
}
```
### Issue n°3: data alignment mismatches with struct method calls
C++ structs can carry methods. That’s not the case in C. Since I needed to access some C++ struct methods from Zig, I decided to re-declare them in Zig as extern struct.

```cpp
// some_cpp_lib_file_.h
struct SomeSruct{
	int prop1 = 0;
	someMethod(){}
} SomeStruct;
// call_the_cpp_lib.zig
const SomeStruct_wrapper = extern{
	int prop1: c_int = 0;
	pub fn someMethod() callConv(.C) void{}
}
const the_cpp_struct: SomeStruct_wrapper = c_wrapper_get_SomeSruct();
const x = prop1 + 1;
//		  ^~~~ SPECTACULAR FAILURE HERE
```

That didn’t work. I ended up having to write a C wrapper for the structs as well.
# The Zig wrapper: forwarding Zig calls to C++
Now that I can call some C++ code from Zig, I also need to do the reverse.
There’s no way to import Zig files into C++, but you can let the compiler’s linker find the zig functions: with another `extern "C"` block.
You declare:

```cpp
// some_zig_file.zig
export fn someZigFunction() callConv(.C)void{}
// zig_wrapper.h
extern "C" void someZigFunction();
void someFunction(); // this function will forward the call
// zig_wrapper_implementation.cpp
void someFunction() {
	someZigFunction(); // call the zig function. Magic!
}
```

And it works. That’s been the weirdest thing in this project. It tickles me that it’d be possible to use some functions without importing them - I suppose I just lack imagination, sometimes.
Speaking of the linker, let’s look at the build.
## Building C++ from Zig

If reading the following section makes you feel like a blind man in an anechoic chamber trying to find his way out, I feel you. Thankfully, Christian Fillion was there to pull me out of the mud on the Reaper discord - practically every day, for months. The man’s pretty much the dad of the dev community that gravitates around Reaper, and his work would deserve a whole book in itself.
### The zig build system

The zig build system’s pretty incredible, though I did have to cry for help a number of times to get it going. Comparing `build.zig` with the makefile hell that it takes to build other DAW extensions, I don’t regret choosing Zig. It’s really wonderful to have a build system that is entirely programmable in the same language as the project you’re trying to compile.

However, there’s been one thing I couldn’t do with it - and that wasn’t even zig’s fault.
### Linking against libCpp?
While you _can_ build all your C/C++ from your `build.zig` file - that’s done with `zig c++` - and you *can* link against libc and libCpp, you can’t link against libStdC++.
Some of the macro expansions in the `extern "C"` blocks  which were provided by the SDK were technically invalid, but allowed by libStdC++. Though Zig could link against libCpp, it couldn’t do so against libStdCpp - if I understand correctly, that’s also because C++ doesn’t have an ABI.
It goes further: the order in which some components of the SDK were being imported by my header files also affected the expansion of the macros. My build involved using a php script to create a `.rc` file that contained some data, which needed to be read by the expanded macros.

So, I found myself 6 weeks into the project with a broken build and a decision to make:
1. either rewrite the portion of the SDK that I needed. That way I could circumvent the macro-expansion issue, or
2. take my c++ build out of the hands of zig c++ and give it to gcc, using libStdCpp.
### Zig, tell GCC to build. Then build.
Option 1 wasn’t feasible: the SDK’s library was way too big and complex for me. Given I’d read my first C-header file a couple of months before, that wasn’t going to fly.
Fortunately, the later was possible. Zig allows you to coordinate its build with external commands. You can tell it to do stuff like «run this php command, then use gcc to take the output as argument, then tell gcc to compile my c++ files, then tell zig to link against the output, then tell zig to build».
Here’s a *very* incomplete example:

```rust
// build.zig
// run the php script that generates the resource.rc file
const php_cmd = b.addSystemCommand(&[_][]const u8{"php"});
php_cmd.addFileArg(b.path("SDK/generate_rc_file.php"));
php_cmd.addFileArg(b.path("src/cwrapper/resource.rc"));

// tell gcc to compile some cpp files
const cpp_cmd = b.addSystemCommand(&[_][]const u8{ "gcc", "-o" });
cpp_cmd.addFileInput(b.path("src/cwrapper/some_cpp_file.rc_mac_dlg"));
cpp_cmd.addFileInput(b.path("src/cwrapper/other_cpp_file.rc_mac_menu"));

// tell gcc to compile AFTER the php script is done running
cpp_cmd.step.dependOn(&php_cmd.step);
// tell zig to link against the GCC output
lib.addObjectFile(cpp_lib);
// tell zig «this is where your output goes»
b.step("default", "Build reaper_zig.so");
// tell zig to build AFTER everything else is done
step.dependOn(&lib.step);
```

Tada! The build works.

# Part 2 - The App
## Architecting the app
With the build in hand, now comes the question: «what are those hardware buttons supposed to do?»
### The controller’s design, its midi implementation
Console1 is divided into 5 knob sections and 1 buttons section.
The buttons are meant to trigger actions («play», «pause», «select a sound track», «mute it», «solo it», etc.), and the knobs are meant to control the sections of a mixing console. Those sections generally are:
- input: sets the volume coming from a recording, filters frequencies extraneous to human hearing
- sound gate: clears out the background noise in a microphone recording
- equalizer: shapes the tone of an instrument
- compressor: evens out the loud parts of a recording
- output: adds some subtle distortion, sets the volume coming out of the console
In addition, each knob module (save for the equalizer) features a meter - the input and output meters show the intensity of the signal, and the gate and compressor meters show the intensity of the changes they effect.
### Midi messages
In order to communicate with the software in the computer, the controller passes messages using the midi protocol. Every time the controller gets touched (button press or turning a knob), it sends a message. The format is super simple.
Each message is an array of 4 `u8` bytes - each representing values from 0 to 127: `<status> <channel #> <main data> <additional data>`. The bytes are most commonly represented in code as hexadecimals (e.g. `0xb0`). If one of the bytes doesn’t need to contain data, it’s just set to `0x0`.  Your typical message would like:
```rust
   0x8     0x12     0x7f  0x0
// status  channel  data  extra data
```
Each byte can act as a bit field - that’s most common for the status channel, which needs to communicate multiple types of message («is it a note?», «is it a knob?», «is it a sustain pedal?»).
If the controller needs to send a lot of messages in a row, it can skip the status byte to go faster. In that case, the receiver is expected to use the previous message’s status byte to interpret the message. That’s called «running status».
```rust
   0x12     0x7f  0x0         0x0
// channel  data  extra data  nothing
```
The console sets all its status bytes to `0xb0`. `0xb0` stands for «continuous controller» - or CCs. Continuous controllers are hardware controls like knobs and joysticks - they represent values that to change over time, as opposed to things like «note start/end», or  «piano pedal pressed/released».
The console only uses CCs, to make things simple: «all my messages are knobs. Midi doesn’t have button presses, so I’m going to represent the buttons as knobs, with values as `0x7f` for "pressed", and `0x0` for "released"».

The channel byte represents which of the knobs or buttons were pushed. Creating the mapping of controls-to-messages required some reverse engineering. I did it by logging the received messages to the CLI while I was turning each knob. There’s CLI programs like `receivemidi` and `sendmidi` on Github for that. The mapping enum ended up a long list:
```rust
pub const CCs = enum(u8) {
    Comp_Attack = 0x33,
    Comp_DryWet = 0x32,
    Comp_Ratio = 0x31,
	// etc.
}
```
The main data byte contains *absolute* values, meaning the value sent always corresponds to the position of the knob. By contrast, some other controllers send *relative* values, which basically say «here’s how far the knob position has moved compared to the previous position».

I tend to prefer relative values, because they allow you to measure the speed at which the knob is being turned - you can measure acceleration and deceleration with it. Console1 only has absolute values, oh! well…

# Part 3: Lifecycle of the extension
The extension works the following:
When my code gets loaded by reaper, it registers a callback with the host. This callback gets called twice: first when Reaper starts up, second when Reaper closes. Those are ideal times for allocating and de-allocating data.

### the config file format
Upon start-up, the app reads some user-defined config files. Those map the controls of the console to a list of audio plug-ins that Reaper needs to load. The contents of the files need to be allocated on the heap, and parsed into data structures that are usable by the app.

I chose to use INI as my config file format - it’s line-based, so my config parsers can use fixed-size buffers into which each line can be read. In practice, this minimizes the amount of allocation needed to parse, but the configs can’t be as structured as they would in JSON: INI only allows one level of nesting.

### the config’s global state
Once the data structures are in, they’re stored in the top-level state of the app. That portion is append-only: the default config is loaded at start-up, and the occasional missing per-plugin configs are loaded as we go. We’ll see if this approach proves satisfactory - loading as you go means you have to pause the controller mid-loop and allocate. If I don’t like it, I’ll revert to loading all the mappings on start-up.

Besides the configs, the app also allocates the C++ classes that do the interface between the host and my app. The classes contain:
- control surface: a list of subscription callbacks that let my app know whenever the user touches the host.
- midi Input: a message-passing interface that lets my app know whenever the user touches the console
- midi Output: an interface through which my app can tell the controller what to do («blink some LEDs», «move some knobs»).
Each of those classes carry destructors, which are called at exit
Also: upon exit, all of the config allocations, including the classes, are freed. 
### Side note on memory allocations
Reading the config files requires allocating their contents. While that could be a memory pit, I’ve found so far that the contents of the configs are pretty small - definitely less than 100MB.
Besides the memory consumption, allocating and de-allocating on the heap can be a performance pit for your app: your CPU is having to wait idle while you’re fetching data.

In a typical app, you’d want to be allocating and de-allocating data as you go. C libraries tend to do that implicitly by calling `malloc`.
Zig’s ethos is built around performance: all memory allocations and de-allocations are meant to be explicit. Library consumers choose when the allocations take place.
As a result, my app can choose its allocation/deallocation strategies according to technical requirements, and data structures. Having this type of freedom has huge benefits for performance and data structure design.

### How fast should my app run?
In the case of my project, I want the controls to remain as real-time as possible. The user needs to be unimpeded by slow-downs while tweaking the console. This is so he can stay in the creative flow as much as possible.
Also, I plan on adding a GUI to my app in the future. Unlike working with a DOM such as for web dev, GUIs make you manage the event loop yourself. This allows you to be in direct control of how data gets moved around. There’s potential for big mess ups as well as awesome perf-minded designs.

### Coding style guidelines
Allocating mid-run is not really an option. When processing messages, it is strictly forbidden to allocate or free the data. Some base rules:
- never allocate data mid-loop
- if I really need to allocate mid-run, the loop should get paused and the allocations take place when the user is in between actions (i.e. he just pushed a button, not a knob).
	*The only case in which I allow myself to do it, is if there’s a config mapping that’s missing and needs to be loaded.*
- when the loop starts and messages get processed, the allocations should have been already done
- minimize passing large data structures between function calls.
	*I try to structure the data as arrays or enum arrays and pass pointers instead. The zig compiler can also choose when it’s favourable to pass by copy or by pointer. The enum arrays approach is my favorite thing about Zig: I can keep string-indexed accesses, without the cruft of moving whole objects around. Also, the game industry made the switch away from OOP for similar reasons.*
- favor comptime-known values as much as possible. Yes, this reads as «prefer hard-coded values»
	  *I chose to avoid storing some of the data as config files for this specific reason (controller midi IDs, state machine mappings, etc.). This considerably restricts the scope of the project. I could have made my app usable by controllers other than the console 1, but the expanded scope would have restricted the avenues for perf optimization, reduced the flexibility of the app, plus there’s other apps that already do that.*
- avoid copying state as much as possible.
	*This implies that sometimes, some pieces of state have to be mutable. Though that’d be suicidal in a multi-threaded or async-heavy context, my app is single-threaded, doesn’t use side effects, and the entry points into the state are fairly limited. Configs are append-only.*
- avoid updating top-level state from side effects.
- top-level side effects that trigger other top-level side effects are absolutely forbidden.
	*Looking at you, web dev. Disregarding this base rule has been the root of un-maintainable code in every web app I’ve dealt with. There’s time and space complexity - well, I dunno if there’s a thing such as maintainability complexity, but I like to think that it’s something that would become measurable with the amount of side effects your app uses.*
- avoid mutable data as much as possible.
	*Thankfully, the Zig compiler cringes at me every time a non-`const` variable doesn’t get re-assigned.*
- absolutely no un-typed data.
	*Another thing I learned from web dev. The overwhelming majority of bugs I’ve had to fix in my short career came from poorly-typed data. Obviously, this is less relevant in a statically-typed language like Zig.*
- Avoid non-null terminated string slices.
	*This is specific to zig: the language can represent strings as un-terminated slices ( `[]const u8` ), which carry a length pointer to mark their bounds. They’re the preferred string format in the language, but they carry a limitation: in order to pass those slices to C apps, you have to copy them, append a null terminator, and stick a pointer in front of them ( `[*:0]const u8` ). Copying requires an allocation, and I want to avoid that. So, my preferred string format is going to be the null-terminated string slice ( `[:0]const u8` ) - it’s easy to cast into a c-string, and doesn’t require allocating a copy.*
- Don’t ignore runtime errors
	*You know murphy’s law: anything that can fail will fail, and should be designed around. Zig goes to great lengths to let the programmer know about any failure-prone operations: mem allocations, parsing, hash-table look-ups, etc. It’s not ok to just let the app blow-up in the users face.*

This list seems to be in contradiction with a lot of modern web dev practice, but it doesn’t have to be.

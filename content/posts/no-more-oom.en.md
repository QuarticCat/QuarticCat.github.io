---
title: "No more OOM in C/C++/Rust builds"
date: 2024-07-02
tags: [cpp, rust, optimization]
---

I've seen people complaining about some gigantic C/C++/Rust projects engulfing all of their memory during building from time to time. Fortunately, there are a few simple methods to alleviate the pain without sacrificing speed. By "simple," I mean you don't have to modify your code!

## Root of the problem

Usually, the most memory-consuming part of C/C++/Rust builds is the linking phase. To link object files, the linker must read all of them into memory. As more and more object files are linked together, the memory usage grows quickly.

What's worse, by default, DWARF embeds all debugging information into object files, which increases file sizes and thus the memory consumption. [GCC Wiki](https://gcc.gnu.org/wiki/DebugFission) mentioned that:

> In a large C++ application compiled with -O2 and -g, the debug information accounts for 87% of the total size of the object files sent as inputs to the link step, and 84% of the total size of the output binary.

## Ninja / CMake

In case you haven't heard about [Ninja](https://ninja-build.org/), it's a build system similar to Makefile. Different from Makefile, it's not for hand-writing but for meta build systems like CMake to generate. It's also much faster than Makefile.

Ninja has a cool feature called [pools](https://ninja-build.org/manual.html#ref_pool). Each pool represents a limitation on the number of jobs. You can assign each of your rules to a pool. With this feature, you can limit the number of concurrent linkers, say to no more than four.

If you simply use a smaller `-j` number, the parallelism of compiling jobs will also be limited. Using Ninja's pools gives you finer control.

As I just said, Ninja itself is not for hand-writing. Typically we invoke Ninja through CMake. A good news is that CMake provides two options on top of Ninja: [`JOB_POOL_COMPILE`](https://cmake.org/cmake/help/latest/prop_tgt/JOB_POOL_COMPILE.html) and [`JOB_POOL_LINK`](https://cmake.org/cmake/help/latest/prop_tgt/JOB_POOL_LINK.html). It's quite easy to apply them to your projects.

## Mold

[Mold](https://github.com/rui314/mold) is the fastest linker in the world. Many people are already aware of that. However, what they may not know is that Mold provides an environment variable [`MOLD_JOBS`](https://github.com/rui314/mold/blob/main/docs/mold.md#environment-variables) to control the parallelism of Mold processes.

> The primary reason for this environment variable is to minimize peak memory usage. Since mold is designed to operate with high parallelism, running multiple mold instances simultaneously may not be beneficial. If you execute N instances of mold concurrently, it could require N times the time and N times the memory. On the other hand, running them one after the other might still take N times longer, but the peak memory usage would be the same as running just a single instance.

A single linker instance can hardly cause OOM (Out-of-Memory). And you don't need to trade speed for this!

Compared to Ninja's solution, this one is faster and more widely applicable. Whatever build system and language you are using, just prepend the build command with `mold -run`. That's it.

## Split DWARF

DWARF has an [extension](https://gcc.gnu.org/wiki/DebugFission) allowing you to split most debug information into separate `.dwo` files, and optionally leave index to them in object files to speedup debugging. By doing so, linker's inputs are largely shrunk, saving you both time and memory.

To enable this feature in C/C++ builds, simply add `-gsplit-dwarf` to your compiler invocations. For example, in CMake you can use [`CMAKE_<LANG>_FLAGS`](https://cmake.org/cmake/help/latest/variable/CMAKE_LANG_FLAGS.html) or corresponding environment variables. To create the index, add `-Wl,--gdb-index` to your linker invocations.

For Rust builds, Cargo has this feature built-in already. Just set [`split-debuginfo`](https://doc.rust-lang.org/cargo/reference/profiles.html#split-debuginfo) to `unpacked` in your profile.

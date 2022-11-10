---
title: "How Do I Boost Difftastic By 3.8x"
date: 2022-10-06
tags: [rust, optimization]
---

[Difftastic](https://github.com/Wilfred/difftastic) is a structural diff that understands syntax. The diff results it generates are very fancy, but its performance is poor, and it consumes a lot of memory. Recently, I boosted it by 3.8x while use only 31% memory ([#393](https://github.com/Wilfred/difftastic/pull/393), [#395](https://github.com/Wilfred/difftastic/pull/395), [#401](https://github.com/Wilfred/difftastic/pull/401)). This post explains how I accomplished this. Hope it can bring you some inspiration.

When I start to write this post, not all optimizations are reviewed and merged. But I will keep updating.

## How Does Difftastic Work

The official manual has a [chapter](https://difftastic.wilfred.me.uk/diffing.html) describing its diff algorithm. Here's my understanding:

Say, we have two files. They are parsed to two syntax trees.

```diff
  -------------------------
  - A         |  - A
    - B       |    - B
      - C     |    - D
    - D       |      - E
      - E     |      - F
```

We want to match them. To do this, we have two pointers to syntax nodes in two trees, respectively.

```diff
  -------------------------
  - A  <-     |  - A  <-
    - B       |    - B
      - C     |    - D
    - D       |      - E
      - E     |      - F
```

In each step, we have some choices, and each of them represents a diff choice (an insertion, a deletion, unchanged, etc.). For example, here we can move both pointers one node further (pre-order traversal) since the nodes they point to are the same.

```diff
  - A
  -------------------------
    - B  <-   |    - B  <-
      - C     |    - D
    - D       |      - E
      - E     |      - F
```

Or we can move the left pointer one node further, which means a deletion.

```diff
- - A
  -------------------------
    - B  <-   |  - A  <-
      - C     |    - B
    - D       |    - D
      - E     |      - E
              |      - F
```

Or we can move the right pointer one node further, which means an insertion.

```diff
+ - A
  -------------------------
  - A  <-     |    - B  <-
    - B       |    - D
      - C     |      - E
    - D       |      - F
      - E     |
```

We keep moving pointers until finishing the traversal in both syntax trees. Then the moving path represents the diff result.

```diff
  - A
    - B
-     - C
    - D
      - E
+     - F
```

Obviously, there are multiple paths. To find the best one, we abstract it as the shortest path problem: a vertex is a pointer pair, and an edge is a movement from one pointer pair to another. Edge lengths are manually defined based on our preferences. For example, it is preferable to see two identical nodes as unchanged than one insertion plus one deletion, therefore the length of the former is shorter.

```text
       +-[1]-> (B, B) -> ... --+
       |                       |
(A, A) +-[9]-> (B, A) -> ... --+-> (EOF, EOF)
       |                       |
       +-[9]-> (A, B) -> ... --+
```

This is a simplified illustration. In the actual code, there are more kinds of edges, and the vertices contain more information.

You don't have to fully understand how it works internally to start optimizing. As long as your optimized code logically equals the previous one, it should be good. It is typical for your comprehension of the code to progressively advance throughout the optimization process. Of course, having a deeper understanding helps you identify more optimization opportunities.

## Benchmarking & Profiling

To perform optimization, you should first define a baseline, i.e., find a benchmark, and then profile it to locate hot spots and choose which code area merits your attention. Fortunately, the official manual also offers a [chapter](https://difftastic.wilfred.me.uk/contributing.html#profiling) that covers all we need.

In addition, I used [hyperfine](https://github.com/sharkdp/hyperfine) with some warm-up runs to improve the accuracy and stability of the result. Here's a [document of LLVM](https://llvm.org/docs/Benchmarking.html) that introduces some tips to reduce benchmarking noise. I didn't use all of them at that time, because I just knew this webpage recently. :)

There are many excellent profiling tools. Examples include [flamegraph](https://www.brendangregg.com/flamegraphs.html), which can be used for more than just CPU metrics, and [valgrind](https://valgrind.org/), which is especially useful for memory profiling.

Apart from those fancy tools, you can try commenting out some code pieces or adding useless fields to structs and then check the performance difference. This will quickly give you a rough estimate of the amount of profit you will gain from optimizing them. Valueless targets are not worth your attention.

Another trick is that you can print something into stderr and then filter them by `<command> >/dev/null 2>&1`. Following that, you can do counting by `| wc -l`, or find the minimum / maximum value by `| sort -n | head/tail -1`. It's sometimes more convenient than implementing the same functionality inside the program. You can use this trick to collect information like how frequently the control flow enters a code block.

## Optimizations

Here comes the main dish. I will select several commits and describe what I have done.

### Free Lunch

*Commits:*
[`06b46e9`](https://github.com/Wilfred/difftastic/pull/393/commits/06b46e935589117ac4583b6ca0d2a3020eb1c6b4)

There are some free optimization approaches you can have a taste first. I call them "free" because applying them is effortless, you don't even have to know about the code.

- LTO: In Rust, this can be enabled by a single line in `Cargo.toml`: `lto = "thin"` or `"fat"`. As for C++, if you are using CMake, then passing `-DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON` should work.

- PGO: For Rust, I recommend [cargo-pgo](https://github.com/Kobzol/cargo-pgo). For C++, you can follow instructions on [Clang's doc](https://clang.llvm.org/docs/UsersManual.html#profile-guided-optimization).

- [BOLT](https://github.com/llvm/llvm-project/blob/main/bolt/README.md)

- [Polly](https://polly.llvm.org/)

- Memory Allocator: The default memory allocator usually performs badly. Better alternatives include {je,tc,mi,sn}malloc and maybe more.

### Simple Simplifications

*Commits:*
[`d48ee2d`](https://github.com/Wilfred/difftastic/pull/393/commits/d48ee2dfdb59e98c71ce8fd99933268a1eee56ee),
[`3b0edb4`](https://github.com/Wilfred/difftastic/pull/393/commits/3b0edb43a1f79637e7d85073d111239e49fd5034),
[`b88625d`](https://github.com/Wilfred/difftastic/pull/393/commits/b88625d09b9f2641dcd4519ce538aebe6f648322)

According to my profile results, most memory was used for storing the `Vertex` type. Any simplification of this type will result in a huge improvement. I discovered that one of its fields implemented `Copy` but was wrapped by `RefCell`, which was overkilled. `Cell` was just enough. Moreover, I found that the internal third-party `Stack` type had a lot of fields that were useless for this problem. It was a very simple functional stack, so I just wrote a new one to replace it.

### Refactor Parent Stack

*Commits:*
[`2c6b706`](https://github.com/Wilfred/difftastic/pull/395/commits/2c6b7060a35a088e4a194fde4c51b18004f64c7f),
[`5e5eef2`](https://github.com/Wilfred/difftastic/pull/401/commits/5e5eef231d2b3a65e303d16c47964e7fcd38322a)

The original parent stack structure was obscure.

```rust
struct Vertex<'a, 'b> {
    // ...
    parents: Stack<EnteredDelimiter<'a>>,
}

enum EnteredDelimiter<'a> {
    PopEither((Stack<&'a Syntax<'a>>, Stack<&'a Syntax<'a>>)),
    PopBoth((&'a Syntax<'a>, &'a Syntax<'a>)),
}
```

```text
|  |l|                  |
|  |l|  |r|             |
|  |l|  |r|  PopEither  |
|-----------------------|
|   l    r    PopBoth   |
|-----------------------|
|   l    r    PopBoth   |
|-----------------------|
|       |r|             |
|  |l|  |r|  PopEither  |
|-----------------------|
```

`Syntax` here represented a syntax tree node. This structure was essentially two stacks that stored `&'a Syntax<'a>` with a constraint that two objects must be popped together if they were pushed together (marked as `PopBoth`).

I first refactored them into this form:

```rust
struct Vertex<'a, 'b> {
    // ...
    lhs_parents: Stack<EnteredDelimiter<'a>>,
    rhs_parents: Stack<EnteredDelimiter<'a>>,
}

enum EnteredDelimiter<'a> {
    PopEither(&'a Syntax<'a>),
    PopBoth(&'a Syntax<'a>),
}
```

```text
|PopEither(l)|  |PopEither(r)|
|PopEither(l)|  |PopEither(r)|
|PopEither(l)|  | PopBoth(r) |
| PopBoth(l) |  | PopBoth(r) |
| PopBoth(l) |  |PopEither(r)|
|PopEither(l)|  |PopEither(r)|
```

It was much more straightforward and consumed less memory. Then I noticed that each `Syntax` recorded its parent, we didn't need to replicate this information in the stack. Therefore, the parent stack could be further compressed to:

```rust
struct Vertex<'a, 'b> {
    // ...
    lhs_parents: Stack<EnteredDelimiter>,
    rhs_parents: Stack<EnteredDelimiter>,
}

enum EnteredDelimiter {
    PopEither,
    PopBoth,
}
```

```text
|PopEither|  |PopEither|
|PopEither|  |PopEither|
|PopEither|  | PopBoth |
| PopBoth |  | PopBoth |
| PopBoth |  |PopEither|
|PopEither|  |PopEither|
```

Here, one layer of parents represented one layer of [delimiters](https://difftastic.wilfred.me.uk/glossary.html). It's extremely rare to see 64 layers of delimiters. Also, the state space of the algorithm is about `O(num_syntax_nodes * 2 ^ stack_depth)`. When there are 64 layers of delimiters, the algorithm is unlikely to finish within a reasonable time and memory limit. Thus, I boldly compressed this stack into a bitmap.

```rust
struct Vertex<'a, 'b> {
    // ...
    lhs_parents: BitStack,
    rhs_parents: BitStack,
}

/// LSB is the stack top. One bit represents one `EnteredDelimiter`.
///
/// Assume the underlying type is u8, then
///
/// ```text
/// new:      0b00000001
/// push x:   0b0000001x
/// push y:   0b000001xy
/// pop:      0b0000001x
/// ```
struct BitStack(u64);

#[repr(u64)]
enum EnteredDelimiter {
    PopEither = 0,
    PopBoth = 1,
}
```

If this is too aggressive, I can turn it into a hybrid of bitmap and stack, which should still be much faster than `Stack<EnteredDelimiter>`.

### Tagged Pointers

*Commits:*
[`d2f5e99`](https://github.com/Wilfred/difftastic/pull/395/commits/d2f5e996b60465ee866ae13677930328740a14b3),
[`cb1c3e0`](https://github.com/Wilfred/difftastic/pull/395/commits/cb1c3e0ea38d31735b3f5433fe6582c137b22a2f)

`&Syntax` must be aligned to `size_of::<usize>()` as `Syntax` contains `usize` fields, which means some of its low bits are always zeros. We can use these bits to store information such as an enum tag. On x86-64 and some other 64 bits platforms, the top 16 bits haven't been used yet. We can store information there as well. But for portability, I didn't use those top bits.

By the way, I wrote a crate called [enum-ptr](https://github.com/QuarticCat/enum-ptr) dedicated to this trick.

### Skip Visited Vertices

*Commits:*
[`3612d08`](https://github.com/Wilfred/difftastic/pull/395/commits/3612d0844af5928416c2237e57a0c9ad000a0ceb),
[`9e11a22`](https://github.com/Wilfred/difftastic/pull/395/commits/9e11a2213ff9168500389145f34a6784dea9e89b)

In Dijkstra's Algorithm, once a vertex is extracted from the heap, its distance will not be relaxed anymore. A vertex might be pushed into the heap multiple times if it was relaxed multiple times and the heap is not capable of the decrease-key operation (see pairing heap and Fibonacci heap). We can mark a vertex as "visited" after it is popped and skip such vertices. You can also skip visited neighbors.

Be aware that things can change if you're using the A* Algorithm. If you are doing a graph search rather than a tree search, which is just the case of difftastic, and your heuristic is admissible but not consistent, then visited vertices should be marked as "not visited" after being relaxed. Besides, the radix heap will be unavailable since it requires the extracted elements follow a monotonic sequence.

My final code ran for ~290ms in the benchmark. Replacing the radix heap with std's binary heap results in an extra ~20ms. A heuristic must bring more speedup than that while keeping admissible. This is challenging. I've tried several ideas but never succeeded.

### Reserve Memory

*Commits:*
[`b0ab6c8`](https://github.com/Wilfred/difftastic/pull/395/commits/b0ab6c899a369cab73d79304002563fe08bae10d),
[`8625f62`](https://github.com/Wilfred/difftastic/pull/401/commits/8625f620af2c09390f2bde43bb9e8f7508afc8e6),
[`97e883d`](https://github.com/Wilfred/difftastic/pull/401/commits/97e883dd5fb263d5d91bb8edbf0618347b66557e)

Trivial.

### Reuse Memory

*Commits:*
[`8a0c82a`](https://github.com/Wilfred/difftastic/pull/401/commits/8a0c82ad623121d63cafeae7eedb3c0471861e7a)

After `9e11a22`, neighbor nodes were no longer required to be stored in vertices. They could be simply thrown away after the loop. The code logic became: create a neighbor vector, find neighbors, and return that vector. Then the profile result showed that creating vectors took a noticeable amount of time. So I created a vector outside the loop and reuse it in each round. This saved numerous memory operations.

### Exploit Invariants

*Commits:*
[`9f1a0ab`](https://github.com/Wilfred/difftastic/pull/395/commits/9f1a0ab1e606e7f32c22d42b6652555426c7b1fc),
[`d2f5e99`](https://github.com/Wilfred/difftastic/pull/395/commits/d2f5e996b60465ee866ae13677930328740a14b3),
[`5e5eef2`](https://github.com/Wilfred/difftastic/pull/401/commits/5e5eef231d2b3a65e303d16c47964e7fcd38322a)

The original `Vertex` was like:

```rust
pub struct Vertex<'a, 'b> {
    // ...
    pub lhs_syntax: Option<&'a Syntax<'a>>,
    pub rhs_syntax: Option<&'a Syntax<'a>>,
    lhs_parent_id: Option<SyntaxId>,
    rhs_parent_id: Option<SyntaxId>,
}
```

But I found that `lhs/rhs_parent_id` is `Some` only when `lhs/rhs_syntax` is `None`, respectively. Thus, they could be replaced by an enum and then compressed into a `usize` using the tagged pointer trick. This saved some instructions and memory footprint.

Later, I found that during the entire shortest path finding procedure, the syntax tree was pinned. Thus, we didn't need to obtain the unique `SyntaxId` at all. The pointer addresses were already unique. This further saved some instructions.

### Prune Vertices

*Commits:*
[`10ce859`](https://github.com/Wilfred/difftastic/pull/401/commits/10ce859bf4c465e3d32fa909c4f6150282910c05)
to
[`edc5516`](https://github.com/Wilfred/difftastic/pull/401/commits/edc5516eb90cc6c638e77140f2ec674378c04a73)

It's quite hard to explain this optimization in detail. It requires a deep understanding of the original implementation. I shall therefore simply introduce my ideas conceptually.

In brief, there were many vertices in the graph that held the following properties:

- One can only go to another
- The edges between them are zero weight.

For example,

```text
--+                                              +-->
  |                                              |
--+-> (l0, r0) -[0]-> (l1, r1) -[0]-> (l2, r2) --+-->
  |                                              |
--+                                              +-->
```

In this case, we can compress these vertices into one:

```text
--+                  +-->
  |                  |
--+-> (l012, r012) --+-->
  |                  |
--+                  +-->
```

By my estimation, this optimization shrank the state space by around 15%~25%, saving a huge amount of time and memory.

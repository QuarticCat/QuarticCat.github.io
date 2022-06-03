---
title: "RVV 也许没有想象的那么好"
date: 2022-02-27
tags: [risc-v, simd, vector]
---

## 引言

RISC-V 作为一个新兴的 ISA ，从前辈的错误中吸取了很多教训，同时提出了一些非常有吸引力的设计，被我的很多朋友喜爱。RISC-V 在我所接触的圈子里经常是“现代”与“优雅”的代名词，而它的向量扩展（RVV）也常常被冠以同等的荣耀，尽管几乎没人摸过有 RVV 的实机，也很少有人真的使用 RVV 进行编程过——这是当然的，毕竟 RVV Intrinsic 的编译器实现还很不完全。在把玩了一段时间 RVV 后，我感到它并没有像很多资料所推销的那么好。

我 SIMD 编程的经验并不多，Vector 架构更是在此前没有接触过。在批评一个自己不甚熟悉的事物时我总是感到惶恐的，还望轻喷。

## RVV 是怎么设计的

与常见的 SIMD 架构不同，RVV 的向量寄存器长度是可变的。这是指不同的芯片（准确地说是 hardware thread 或者 hart）可以拥有不同长度的向量寄存器，并且我们可以动态改变参与运算的长度。要做到这一点，需要程序在运行时通过某些指令获取和设置长度参数。不仅如此，RVV 的运算只区分 vector 与 vector 的运算 / vector 与 scalar 的运算，以及有无符号，并不区分元素长度，这意味着元素长度也是一个动态设置的参数。此外，RVV 允许我们只使用向量寄存器的一部分，或是把多个向量寄存器并起来使用，这又需要另一个动态参数。具体来说，一个向量寄存器的类型主要由以下几个参数共同决定：

- `VLEN`，向量寄存器的长度
- `vl`，一个 CSR（Control and Status Register），控制运算中实际用到的元素个数
- `vtype`，一个 CSR ，包括这几个部分：
  - `vill`，表示 `vtype` 设置是否合法
  - `vma`/`vta`，控制被 masked-off 的元素和尾部的元素的运算行为
  - `vsew`，控制单个元素的长度，我们用 `SEW = 8 | 16 | 32 | 64` 代表它的值
  - `vlmul`，控制一个操作使用多少个寄存器，我们用 `LMUL = 1/8 | 1/4 | 1/2 | 1 | 2 | 4 | 8` 代表它的值

值得一提的是这个 `LMUL`。当它小于 `1`，比如说 `1/2` 的时候，我们就只启用寄存器的一半长度。当它大于 `1`，比如说 `8` 的时候，我们就会把 8 个寄存器并起来一起用。

还有其他几个参数，这篇文章不会讨论到，就不列举出来了。

在汇编中，我们使用 `vset{i}vl{i}` 来同时设置 `vl` 和 `vtype`，然后再对向量寄存器进行运算。更详细的介绍可以看 [RVV Spec](https://github.com/riscv/riscv-v-spec) ，这里就不再赘述了。

## RVV C Intrinsic 的问题

在 RVV C Intrinsic 中，`vl`、`vma`、`vta` 都是在函数调用的时候指定的，而 `vsew`、`vlmul` 则是硬编码在数据类型里的，由编译器自动插入 `vset{i}vl{i}` 指令。于是我们有了下面这张恐怖的表（来自 [RVV Intrinsic RFC](https://github.com/riscv-non-isa/rvv-intrinsic-doc/blob/master/rvv-intrinsic-rfc.md#type-system)）：

> ### Data Types
>
> Encode `SEW` and `LMUL` into data types. We enforce the constraint `LMUL ≥ SEW/ELEN` in the implementation. There are the following data types for `ELEN` = 64.
>
> | Types        | LMUL = 1     | LMUL = 2     | LMUL = 4     | LMUL = 8     | LMUL = 1/2    | LMUL = 1/4    | LMUL = 1/8
> | ------------ | ------------ | ------------ | ------------ | -----------  | ------------- | ------------- | --------------
> | **int64_t**  | vint64m1_t   | vint64m2_t   | vint64m4_t   | vint64m8_t   | N/A           | N/A           | N/A
> | **uint64_t** | vuint64m1_t  | vuint64m2_t  | vuint64m4_t  | vuint64m8_t  | N/A           | N/A           | N/A
> | **int32_t**  | vint32m1_t   | vint32m2_t   | vint32m4_t   | vint32m8_t   | vint32mf2_t   | N/A           | N/A
> | **uint32_t** | vuint32m1_t  | vuint32m2_t  | vuint32m4_t  | vuint32m8_t  | vuint32mf2_t  | N/A           | N/A
> | **int16_t**  | vint16m1_t   | vint16m2_t   | vint16m4_t   | vint16m8_t   | vint16mf2_t   | vint16mf4_t   | N/A
> | **uint16_t** | vuint16m1_t  | vuint16m2_t  | vuint16m4_t  | vuint16m8_t  | vuint16mf2_t  | vuint16mf4_t  | N/A
> | **int8_t**   | vint8m1_t    | vint8m2_t    | vint8m4_t    | vint8m8_t    | vint8mf2_t    | vint8mf4_t    | vint8mf8_t
> | **uint8_t**  | vuint8m1_t   | vuint8m2_t   | vuint8m4_t   | vuint8m8_t   | vuint8mf2_t   | vuint8mf4_t   | vuint8mf8_t
> | **vfloat64** | vfloat64m1_t | vfloat64m2_t | vfloat64m4_t | vfloat64m8_t | N/A           | N/A           | N/A
> | **vfloat32** | vfloat32m1_t | vfloat32m2_t | vfloat32m4_t | vfloat32m8_t | vfloat32mf2_t | N/A           | N/A
> | **vfloat16** | vfloat16m1_t | vfloat16m2_t | vfloat16m4_t | vfloat16m8_t | vfloat16mf2_t | vfloat16mf4_t | N/A
>
> There are the following data types for `ELEN` = 32.
>
> | Types        | LMUL = 1     | LMUL = 2     | LMUL = 4     | LMUL = 8     | LMUL = 1/2    | LMUL = 1/4    | LMUL = 1/8
> | ------------ | ------------ | ------------ | ------------ | -----------  | ------------- | ------------- | --------------
> | **int32_t**  | vint32m1_t   | vint32m2_t   | vint32m4_t   | vint32m8_t   | N/A           | N/A           | N/A
> | **uint32_t** | vuint32m1_t  | vuint32m2_t  | vuint32m4_t  | vuint32m8_t  | N/A           | N/A           | N/A
> | **int16_t**  | vint16m1_t   | vint16m2_t   | vint16m4_t   | vint16m8_t   | vint16mf2_t   | N/A           | N/A
> | **uint16_t** | vuint16m1_t  | vuint16m2_t  | vuint16m4_t  | vuint16m8_t  | vuint16mf2_t  | N/A           | N/A
> | **int8_t**   | vint8m1_t    | vint8m2_t    | vint8m4_t    | vint8m8_t    | vint8mf2_t    | vint8mf4_t    | N/A
> | **uint8_t**  | vuint8m1_t   | vuint8m2_t   | vuint8m4_t   | vuint8m8_t   | vuint8mf2_t   | vuint8mf4_t   | N/A
> | **vfloat32** | vfloat32m1_t | vfloat32m2_t | vfloat32m4_t | vfloat32m8_t | N/A           | N/A           | N/A
> | **vfloat16** | vfloat16m1_t | vfloat16m2_t | vfloat16m4_t | vfloat16m8_t | vfloat16mf2_t | N/A           | N/A
>
> ### Mask Types
>
> Encode the ratio of `SEW`/`LMUL` into the mask types. There are the following mask types.
>
> n = `SEW`/`LMUL`
>
> | Types | n = 1    | n = 2    | n = 4    | n = 8    | n = 16    | n = 32    | n = 64
> | ----- | -------- | -------- | -------- | -------- | --------- | --------- | ---------
> | bool  | vbool1_t | vbool2_t | vbool4_t | vbool8_t | vbool16_t | vbool32_t | vbool64_t

注意到这里有很多的 N/A ，这是因为 RVV Spec 对 VLEN 的要求非常宽松，只要求至少能容得下一个最大元素（`VLEN >= ELEN`）即可，导致这些类型可能在某些芯片上无法使用（例如一款 RV64V 芯片的 `VLEN = 64`，则我们无法使用 `SEW = 64, LMUL = 1/8` 的 `vtype`），因此 Intrinsic 有很多的类型无法提供。不过这些类型看起来也并不是很重要，因为 `LMUL < 1`的情况似乎本来就不常见，一般是需要 [Widening Vector Arithmetic Instructions](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#102-widening-vector-arithmetic-instructions) 或 [Narrowing Vector Arithmetic Instructions](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#103-narrowing-vector-arithmetic-instructions) 操作的时候才会用到。而这些操作不会用到没有的那些类型，所以这大概不算是一个问题。

另外，尽管这份 RFC 本身就提供了很多重载函数，但糟糕的是它的重载函数命名基本对标汇编指令，非常多本可以重载在一起的函数没有重载在一起，当你尝试对它们进行包装时还需要做很多额外的判断，详情可以见[这个 issue](https://github.com/riscv-non-isa/rvv-intrinsic-doc/issues/117)。当然这一点未来应该是可以改进的。

这都不是什么严重的问题，严重的问题在于，由于 RVV 的变长特性，这些 Intrinsic 类型都是变长类型（sizeless types / unsized types）。

## 变长类型的生态困境

首先，C 语言标准中其实是有变长类型的，那就是 [VLA](https://en.cppreference.com/w/c/language/array#Variable-length_arrays)。当 VLA 被构造时，会动态地扩展当前栈帧。但与 `alloca` 不同的是，作为变量，VLA 有生命周期，编译器可以去分析复用栈内存。RVV Intrinsic 类型目前在 LLVM/Clang 上的实现就非常类似于 VLA 。

然而问题来了，C++ 标准压根不支持 VLA（尽管 GCC 和 Clang 有相应扩展），也没有其他这样的变长类型。Rust 里尽管有变长类型（比如 `dyn Trait` 和 `[T]`），但只允许以引用的形式持有，而且完全没有动态扩展栈的能力。在这两门语言中想要支持 RVV Intrinsic ，恐怕会有很多的问题需要考虑，需要做很多的特殊处理。比如把它作为静态存储期变量（如全局变量和 static 变量）要怎么办，放在 struct 里面要怎么办，传参要怎么办，作为返回值要怎么办。这些都是 VLA 也做不到，而其他所有的 SIMD Intrinsic 类型甚至都不需要考虑的事情。

目前 C++ 与 Rust 的代码几乎全部都是针对定长类型写的，只要变长类型不能完全像定长类型一样使用，几乎可以肯定它会在某些地方出问题。例如在某个利用不求值语境的 expression SFINAE ，再比如某个 constant expression 里的 `sizeof` 调用。我们在太多的地方无意识地依赖于“定长”这一特性。而且，“变长”还是一个会“染色”的特性。如果一个 struct 含有变长成员，那么它也会变成变长类型。换言之，如果一个 struct 的调用链中有任何地方依赖于它的定长特性，你就无法让它持有变长成员。

如果我没记错的话，目前 LLVM/Clang 的实现是只允许作为局部变量、传参和作为返回值的，其他上面提到的都不允许。GCC 对 RVV 的支持进程则是长期停滞在了一个很不完全的状态。其实我们从编译器对 ARM SVE 的支持就大概可以窥见我们目前能做到什么程度了，而这个程度不太理想。

Rust 方面更惨，完全不支持。就如前面所说，Rust 目前完全没有动态扩展栈的能力。这里有[一篇文章](https://poignardazur.github.io/2022/02/23/rust-unsized-vars-analysis/)介绍了 Rust 对变长类型的支持现状，看起来推进得很缓慢，希望 async 方面的需求能成为一个大的推动力，或者在这之前 Rust 能单独给 RVV Intrinsic 类型开洞吧。

实际上我非常怀疑目前是否有任何一门语言可以很好地给变长类型建立**零开销抽象**。变长类型的生态困境意味着 RVV 在高级语言层面将会严重缺乏一致性和组合性。

也许问题没有我说的那么严重，因为通常来说使用 SIMD/Vector 的地方都是比较特定的，即使有这么多的操作都不支持，也可能不影响你使用它。但目前大多数 SIMD 库的作者就得头疼这个问题了，他们的库设计可能根本不允许他们添加对 RVV 的（完整）支持。除了有很多 SIMD 和 Vector 的架构差异问题外，最基本的，你甚至无法把一个 RVV Intrinsic 类型包到 struct 里。要头疼的还有 GCC 和 Clang 的向量扩展，它们完全是定长的。而你如果作为这些东西的使用者，也就难以低成本地支持 RVV 了。顺带一提，Rust 的 `std::simd` 目前直接不考虑支持 RVV 。

## 类型选择难题

在 SIMD 编程里，类型的选择通常是**比较**容易得出结论的，但 RVV 不是。如果你选择大的 `LMUL`，那么你的可用寄存器就更少，带来更大的 spill 概率。如果你选择小的 `LMUL`，那么你就可能得到较低的效率。看起来 `LMUL` 的选择像是一个调参工作，并且这个工作是 RVV 独有的，SIMD 库恐怕难以对其建立跨平台的抽象。

还有一种情况我个人并不是十分确定，就是 SIMD 编程有时强烈依赖内联优化（特别是对于那些包装库），而一个单独看不会 spill 的函数可能在内联到另一个函数后出现了 spill 。假如这是一个符合现实的情况的话，这个调参过程将会变得更加麻烦。

此外还要注意的是，widening 和 narrowing 操作会修改 `SEW` 和 `LMUL`，有时 `LMUL` 的选择还要考虑你是否需要用到这些指令。这里就有一个[例子](https://github.com/riscv/riscv-v-spec/blob/master/fraclmul.adoc)。

我特别好奇目前尝试支持 RVV 的那些 SIMD 库（比如 Google 的 [highway](https://github.com/google/highway)）要怎么处理 `LMUL` 的选择问题。在我写这篇文章的时候，highway 的 RVV 支持仍在进行中。

## 可能更长的上下文切换时间

《RISC-V 手册》中骄傲地宣称，RVV 可以避免传统 Vector 架构上下文过大的问题，因为 RVV 有一个指令 `vsetdcfg` 让程序按需启用寄存器，因此切换上下文时不必包含那些没用到的寄存器。然而这本书对于 RVV 的介绍已经过时很久了。这个指令已经在现在的 RVV Spec 中被废弃了。目前只有一个 `mstatus.VS`（以及其他 priviledge level 的对应状态）对于减少上下文切换开销有帮助。但这是一个非常粗粒度的机制，它只记录向量寄存器整体是否被改写过，一旦上下文切换发生并且使用了任意向量寄存器，我们仍然需要保存所有的向量寄存器。如果我没有理解错的话，这是说传统 Vector 架构中上下文过大的问题仍然没有得到很好的解决。

## RVV 真的可以模拟 SIMD 吗

我曾经在某些地方看到有人声称 RVV 也可以模拟 SIMD，我觉得没有这么简单。

虽然我们可以通过 `vl` 来设置固定长度，但考虑 `VLEN` 的多样性，这样做要么得拉大 `LMUL` 并得到更大的 spill 概率（和可能的效率损失？），要么得放弃一些 `VLEN` 较小的平台。实际上我认为后者是 RVV 的设计者们期望的做法，因为他们搞出了[`Zvl*`系列扩展](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#181-zvl-minimum-vector-length-standard-extensions)，这系列扩展限定了向量寄存器的最小长度。

或者，放弃二进制跨平台，仅支持源码跨平台，在编译期根据 `VLEN` 选择类型？再要么，像一些 SIMD 库那样做动态派发，为几个 `VLEN` 范围的平台同时生成代码，根据平台动态选择执行路径？看起来总得做一些牺牲。而且目前 LLVM/Clang 的实现并没有在编译期提供 `VLEN` 的值，似乎也无法用宏判断支持哪些 `Zvl*` 扩展，所以只是理论上可行。

RVV 描绘的愿景很美好，不同芯片，不同寄存器长度，同一套指令集，即使以后想要扩展寄存器长度也不需要新增指令。但他们给出的例子都是处理一串很长的数据，寄存器长度的变化只是影响循环的次数，因此这些汇编可以完美地跨所有支持 RVV 的平台而不需要任何额外努力。然而可以做数据并行的地方并不止这种场景，[这篇文章](https://www.sigarch.org/simd-instructions-considered-harmful/)下的一段评论很贴合我现在的想法：

> If you work with long dense vectors and nothing else, you don't need any CPU instructions. GPGPUs win performance and power efficiency by a factor of magnitude.
>
> Current SIMD can be used for more than that.
>
> A register can be treated as a complete small vector as opposed to a chunk in a long vector. Try implementing 3D vectors cross product with your approach and you'll see.
>
> A register can be treated as a 2D bitmap as opposed to vector, here's an example: https://github.com/Const-me/SimdIntroArticle/blob/master/FloodFill/Vector/vectorFill.cpp#L135-L138
>
> -- Soonts

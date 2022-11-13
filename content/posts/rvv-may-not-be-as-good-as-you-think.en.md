---
title: "RVV May Not Be as Good as You Think"
date: 2022-02-27
tags: [risc-v, simd, vector]
---

As an emerging ISA, RISC-V learns a lot from its predecessors' mistakes and brings some very appealing designs. In my circles, RISC-V is frequently associated with the words "modern" and "elegant". Its vector extension (RVV) is often given equivalent praise, even though nearly no one has used a real-world RVV machine (including me) or even programmed in RVV. After experimenting with RVV for a while, I feel that it is not as good as many people claimed.

## How Is RVV Designed

In contrast to SIMD architectures, RVV has variable-length vector registers. That means different chips (hardware threads, or harts, to be precise) can have different vector register lengths while sharing the same instruction set. To accomplish this, the software must get and set the length parameter with some instructions at runtime. RVV operations distinguish only between vector-vector and vector-scalar, signed and unsigned, but not element lengths. As a result, the element length is a dynamic parameter as well. Furthermore, RVV enables us to use only a portion of a vector register or to combine multiple vector registers, which necessitates the usage of a dynamic parameter. The type of a vector register is mainly governed by the factors listed below.

- `VLEN`: a constant, represents the length of a vector register on this chip
- `vl`: a *Control and Status Register*, or CSR, controls the number of elements used in operations, making it easier to handle the tail elements of an array
- `vtype`: a CSR, includes
  - `vill`: represents whether the `vtype` configuration is ill-formed or not
  - `vma` / `vta`: controls the operation behavior of those masked-off elements and tail elements
  - `vsew`: controls the length of a single elemnt, represented by `SEW = 8 | 16 | 32 | 64`
  - `vlmul`: controls how many registers are used in an operation, represented by `LMUL = 1/8 | 1/4 | 1/2 | 1 | 2 | 4 | 8`

We use `vset{i}vl{i}` instructions to set both `vl` and `vtype`.

I need to elaborate on the parameter `LMUL`. When `LMUL = 1/2`, for example, we only use half of the registers. When `LMUL = 8`, we combine 8 contiguous registers into one, resulting in 32 / 8 = 4 accessible registers. Since there are only 5 bits for a register index in all RVV instructions, we don't receive more registers when `LMUL < 1`.

There are a few other parameters. However, they will not be discussed in this blog, so I will not list them.

More details can be found in the [RVV Spec](https://github.com/riscv/riscv-v-spec). I'll end my introduction here.

## Annoyances of RVV C Intrinsics

In RVV C intrinsics, `vl`, `vma`, `vta` are specified at function invocations, whereas `vsew`, `vlmul` are hard-coded into types. Compilers are responsible to insert `vset{i}vl{i}` instructions for you. We now have the following horrible table (source: [RVV Intrinsic RFC](https://github.com/riscv-non-isa/rvv-intrinsic-doc/blob/master/rvv-intrinsic-rfc.md#type-system)).

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

There are a lot of N/A here, which makes it a little difficult to generate code with C macros. This is because the RVV specification has a loose `VLEN` restriction, requiring just that it can contain at least one largest element (i.e. `VLEN >= ELEN`). As a result, these N/A types may not be available on some chips (for example, an RV64V chip with `VLEN = 64` cannot support `SEW = 64, LMUL = 1/8`). These types don't seem to matter much, though, because the `LMUL < 1` case seems to be uncommon, and is usually used in [widening instructions](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#102-widening-vector-arithmetic-instructions) or [narrowing instructions](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#103-narrowing-vector-arithmetic-instructions), which do not use those N/A types.

Thanks to `LMUL`, the amount of intrinsic types is huge, making the size of the header file and docs megabytes large. The good news is that RVV inrtinsics provide [overloaded functions](https://github.com/riscv-non-isa/rvv-intrinsic-doc/blob/master/rvv-intrinsic-rfc.md#overloaded-interface). But the names of these functions are essentially mapped to assembly instructions. There are a lot of functions that could be overloaded together while they aren't. When you try to wrap them, you still need to do a lot of extra work, as stated in [this issue](https://github.com/riscv-non-isa/rvv-intrinsic-doc/issues/117).

These are just some small annoyances during my experience of RVV. I won't use them to criticize RVV. The major problem is that these intrinsic types are all *dynamically sized types* (or *sizeless types*, or *unsized types*) due to RVV's variable-length nature. And I'm afraid that DSTs are poorly supported in all languages, not just C.

## The Ecosystem Has Not Prepared for DSTs

First of all, the C language standard actually has a DST, i.e., the [variable-length array](https://en.cppreference.com/w/c/language/array#Variable-length_arrays). When a VLA is constructed, the current stack frame will be dynamically extended. It's like `alloca` with some extra information such as type and lifetime. The implementation of RVV intrinsic types in Clang is very similar to VLA.

However, while being supported as a compiler extension by GCC and Clang, VLA is not part of the C++ standard. C++ standard does not have any DST. As for Rust, although it does have DSTs like `dyn Trait` and `[T]`, they can only be held indirectly using references or pointers. Dynamically extending stack frames is not possible with Rust at all. To properly support RVV, a number of issues must be taken into account, and maybe many changes must be made in these two languages. Consider these: How do you store RVV variables as static variables? How do you put them in a struct? How do you pass them as arguments to a function and return them from a function? None of these actions can be done on VLA variables. They are so fundamental and natural to any other SIMD type, but they pose a significant challenge to RVV.

Currently, the vast majority of existing C++ code are written against statically sized types. We rely on the static sizes in so many places without awareness. Have you ever think of that a `sizeof` in some constant evaluation context may break? What's worse, DST is [colored](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/). If a struct contains a DST, then it is a DST, too. The suffering and sorrow spreads along the dependency chain.

So what is the status quo? If I recall properly, LLVM/Clang only allows RVV types to be used as local variables, arguments, and return values. Other than these, none of the aforementioned uses are allowed. While GCC's support for RVV has stuck in an intermediate state for a long time (9/29/2022 update: [GCC has supported RVV v1.0](https://github.com/riscv-collab/riscv-gnu-toolchain/issues/778)). As RVV is not the first vector architecture implementation, we can investigate the language support status for its predecessor, ARM SVE, to see how far we can go. Well, the support is, not less constrained than RVV.

Is Rust better? Rust apparently possesses more language facilities for DSTs. It has a `Sized` trait used everywhere, implicitly or explicitly. And it permits structs with DST fields as long as they are the final ones (which is weird since Rust doesn't guarantee the memory layout). So I believe that the process of Rust embracing RVV will be smoother. However, as was already said, Rust is unable to dynamically extend stack frames, making it impossible to even put a DST variable on the stack.

Actually, I seriously doubt that a language exists that supports DSTs well and can build *zero-cost abstractions* on top of them. Lack of ecosystem support suggests that RVV will be less consistent and composable in terms of language level.

Maybe I've taken the problem too seriously, because scenarios that uses SIMD/Vector are quite specific. Even though RVV lacks so much abilities, you probably won't get affected. But those SIMD library authors will. Their code design might automatically reject RVV. Putting aside the distinctions between SIMD and vectors, there is a more pressing issue: RVV types cannot be wrapped in a struct. In addition to SIMD libraries, GCC and Clang's vector extension is also hard to support RVV as it requires you to specify the sizes at compile-time. That indicates that there isn't a lot of SIMD-accelerated code can support RVV cheaply. By the way, Rust's `std::simd` simply gives up supporting RVV for now.

Only Google's [highway](https://github.com/google/highway) pronounces support RVV, as far as I'm aware. However, some of its modules, such as vqsort, don't. It is common for other sorting networks to employ transposes, but vqsort's sorting network uses a number of permutations to avoid transposing, making it extremely challenging to convert to RVV because it is length-agnostic. It seems that [nsimd](https://github.com/agenium-scale/nsimd) tried to support RVV but stopped a long time ago.

## Choice of Intrinsic Types Is Not Clear

The SIMD type to employ is typically obvious. Numerous SIMD libraries, such as C++'s experimental `<simd>`, can choose an underlying SIMD type for you automatically. For instance, on x86 platforms, the fallback order is commonly AVX512 -> AVX2 -> SSE2, despite the fact that the time required to switch between licenses and the degree of downclocking of AVX512 and AVX2 differ amongst microarchitectures. And since you simply need to take into account the element type, it is also evident for ARM SVE. However, things become considerably more confusing in RVV.

Recall that RVV has a `LMUL` parameter. Larger `LMUL` values are expected to increase speed at the expense of the number of available registers, which suggests a higher likelihood of spilling. You might need to tune `LMUL` to get a higher efficiency. I initially believed it to be a special procedure that only RVV possesses. But after a while, I noticed how similar it is to the loop unrolling problem.

To some extent, compilers can decide how to unroll loops for you. Then what about `LMUL`? Can compilers choose a proper value for you? I don't know. But I think this is not as easy. Because `LMUL` is not an opt-in feature. You cannot pretend it always equals to one. Widening and narrowing instructions (e.g., convert `u32` to `u64` or the reverse) will change `LMUL`. Taken that into consideration, the optimizing process could be more complex than loop unrolling. It reminds me of [how RISC-V's genius design bring problems to linker implementations](https://maskray.me/blog/2021-03-14-the-dark-side-of-riscv-linker-relaxation).

If the compiler is unable to select an appropriate `LMUL` for you, then you have to tune `LMUL` manually. Another issue now: can SIMD libraries offer a unified, cross-platform API over it? That is challenging, in my opinion, and I haven't come across any related design.

Given the resemblance between selecting `LMUL` values and unrolling loops, I have to wonder if `LMUL` is really essential. Will the speedup warrant the additional complexity it adds? We don't know because RVV hardware is currently scarce.

## Possible Higher Context Switch Cost

The context size issue plagues older vector processors (according to some articles, though, I'm not familiar with that period of history). Their vector registers are typically made to be long in order to achieve a high speedup. This method will undoubtedly bloat the context size. As a result, operating systems must spend additional time and resources on register saving during context switching.

*The RISC-V Reader*, a popular resource for RISC-V newcomers, proudly claims that RVV can avoid this problem, because RVV has a dedicated instruction `vsetdcfg` that can enable / disable registers by need, so that we can only pay for what we use. Sounds promising, doesn't it? However, *The RISC-V Reader* is very out-dated. The instruction `vsetdcfg` has already been deprecated in the current RVV spec. RVV now only has a very coarse-grained mechanism that records whether any vector register is modified or not. If the vendor chooses a long-length implementation, I believe RVV will also experience the context size issue. This problem is unlikely to bother you though. Super long vectors are usually designed for HPC, of which the resource is usually dedicated to one single program at a time.

## Can RVV Emulate SIMD?

Some blogs and talks say that RVV can, at worst, emulate SIMD. THIS IS NOT TRUE.

Think of that, how can we use a stuff with less information (register sizes only known at run-time) to emulate a stuff with more information (register sizes known at compile-time)? To expose unified APIs to users, the only feasible way is to erase the extra information from SIMD types. And this is what highway does.

But can't we set the desired size at run-time to match with SIMD? One might ask. No, you can't. You must first deal with the DST issue, as I elaborated. And after that, you have to deal with the big variety of `VLEN`. Recall that the RVV Spec only stipulates `VLEN >= ELEN`. So in some processors your `vl` settings might fail. What if we use `LMUL` to concat registers? Well, then you plunge into the type choosing problem.

A way out is to use [`Zvl*` extensions](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#181-zvl-minimum-vector-length-standard-extensions), which specifies the minimum vector register length. Theoretically, you can use feature flags (e.g., pre-defined macros like `__AVX2__` in C++) to pick different implementations for different `VLEN` at compile-time or simply reject those platforms that don't have sufficient length. Of course, once you do this, you lose the portability advantage of RVV. And even you use only a part of the register, the context size won't be smaller. And as far as I know, such flags haven't been implemented in LLVM/Clang yet.

That's not the end. Despite all the challenges, there are some situations where SIMD emulation is still impossible (or too expansive), for instance, permutations. RVV does have some permutation instructions like slideup, slidedown, gather, scatter, and compress. And they are very helpful. You can also see some of them in AVX512. But SIMD's general permutation instructions can do more than that; they accept a lookup table to rearrange elements within a register, which is not possible for RVV. As the length is agnostic, the lookup table size is unknown. [Simdjson](https://github.com/simdjson/simdjson) utilizes permutations to classify characters. Vqsort utilizes permutations to implement its sorting network. Neither of them can be cheaply emulated by RVV.

---
title: "How To Read C++ Types"
date: 2021-02-13
tags: [cpp]
---

The readability of C++'s types is terrible, and most beginner tutorials don't go into detail about how to read them. They at most discuss the difference between *top-level const* and *low-level const*. Many of my friends have asked me about this, and I've talked about it numerous times. So I thought, why not write a blog post on it?

## A Common Misunderstanding

Before talking about type reading in detail, I'd like to correct a common misunderstanding.

Q: What's the type of `a` in `int a[5]`?

A: `int[5]`, and will *decay* to `int*` when appropriate.

Q: What's the type of `a` in `int a[5][6]`?

A: `int[5][6]`, and will *decay* to `int(*)[6]`, the pointer to `int[6]`, when appropriate.

Because of the existence of *decay* rules, many C++ beginners consider array types and pointer types the same. But they are different.

Similarly, function types will also *decay* to pointer types. For instance, `int(int, int)` will *decay* to `int(*)(int, int)`.

## Solving Equations

You might have seen some people saying that reading C++'s types is just solving equations. Let me explain this statement.

Leaving aside CVR (const, volatile, reference), the most basic declarations in C++ can be divided into two parts: the **type names** written on the leftmost are the *type specifiers*, and the rest are the *declarators*. These parts are inherited from C. They are the nastiest component of C++ types. These two names are not intuitive, I like to refer them as "return types" and "expressions", as will be explained later. Let's first start with a few examples.

| Declaration | Type Specifier | Declarator |
|:-:|:-:|:-:|
|`int a`|`int`|`a`|
|`int* a`|`int`|`*a`|
|`int a[5]`|`int`|`a[5]`|
|`int* a[5]`|`int`|`*a[5]`|
|`int (*a)[5]`|`int`|`(*a)[5]`|

After separating these two parts, it is now easy to understand how the types of `a`s above are determined: using `a` as the form of the *declarator*, then the type of the return value is just the *type specifier*. The so-called equation solving is just such a process:

1. From `int *a[5]` we have that the type of `*a[5]` is `int`.
2. That is, dereferencing `a[5]` returns a `int`, which means the type of `a[5]` is `int*`.
3. That is, indexing `a` returns a `int*`, which means the type of `a` is an array of 5 `int*`。

Another example:

1. From `int (*a)[5]` we have that the type of `(*a)[5]` is `int`.
2. That is, indexing `(*a)` returns a `int`, which means the type of `(*a)` is `int[5]`.
3. That is, dereferencing `a` returns a `int[5]`, which means the type of `a` is a pointer to `int[5]`.

You can see that the operator precedence here is exactly the same as in expressions, and the form is basically the same as well.

One more example:

1. From `int (*a)(int, int)` we have that the type of `(*a)(int, int)` is `int`.
2. That is, calling `(*a)` in the form of `(int, int)` returns a `int`, which means the type of `(*a)` is a function that has two `int` parameters and returns a `int`, i.e., `int(int, int)`.
3. That is, dereferencing `a` returns `int(int, int)`, which means the type of `a` is a pointer to `int(int, int)`.

A more complex example:

1. From `int (*(Foo::*a[5])(int, int))(int)` we have that the type of `(*(Foo::*a[5])(int, int))(int)` is `int`.
2. So the type of `(*(Foo::*a[5])(int, int))` is `int(int)`.
3. So the type of `(Foo::*a[5])(int, int)` is a pointer to `int(int)`, i.e., `int(*)(int)`.
4. So the type of `(Foo::*a[5])` is a function that has two `int` parameters and returns a `int(*)(int)`.
5. So the type of `a[5]` is a member pointer to that function.
6. So the type of `a` is an array of 5 that member pointers.

**In short, you can consider the *declarator* as an expression, and the type of this expression is the *type specifier*.** Using this rule, you can infer the type backward.

Of course, writing complex types in the way shown above is not recommended. That declaration is only for illustration. In real world code, it might be written as the following:

```cpp
struct Foo {
    auto bar(int, int) -> int(*)(int);
};

decltype(&Foo::bar) a[5];
```

Before C++11, you can use `typedef` to decompose that declaration to improve readability.

```cpp
struct Foo {
    typedef int (*bar_t)(int);
    bar_t bar(int, int);
};

typedef Foo::bar_t (Foo::*foo_bar_t)(int, int);
foo_bar_t a[5];
```

Such type aliases are very common in C.

But if the type comes from demangling, it will be the hideous one-line looking.

## Multi-Variable Declarations

After you can distinguish between *type specifiers* and *declarators*, it's easy to understand the rules of C++ multi-variable declarations. In multi-variable declarations, the comma-separated *declarators* (the expressions) share one *type specifier* (the return type). For example:

```cpp
int *a, b;
```

The type of `a` is `int*` while the type of `b` is `int`, since `*` is a part of the *declarator*. If you wish to define two pointers, you should write `int *a, *b` instead.

More complex examples follow the same rules. Since I can't think of any point in doing this, I won't give more examples.

## `const` And `volatile`

`const` and `volatile` take the same position in declarations, and they can be used together. Here for simplicity, only `const` will be used in the following examples. All `const` below can be (syntactically) legally replaced by `volatile`, `const volatile`, and `volatile const`. The later two are semantically equal.

`const` usually has a clear meaning, as in `const int a` and `int (*a)(const std::string&)`. Confusions of `const` often involve pointers. For pointers, `const` needs to express two meanings:

1. The pointer itself is `const`, i.e., it cannot point to other values, like `char *const a`.
2. The pointee it points to is `const`, i.e., you cannot modify the pointee through the pointer, like `const char* a`.

When there are nested pointers, we also have to consider the `const` property of those inner pointers, so the types become complicated. With the previous knowledge, we can divide `const`s into `const`s on *type specifiers* and `const`s on *declarators*. Look at the following type:

```cpp
const int * const * * const a;
```

The leftmost `const` is on the *type specifier*, and the rest `const`s are on the *declarators*.

`const`s on *type specifiers* are required to be placed aside *type specifiers*. Both `const T` and `T const` are legal and semantically equal. Thus, the above example can be written as:

```cpp
int const * const * * const a;
```

They both mean that

1. The type of `* const * * const a` is `const int` (cannot modify `***a`).

To distinguish that which `const` specifies which part, you only need to take care that when you encounter a `const` during the equation solving.

2. The type of `* * const a` is a const pointer to `const int` (cannot modify `**a`).
3. The type of `* const a` is a non-const pointer to the previous pointer (can modify `*a`).
4. The type of `a` is a const pointer to the previous pointer (cannot modify `a`).

The famous C++ book *C++ Primer* divides `const`s into *top-level consts* and *low-level consts*. The term *top-level const* refers to the `const` that specifies the pointer itself, i.e., the innermost one; the term *low-level const* refers to the rest.

## `&` And `&&`

References in C++ is very special. Since C++ standard specifies that references don't necessarily occupy memory space, you can't write arrays of references (`int& a[5]`) and pointers to references (`int&* a`) and many other reference related types. I think it's a huge design failure. It adds a lot of obstacles out of nowhere.

Anyway, except for function types (and types that contains function types) where `&` and `&&` can appear in the argument type and return value type, in other cases `&` and `&&` can only appear at the top level of the type. The meaning of "top-level" here is the same as *top-level const* in the previous section.

Directly nesting `&` and `&&`, like `int& && a`, is not allowed. But indirectly nesting them is allowed. For example:

```cpp
using ref1 = int&&;
using ref2 = ref1&&;
using ref3 = ref2&;
```

In this case, *reference collapsing* will occur. The rule of *reference collapsing* is simple: if there are all `&&`, then final type will be `&&`; if there is one `&`, then the final type will be `&`. So the `ref2` above is actually `int&&`, while the `ref3` is `int&`.

Another usage of `&&` is in templates. If `T` is a template parameter, then `T&&` (without `const` and `volatile`) is a *universal reference*. It has the following rules:

1. If the argument passed to `T&&` has type `Foo&`, then `T` is inferred to `Foo&`, then `T&&` becomes `Foo&` after *reference collapsing*.
2. If the argument passed to `T&&` has type `Foo&&`, then `T` is inferred to `Foo`, then `T&&` becomes `Foo&&` after *reference collapsing*.

Note that it's important that `T` is inferred to what type, because you might want to use `T` somewhere inside the template.

## Member Functions

In C++, sometimes we can see code like this:

```cpp
struct Foo {
    int a(int, int) const;
};

struct Bar {
    int a(int, int) &&;
};
```

This is easy to understand. Each non-static member function in C++ has an implicitly defined `this` pointer that points to the instance. The `const` and `&&` written after it are used to specify instances.

On some other languages like Rust and Python, the passed instance is written explicitly as the first parameter, whereas in C++ it is implicit (before C++23). So you have to add type specifiers elsewhere. The following examples loosely reveal how those type specifiers work.

```cpp
struct Foo {
    // void a(const Foo& this_, int) {
    //     const Foo* this = &this_;
    // }
    void a(int) const&;

    // void a(Foo&& this_, int) {
    //     Foo* this = &this_;
    // }
    void a(int) &&;

    // void a(Foo& this_, int) {
    //     Foo* this = &this_;
    // }
    void a(int) &;

    // foo.a(1) -> Foo::a(foo, 1)
};

struct Bar {
    // void a(const Bar& this_, int) {
    //     const Bar* this = &this_;
    // }
    //
    // or just:
    //
    // void a(const Bar* this, int);
    void a(int) const;

    // void a(Bar& this_, int) {
    //     Bar* this = &this_;
    // }
    //
    // or just:
    //
    // void a(Bar* this, int);
    void a(int);

    // bar.a(1) -> Bar::a(bar, 1)
    //
    // or:
    //
    // bar.a(1) -> Bar::a(&bar, 1)
};
```

Member functions with references cannot be overloaded with member functions without references. You either choose the set demonstrated by `Foo` or the set demonstrated by `Bar`. Some other possible overloads, such as those with `volatile`, are omitted.

## `typedef` And `using`

I have shown examples of `typedef` and `using` in previous sections. The syntax of `typedef` is exactly the same as defining a variable. The only one difference in syntax between `typedef` and `using` is that `using` put the type name in the left. In the following example, both `foo_t` and `bar_t` have the same type of `func_ptr`.

```cpp
int (*func_ptr)(int);
typedef int (*foo_t)(int);
using bar_t = int(*)(int);
```

Compared to `typedef`, one advantage of `using` is that it can use templates. E.g.,

```cpp
template<typename T>
using foo_t = std::vector<T>;

foo_t<int> foo;
```

Before C++11, you can implement this through wrapping `typedef` by a `struct` or a `class`. E.g.,

```cpp
template<typename T>
struct Bar {
    typedef std::vector<T> type;
};

Bar<int>::type bar;
```

This post only discusses types, so other semantics of `using` will be skipped.

## More

The components of the C++ declaration statement are quite complex, not just the *type specifiers* and *declarators*. But the rest of them are much more human-friendly, and many of them are not part of types. For those who interested in learning more, please read [cppreference](https://en.cppreference.com/w/cpp/language/declarations).

Types could have been clear and easy to read, it's just that C and C++'s design is terrible. Now that you've read this article, congratulations on solving a problem that doesn't even exist in other languages.

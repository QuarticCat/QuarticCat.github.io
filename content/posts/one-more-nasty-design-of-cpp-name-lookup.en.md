---
title: "One More Nasty Design of C++'s Name Lookup"
date: 2021-06-23
tags: [cpp]
showToc: false
---

As we all know that C++'s name lookup has always been extremely counter-intuitive. The infamous [argument-dependent lookup (ADL)](https://en.cppreference.com/w/cpp/language/adl), for example, often leads to unexpected behavior and, worse yet, is often difficult to troubleshoot. This can happen when you define a function in your current namespace, but when you call it the compiler selects another function with the same name thousands of miles away, even though you did not `using` that foreign function in the current namespace. Here are two typical examples.

```cpp
namespace fuckadl {

struct Fuck {};

void foo(Fuck) { puts("not mine"); }

}  // namespace fuckadl

// case 1
void foo(fuckadl::Fuck) { puts("mine"); }

// // case 2
// template<class T>
// void foo(T) { puts("mine"); }

int main() {
    foo(fuckadl::Fuck{});
}
```

In case 1, you will get a compilation error, which complains about ambiguity. In case 2, which is worse, you won't even receive a warning. The compiler will simply select `fuckadl::foo` for you. If you are a beginner of C++, it is likely to take you a whole day to debug this problem.

This design directly breaks the encapsulation of namespaces, while countless header-only libraries are using namespaces to hide internal symbols from caller sites (after all, C++'s module mechanism has been delayed for so long), and you don't know when you'll collide with someone else's function names.

I recently discovered another nasty design of C++'s name lookup. Here it is, C++ Standard Draft N3337 10.2 Member name lookup \[class.member.lookup\]:

> 1. Member name lookup determines the meaning of a name (id-expression) in a class scope (3.3.7). Name lookup can result in an ambiguity, in which case the program is ill-formed. For an id-expression, name lookup begins in the class scope of this; for a qualified-id, name lookup begins in the scope of the nested-name-specifier. **Name lookup takes place before access control** (3.4, Clause 11).

The last sentence is the main point. It means that access specifiers like `public` and `private` are invisible when performing the name lookup. I'll give a very counter-intuitive case (modified from [this question](https://stackoverflow.com/questions/21636150/typecast-operator-in-private-base)):

```cpp
struct Base {
    operator bool() {
        return true;
    }
};

struct Derived: private Base {
    operator int() {
        return 1;
    }
};

int main() {
    std::cout << Derived{};
}
```

Note the use of `private` inheritance here. To put it informally, the `Derived` class actually looks like this from the callers' perspective:

```cpp
struct Derived {
  public:
    operator int() {
        return 1;
    }

  private:
    operator bool() {
        return true;
    }
};
```

However, the compiler will emit an ambiguity error for this code! Because the compiler doesn't see access specifiers at this point, both `operator bool()` and `operator int()` are valid candidates, and that's why we have ambiguity.

Since C++ does not have zero-sized types, before the advent of `[[no_unique_address]]` in C++20, one common way to introduce zero-sized members was to use `private` / `protected` inheritance, utilizing the [empty base optimization](https://en.cppreference.com/w/cpp/language/ebo) to achieve zero size. This is not rare in template libraries. For example, Boost's [`compressed_pair`](https://theboostcpplibraries.com/boost.compressed_pair) is implemented in this way.

Thanks to this bizarre design, we have to consider whether the names of all members of the base class will conflict with the derived class when using this technique, even if the inheritance is `private`. This is so outrageous that it could definitely be considered an abstraction leak.

Wait, you say you don't use `private` inheritance at all? Well, it can still haunt you. Here's a simpler example:

```cpp
struct Base1 {
  public:
    int x = 1;
};

struct Base2 {
  private:
    int x = 1;
};

struct Derived: Base1, Base2 {};

int main() {
    Derived{}.x;
}
```

The compiler will emit an ambiguity error for this code as well for the same reason. When looking for `.x`, the compiler doesn't see access specifiers. It finds that both `Base1` and `Base2` have an `x` member, which is ambiguous.

This example is a bit more outrageous than the last one. Imagine that, if `Base2` come from outside, you won't even see such a member name in its API document. And most likely the name of this member name is not guaranteed. After all, it's implementation details. Yet this member will get in the way of your name lookup! Similar to ADL, this design makes `private` encapsulate nothing.

Of course, also similar to ADL, there are approaches to get around this. Below is one possible solution. But the existence of this problem is really annoying.

```cpp
struct Derived: Base1, Base2 {
    using Base1::x;
};
```

Next is the most outrageous thing: ambiguity errors can trigger substitution failures, so they can be used in SFINAE or Concept. That means, by utilizing the code logic above, we can detect externally whether a class has some `private` / `protected` members.

Let me demonstrate it. Suppose we are going to detect member `x`, no matter whether it is `public` or not, then we can construct a class `A` with a member `x` and a class `B` that inherits both `A` and the class to be detected, and then produce a substitution failure by accessing the `x` member of `B`. The code is as follows:

```cpp
struct A {
    int x;
};

template<class T>
struct B: A, T {};

template<class T>
concept TestX = !requires(B<T> b) {
    b.x;
};
```

Lastly, we write two simple test cases.

```cpp
struct HasX {
  private:
    int x;
};

struct HasY {
  private:
    int y;
};

static_assert(TestX<HasX>);  // success
static_assert(TestX<HasY>);  // error
```

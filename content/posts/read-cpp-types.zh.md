---
title: "C++ 类型阅读入门"
date: 2021-02-13
tags: [cpp]
---

C++ 的类型可读性很差，并且大多数入门材料中并没有详细介绍如何阅读它们，最多只是讲到 top-level const 和 low-level const 的区别。有不少朋友问过我这方面的问题，讲得多了，干脆整理起来写篇文章。

## 常见误解

在详细讲类型的阅读之前，需要纠正一个常见的误解。

问：`int a[5]` 里的 `a` 是什么类型的？

答：`int[5]` 类型，在适当的时候会「退化」（decay）成为 `int*` 类型。

问：`int a[5][6]` 里的 `a` 是什么类型的？

答：`int[5][6]` 类型，在适当的时候会「退化」成为 `int(*)[6]` 类型，即指向 `int[6]` 的指针。

由于「退化」这种隐式转换的存在，很多初学 C++ 的人会把数组类型等同于指针类型。

类似的，函数类型也会「退化」到指针类型，如 `int(int, int)` 会「退化」成为 `int(*)(int, int)`。

## 解方程

当你查阅如何阅读一些复杂的类型时，你可能会看到网络上一些人说 C++ 的类型就是解方程，让我来详细解释一下这句话。

抛去 CVR (const, volatile, reference) 等不谈，C++ 最基本的声明分为两个部分：写在最左边的**类型名**是「类型说明符」（type specifier），剩下的部分是「声明符」（declarator）。这部分内容是从 C 继承过来的，它们是 C++ 类型里最恶心的地方。但这两个名字实在没啥识别度，我喜欢不严谨地称呼为返回值类型和调用表达式，这点后面会解释。先来看几个例子吧：

|声明|类型说明符|声明符|
|:-:|:-:|:-:|
|`int a`|`int`|`a`|
|`int* a`|`int`|`*a`|
|`int a[5]`|`int`|`a[5]`|
|`int* a[5]`|`int`|`*a[5]`|
|`int (*a)[5]`|`int`|`(*a)[5]`|

作出这种分别后，就可以理解上表中`a`的类型是怎样决定的了：**以「声明符」的形式调用`a`后，得到的返回值类型为「类型说明符」**。所谓的解方程就是这样一个过程：

1. 由 `int *a[5]` 得到 `*a[5]` 的类型是 `int`。
2. 即对 `a[5]` 解引用得到 `int` ，推出 `a[5]` 的类型是 `int*`。
3. 即对 `a` 取下标得到 `int*` ，推出 `a` 的类型是 5 个 `int*` 的数组。

再比如：

1. 由 `int (*a)[5]` 得到 `(*a)[5]` 的类型是 `int`。
2. 即对 `(*a)` 取下标得到 `int`，推出 `(*a)` 的类型是 `int[5]`。
3. 即对 `a` 解引用得到 `int[5]`，推出 `a` 的类型是指向 `int[5]` 的指针。

可以看到这里运算符优先级和调用时完全一致，形式上也基本一致。

再来个函数指针的例子：

1. 由 `int (*a)(int, int)` 得到 `(*a)(int, int)` 的类型是 `int`。
2. 即对 `(*a)` 以 `(int, int)` 的形式调用得到 `int`，推出 `(*a)` 的类型是有两个 `int` 参数并且返回 `int` 的函数，即 `int(int, int)`。
3. 即对 `a` 解引用得到 `int(int, int)`，推出 `a` 的类型是指向 `int(int, int)` 的指针。

更复杂一点：

1. 由 `int (*(Foo::*a[5])(int, int))(int)` 得到 `(*(Foo::*a[5])(int, int))(int)` 的类型是 `int`。
2. 推出 `(*(Foo::*a[5])(int, int))` 的类型是 `int(int)`。
3. 推出 `(Foo::*a[5])(int, int)` 的类型是指向 `int(int)` 的指针，即 `int(*)(int)`。
4. 推出 `(Foo::*a[5])` 的类型是有两个 `int` 参数并且返回 `int(*)(int)` 的函数。
5. 推出 `a[5]` 的类型是指向这种函数的成员指针。
6. 推出 `a` 的类型是 5 个这种成员指针的数组。

**简而言之，你可以把「声明符」部分看成一个调用表达式，这个调用表达式的返回值类型为「类型说明符」，这样就可以一步步倒推出类型来。**

当然了，这么复杂的类型是不推荐这么写的，这里只是为了演示一下解方程的过程。上面那个`a`正常一点的声明大概会长这样：

```cpp
struct Foo {
    auto bar(int, int) -> int(*)(int);
};

decltype(&Foo::bar) a[5];
```

即使在 C++11 以前，也可以利用 `typedef` 拆解成多层，提升可读性：

```cpp
struct Foo {
    typedef int (*bar_t)(int);
    bar_t bar(int, int);
};

typedef Foo::bar_t (Foo::*foo_bar_t)(int, int);
foo_bar_t a[5];
```

这种包装在 C 语言中还是挺常见的。

不过如果你要看的类型是 demangle 出来的（或者其他查看类型名的方式），那就是长成那种魔鬼样子了。

## 多变量声明

有了前面的知识，就能够理解 C++ 多变量声明的规则了。多变量声明中，逗号分割的是「声明符」（即调用表达式部分），它们共享「类型说明符」（即返回值部分），比如：

```cpp
int *a, b;
```

其中 `a` 的类型为 `int*` 而 `b` 的类型则为 `int`，因为 `*` 是「声明符」的一部分。如果你希望定义两个指针，那么应该写成 `int *a, *b`。

更复杂的例子也遵循同样的规则。因为实在想不到更复杂的情况这样写有什么意义，就不再举例子了。

## `const` 与 `volatile`

`const` 与 `volatile` 在类型中的位置是一样的，并且他们两个可以一起使用，因此这里仅仅用 `const` 作为例子。下面出现的所有 `const` 都可以合法地替换成 `volatile`、`const volatile` 和 `volatile const`，后两者语义上等价。

`const` 在一般情况下都很明确，如 `const int a` 和 `int (*a)(const std::string&)`。`const` 的理解障碍主要出在指针的情况中。对于指针来说，`const` 需要能表达两种含义：

1. 指针本身是 `const` 的，即这个指针没法再指向别的值，如 `char *const a`。
2. 指针所指向的对象是 `const` 的，即你没法通过这个指针修改它指向的对象，如 `const char* a`。

当指针嵌套时，我们还要考虑指针指向的指针的 `const` 属性，于是类型就变得复杂起来。有了前面的知识，我们可以把 `const` 分为修饰「类型说明符」的 `const` 和修饰「声明符」的 `const`。看下面这个类型：

```cpp
const int * const * * const a;
```

从左往右的第一个 `const` 为修饰「类型说明符」的 `const`，其余的为修饰「声明符」的 `const`。

修饰「类型说明符」的 `const` 只要写在「类型说明符」旁边，不论 `const T` 还是 `T const` 都是合法且等价的。也就是上面这个例子还可以写成：

```cpp
int const * const * * const a;
```

它们都说明：

1. `* const * * const a`（当然去掉 `const` 后它才是一个合法的表达式，下同）的类型为 `const int`（不能修改 `***a`）

修饰「声明符」的 `const` 只要看解方程时解到哪个表达式，就说明修饰的是哪个表达式的类型。接着上面这个例子：

2. 推出 `* * const a` 是一个指向 `const int` 的 const 指针（不能修改 `**a`）
3. 推出 `* const a` 是一个指向前面那个指针的非 const 指针（可以修改 `*a`）
4. 推出 `a` 是一个指向前面那个指针的 const 指针（不能修改 `a`）

著名的 C++ 入门书籍《C++ Primer》把 `const` 分为了「顶层 const 」（top-level const）和「底层 const 」（low-level const）（此为原版翻译，显然并不准确）。顶层指的是修饰指针自身的，也就是类型中最内层的；底层指的是其余的。这两个术语在只讨论一层指针的时候才比较好用，当然这也是绝大多数情况。

## `&` 与 `&&`

C++ 的引用是一种非常特殊的类型。由于 C++ 规定引用不一定占用内存，所以你既没法写出引用类型的数组（`int& a[5]`），也没法写出指向引用类型的指针（`int&* a`），除此之外还有很多地方无法使用引用。我认为这是一个非常失败的设计，凭空增加了很多障碍。它明明只是一个「不可空指针」（non-nullable pointer）而已。右值引用也并没有什么特别，只是带有所有权转移语义的引用而已。

总之，除了函数类型（以及嵌套了函数类型的类型）里 `&` 和 `&&` 会出现在参数类型和返回值类型中，其他情况下 `&` 和 `&&` 只会出现在类型的最顶层。这里的顶层的含义与前文的「顶层 const 」相同。

`&` 和 `&&` 没法直接嵌套使用，比如 `int& && a` 是非法的。但是间接嵌套是合法的，比如：

```cpp
using ref1 = int&&;
using ref2 = ref1&&;
using ref3 = ref2&;
```

在这种情况下会发生「引用折叠」（reference collapsing）。规则也很简单，如果全是 `&&`，那么会折叠成一个 `&&`。只要有一个 `&`，就会折叠成 `&`。比如上面的 `ref2` 实际上是 `int&&`，而 `ref3` 则是 `int&`。

`&&` 还有一种用法是在模板中。如果 `T` 是一个模板参数，那么 `T&&`（不能带有 `const` 和 `volatile`）就是一个「万能引用」（universal reference）。具体规则为：

1. 如果传入 `T&&` 的类型为 `Foo&`，那么 `T` 就被推导为 `Foo&`，而 `T&&` 经过折叠变成 `Foo&`。
2. 如果传入 `T&&` 的类型为 `Foo&&`，那么 `T` 就被推导为 `Foo`，而 `T&&` 则是 `Foo&&`。

请注意，`T` 被推导为什么类型这一点很重要，因为你可能在模板内部再次使用 `T`。

## 成员函数

在 C++ 的成员函数中，我们有时会见到这样的例子：

```cpp
struct Foo {
    int a(int, int) const;
};

struct Bar {
    int a(int, int) &&;
};
```

这个其实也很好理解。C++ 的非 static 成员函数都可以调用 `this`，一个指向实例的指针。写在后面的 `const` 和 `&&` 就是用来修饰实例的。

在其他一些语言（比如 Rust 和 Python ）中传入的实例是显式地作为第一个参数写出的，而 C++ 是隐式的（在 C++23 前），于是只能在其他地方加上类型修饰。不过即使不是隐式的，你也没法在 C++ 中直接写出 `this` 的类型，但这仍然是 C++ 的锅——因为 `this` 是一个指针，而 C++ 中指针没法指向引用类型。下面两个例子用注释不严谨地揭示了后置的类型修饰是怎么运作的：

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

带有引用的成员函数无法和不带有引用的成员函数重载在一起，也就是你要么选择 `Foo` 演示的这一组，要么选择 `Bar` 演示的这一组。一些其他可能的重载，如带有 `volatile` 的，就不再写出了。

从这个例子你也可以再一次看到 C++ 的引用设计得多么失败。

## `typedef` 与 `using`

前文已经有 `typedef` 和 `using` 的例子了。`typedef` 的语法其实很简单，和定义变量是完全一致的。而 `using` 语法的区别仅仅在于**把类型名的部分提到了左边**。下面这个例子中，`foo_t` 和 `bar_t` 都和 `func_ptr` 的类型一致。

```cpp
int (*func_ptr)(int);
typedef int (*foo_t)(int);
using bar_t = int(*)(int);
```

`using` 相比 `typedef` 的一个优势是可以直接定义「别名模板」（alias template），如：

```cpp
template<typename T>
using foo_t = std::vector<T>;

foo_t<int> foo;
```

在 C++11 以前，这种需求可以通过套一层 `struct` 或 `class` 实现，如：

```cpp
template<typename T>
struct Bar {
    typedef std::vector<T> type;
};

Bar<int>::type bar;
```

本文仅讨论类型，所以 `using` 的其他语义就略过了。

## 更多

C++ 的声明语句构成部分相当复杂，并非只有「类型说明符」和「声明符」，但其余部分相比上文提到的那些已经相当人类友好了，没有多少学习障碍，而且其中的很多与类型无关。有兴趣了解的可以参阅 [cppreference](https://zh.cppreference.com/w/cpp/language/declarations)。

类型本可以明确易读，只是 C 和 C++ 设计得很失败。既然读到这里了，那么恭喜你解决了一个其他语言不存在的问题。

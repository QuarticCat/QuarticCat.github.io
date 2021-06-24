---
title: "C++ 名称查找的又一个恶心设计"
date: 2021-06-23
tags: [cpp]
---

众所周知，C++ 的名称查找一直以来都很反直觉。比如这个 [ADL](https://en.cppreference.com/w/cpp/language/adl) ，其恶心程度在 C++ 的各种 feature 里绝对排得上号。

这玩意经常在意想不到的地方恶心到你，还往往难以排查。具体表现为你在当前的命名空间里自己定义了一个函数，结果在调用它的时候编译器却找到了十万八千里外的另一个同名函数，而你明明没有在当前命名空间引入该函数。这种情况甚至不会有一个提示。假如是不知道这个 feature 的 C++ 新人，怕是 debug 一天也找不到哪里出了问题。

这种行为直接破坏了命名空间的封装意义，要知道无数的 header only 库都在用命名空间来对外隐藏内部符号（谁让 C++ 的模块机制拖延了这么久呢），你都没法知道什么时候就和别人函数名字撞上了。

最近发现了 C++ 名称查找的又一个恶心设计。来请出我们的主角，C++ Standard Draft N3337 10.2 Member name lookup \[class.member.lookup\]:

> 1. Member name lookup determines the meaning of a name (id-expression) in a class scope (3.3.7). Name lookup can result in anambiguity, in which case the program is ill-formed. For an id-expression, name lookup begins in the class scope of this; for a qualified-id, name lookup begins in the scope of the nested-name-specifier. **Name lookup takes place before access control** (3.4, Clause 11).

最后一句是重点，它意味着在进行 name lookup 的时候看不到`public`、`private`之类的 access specifier 。下面我给出一个非常违反直觉的 case （修改自[该问题](https://stackoverflow.com/questions/21636150/typecast-operator-in-private-base)）：

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

请注意这里使用了`private`继承。不严谨地解释的话，`Derived`类对外实际相当于这样：

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

然而这串代码编译器会报二义性错误，没错！因为编译器这时还看不到 access specifier ，因此`operator bool()`和`operator int()`都会当作候选，于是就产生二义性了。

由于 C++ 没有 Zero-Sized Types ，在 C++20 出现`[[no_unique_address]]`之前，我们想要引入零大小成员的一种方式就是使用`private`/`protected`继承，利用空基类优化来实现 zero size 。这在模板库中并不是一种极度罕见的技巧，Boost 的[`compressed_pair`](https://theboostcpplibraries.com/boost.compressed_pair)就是以这种方式实现的。

然而由于 C++ 名称查找的这一恶心特性，我们在使用这种技巧时将不得不考虑基类的所有成员是什么，是否会与派生类产生意外的结果，哪怕我们使用的是`private`继承。这实在是太离谱了，绝对可以算得上是一种抽象泄露。

好吧，如果你说你不会用到`private`继承，那么下面我给出一个更容易见到的例子：

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

这串代码编译器也会报二义性错误，同样的，因为编译器在查找`.x`的时候还看不到 access specifier ，它发现`Base1`和`Base2`都有一个`x`成员，就产生了二义性。

这个例子更离谱一些。想象一下，假如其中`Base2`来自外部，那么你在它的接口文档中根本不会看到它内部有这么一个成员，而极大概率这个成员的名字也是不保证的，毕竟属于它的内部实现。然而这个成员却会妨碍你的名称查找！跟 ADL 的问题一样，这封装已经漏成筛子了，封装了个寂寞。

当然和 ADL 的问题一样，并不是没有办法绕过，只需要稍微改下`Derived`类即可，只是这种问题的存在实在让人恶心。

```cpp
struct Derived: Base1, Base2 {
    using Base1::x;
};
```

接下来是最离谱的事情：二义性错误也可以触发 substitution failure ，因此可以被用于 SFINAE 或者 Concept 。那么结合上文，我们可以在外部探测到一个类是否拥有某个`private`/`protected`成员。

假如我们要探测一个类是否有名为`x`的成员，不论`public`与否，那么我们可以构造一个具有相同成员的类 A ，再构造一个类 B 同时继承 A 和待检测的类，然后通过访问`B{}.x`制造 substitution failure 。代码如下：

```cpp
struct A {
    int x;
};

template<class T>
struct B: A, T {};

template<class T>
concept TestX = !requires {
    std::declval<B<T>>().x;
};
```

这里使用`std::declval`的原因是我们无法保证`B<T>`可以被默认构造。最后我们写两个简单的测试样例：

```cpp
struct HasX {
  private:
    int x;
};

struct HasY {
  private:
    int y;
};

static_assert(TestX<HasX>);
static_assert(TestX<HasY>);
```

可以看到只有`TestX<HasY>`为 false 。

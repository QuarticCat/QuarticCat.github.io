---
title: "C++ Tricks：更好的访问者模式"
date: 2021-03-10
tags: [cpp, design-pattern]
---

## 引言

C++ 作为一门没有直接在语言层面支持 tagged union 的 OOP 语言，在进行诸如操作 AST 一类的处理时常常会采用访问者模式。我大一时写的一个[弱智解释器](https://github.com/QuarticCat/yacis)中也是如此。很可惜当时在编程水平和 deadline 的双重限制下没能好好研究，时隔近一年，我对访问者模式也有了更多的理解，打算讲讲这个设计模式的问题和 C++ 中的对应解决办法。

## 起步

相信所有人初学访问者模式的时候见到的都是类似下面这样的代码：

```cpp
struct FooAcc;
struct BarAcc;

struct Visitor {
    void visit(FooAcc&);
    void visit(BarAcc&);
};

struct AccBase {
    virtual void accept(Visitor& visitor) = 0;
};

struct FooAcc: AccBase {
    void accept(Visitor& visitor) override {
        visitor.visit(*this);
    };
};

struct BarAcc: AccBase {
    void accept(Visitor& visitor) override {
        visitor.visit(*this);
    };
};

void Visitor::visit(FooAcc&) {
    // do something
}

void Visitor::visit(BarAcc&) {
    // do something
}
```

通过调用`accepter.accept(visitor)`的方式，我们实现了双派发：第一层是多态对象`accepter`的虚派发，第二层是`Visitor::visit`方法的重载。有了这个模式，我们能用以统一的方式调用一个处理多态对象的方法，而在这个方法内部又可以分别对这个多态对象的不同类型进行分别处理。这非常适合用于 AST 的处理。我们会把不同的 AST 节点都写成某个基类的派生类以方便存储，但是在处理 AST 时又希望针对不同类型的节点分别处理。于是我们可以把这里的 Accepter 换成 Node ，Visitor 换成某个处理 AST 的方法。

## 多 Visitor 写法

现在问题来了，对于一个编译器/解释器，我们通常会分成好几个阶段去处理一颗 AST ，比如一个阶段做类型检查，一个阶段做代码生成，这时候我们就有不止一个 Visitor 了。那么上面的代码要怎么改呢？最简单的方法莫过于给每个 Accepter 再增加一套`accept`方法：

```cpp
struct FooAcc;
struct BarAcc;

struct BazVis {
    void visit(FooAcc&);
    void visit(BarAcc&);
};

struct QuxVis {
    void visit(FooAcc&);
    void visit(BarAcc&);
};

struct AccBase {
    virtual void accept(BazVis& visitor) = 0;
    virtual void accept(QuxVis& visitor) = 0;
};

struct FooAcc: AccBase {
    void accept(BazVis& visitor) override {
        visitor.visit(*this);
    };
    void accept(QuxVis& visitor) override {
        visitor.visit(*this);
    };
};

struct BarAcc: AccBase {
    void accept(BazVis& visitor) override {
        visitor.visit(*this);
    };
    void accept(QuxVis& visitor) override {
        visitor.visit(*this);
    };
};
```

这当然是可以的，但当 Accepter 的数量较多时，这种修改将会是个灾难。我们可以很直观地想到，给所有的 Visitor 设置一个基类不就可以省去这项工作了么，新增的 Visitor 只需要继承这个基类就可以了：

```cpp
struct FooAcc;
struct BarAcc;

struct VisBase {
    virtual void visit(FooAcc&) = 0;
    virtual void visit(BarAcc&) = 0;
};

struct BazVis: VisBase {
    void visit(FooAcc&) override;
    void visit(BarAcc&) override;
};

struct QuxVis: VisBase {
    void visit(FooAcc&) override;
    void visit(BarAcc&) override;
};

struct AccBase {
    virtual void accept(VisBase& visitor) = 0;
};

struct FooAcc: AccBase {
    void accept(VisBase& visitor) override {
        visitor.visit(*this);
    };
};

struct BarAcc: AccBase {
    void accept(VisBase& visitor) override {
        visitor.visit(*this);
    };
};
```

这当然也是可以的，但是这导致我们增加了一层 Visitor 的虚派发，然而绝大部分场景下我们很清楚我们需要调用的是哪个 Visitor ，这层虚派发完全是多余的。此外，当不同 Visitor 的`visit`成员函数需要返回不同的类型的时候，这个方法就很难办了，几乎一定需要再次引入额外的开销。

## 零开销多 Visitor 写法

注意到，所有的`accept`方法几乎是一样的，我们之所以要在每个 Accepter 里写一遍是因为`*this`的类型不同，直接统一写在基类中没有办法选择正确的重载函数。既然和`*this`的类型有关，那么我们可以立马想到 CRTP ：

```cpp
struct FooAcc;
struct BarAcc;

struct BazVis {
    void visit(FooAcc&);
    void visit(BarAcc&);
};

struct QuxVis {
    int visit(FooAcc&);
    int visit(BarAcc&);
};

struct AccBase {
    virtual void accept(BazVis& visitor) = 0;
    virtual int accept(QuxVis& visitor) = 0;
};

template<class Derived>
struct AccCRTP: AccBase {
    void accept(BazVis& visitor) override {
        visitor.visit(*static_cast<Derived*>(this));
    };
    int accept(QuxVis& visitor) override {
        visitor.visit(*static_cast<Derived*>(this));
    };
};

struct FooAcc: AccCRTP<FooAcc> {};

struct BarAcc: AccCRTP<BarAcc> {};
```

我们将直接继承`AccBase`改为通过继承`AccCRTP<Self>`来间接继承`AccBase`，`AccCRTP<Self>`为我们提供了所有`accept`方法的实现，所有的子 Accepter 甚至一遍`accept`函数都不用写。同时，空基类优化会将 CRTP 类占用的空间去掉，这里没有额外的开销。当我们增加一个 Visitor 时，我们实际上只需要修改`AccBase`和`AccCRTP`这两个地方，这是一步巨大的跨越。要分别为不同的 Visitor 指定不同的返回值也很简单，就如同上面示例的那样。

## 只改一处的零开销多 Visitor 写法

即便已经把工作量减少到了这种程度，我们依然可以看见很多重复代码，Visitor 一旦多起来就会让人看着难受，想找办法把重复代码去掉。这是可行的，我们将开始把玩模板元编程魔法。首先尝试自动实现`AccBase::visit`，我们这里使用一个`TypeList`来存所有的 Visitor 。

```cpp
struct FooAcc;
struct BarAcc;

struct BazVis {
    using return_type = void;
    return_type visit(FooAcc&);
    return_type visit(BarAcc&);
};

struct QuxVis {
    using return_type = int;
    return_type visit(FooAcc&);
    return_type visit(BarAcc&);
};

template<class... Ts>
struct TypeList {};

using VisitorList = TypeList<BazVis, QuxVis>;

template<class T>
struct AccBaseHelper {
    virtual typename T::return_type accept(T& visitor) = 0;
};

template<class>
struct AccBaseImpl;

template<class... Ts>
struct AccBaseImpl<TypeList<Ts...>>: AccBaseHelper<Ts>... {};

using AccBase = AccBaseImpl<VisitorList>;
```

这样我们就只需要修改`VisitorList`，`AccBase`里的内容会自动实现。但是为什么下半部分不直接这样写呢：

```cpp
template<class = VisitorList>  // note here
struct AccBase;

template<class... Ts>
struct AccBase<TypeList<Ts...>>: AccBaseHelper<Ts>... {};

// remove using
```

这是因为这样写的话在很多地方必须得写成`AccBase<>`，看起来比较蠢。

这个方法乍看之下很不错，但其实我们引入了额外开销，只需测一下`sizeof(AccBase)`就能发现它会随着 Visitor 的增加而增加。问题在于我们用了多继承和虚函数，每多一个 Visitor 就会增加一个虚表指针。所以这里我们用递归继承做一个简单的单继承化，把它们合成一张表（因此只需要一个虚表指针）：

```cpp
// remove AccBaseHelper

template<class>
struct AccBaseImpl {};

template<class T, class... Ts>
struct AccBaseImpl<TypeList<T, Ts...>>: AccBaseImpl<TypeList<Ts...>> {
    virtual typename T::return_type accept(T& visitor) = 0;
};
```

由于 C++ 的 name hiding 的机制，我们必须在派生类里显式`using`基类的方法，否则基类的方法会被派生类的同名方法覆盖，不会被查找到。但`AccBaseImpl`并不需要这样做，因为它的方法都是纯虚的，永远也不会被调用。尽管如此，Clang 还是会对此进行抱怨，如果你希望消除这一警告，可以改成下面这样子：

```cpp
template<class>
struct AccBaseImpl;

template<class T>
struct AccBaseImpl<TypeList<T>> {
    virtual typename T::return_type accept(T& visitor) = 0;
};

template<class T, class... Ts>
struct AccBaseImpl<TypeList<T, Ts...>>: AccBaseImpl<TypeList<Ts...>> {
    using AccBaseImpl<TypeList<Ts...>>::accept;
    virtual typename T::return_type accept(T& visitor) = 0;
};
```

同样的手段我们可以应用于 CRTP 类。在这里`using`基类方法就是必须的了，不过因为`AccBase`已经有`accept`方法了，所以可以简化情况：

```cpp
template<class Derived, class = VisitorList>
struct AccCRTP: AccBase {};

template<class Derived, class T, class... Ts>
struct AccCRTP<Derived, TypeList<T, Ts...>>: AccCRTP<Derived, TypeList<Ts...>> {
    using AccCRTP<Derived, TypeList<Ts...>>::accept;
    typename T::return_type accept(T& visitor) override {
        visitor.visit(*static_cast<Derived*>(this));
    };
};
```

`AccCRTP`总是要写模板参数的，所以没有必要用`using`再包一层。完整代码如下：

```cpp
struct FooAcc;
struct BarAcc;

struct BazVis {
    using return_type = void;
    return_type visit(FooAcc&);
    return_type visit(BarAcc&);
};

struct QuxVis {
    using return_type = int;
    return_type visit(FooAcc&);
    return_type visit(BarAcc&);
};

template<class... Ts>
struct TypeList {};

using VisitorList = TypeList<BazVis, QuxVis>;

template<class>
struct AccBaseImpl;

template<class T>
struct AccBaseImpl<TypeList<T>> {
    virtual typename T::return_type accept(T& visitor) = 0;
};

template<class T, class... Ts>
struct AccBaseImpl<TypeList<T, Ts...>>: AccBaseImpl<TypeList<Ts...>> {
    using AccBaseImpl<TypeList<Ts...>>::accept;
    virtual typename T::return_type accept(T& visitor) = 0;
};

using AccBase = AccBaseImpl<VisitorList>;

template<class Derived, class = VisitorList>
struct AccCRTP: AccBase {};

template<class Derived, class T, class... Ts>
struct AccCRTP<Derived, TypeList<T, Ts...>>: AccCRTP<Derived, TypeList<Ts...>> {
    using AccCRTP<Derived, TypeList<Ts...>>::accept;
    typename T::return_type accept(T& visitor) override {
        visitor.visit(*static_cast<Derived*>(this));
    };
};

struct FooAcc: AccCRTP<FooAcc> {};

struct BarAcc: AccCRTP<BarAcc> {};
```

其实这种方式并非完全零开销，尽管基类子对象被消除了，但每一层继承都会在虚表上留下条目，只不过这一部分开销相对来说微不足道。

## 一处都不用改甚至省去一堆前向声明而且还缓存友好的零开销多 Visitor 写法

这个东西叫做 tagged union 。

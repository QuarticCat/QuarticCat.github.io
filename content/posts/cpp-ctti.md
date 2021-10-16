---
title: "C++：编译期类型信息"
date: 2021-02-14
tags: [cpp]
---

## 引言

尽管标准库中已经有了[`typeid`](https://zh.cppreference.com/w/cpp/language/typeid)运算符，但是由于其需要支持检查多态类型，带来了非常多的限制：

1. 它必须启用 RTTI（Run-Time Type Information）。而很多项目是禁用 RTTI 的，所以无法使用`typeid`。
2. 它可能对表达式进行求值，详见 [cppreference](https://zh.cppreference.com/w/cpp/language/typeid) 。这可能带来意外的运行时开销甚至副作用，尤其是常用的`sizeof`和`decltype`都是完全静态的，不熟悉`typeid`的程序员可能完全意识不到这种动态行为的产生。
3. 它不是`constexpr`的，即使其类型本可以静态求出。这意味着很多场景都无法使用 `typeid`，比如模板参数、switch-case 语句的 case 值等。

这就是我们为什么需要编译期类型信息，即 CTTI（Compile-Time Type Information）。

## 实现原理

C++ 标准中并没有提供相关的设施供我们实现这一功能。但通过 GCC 和 Clang 的一个**特殊的预定义变量**`__PRETTY_FUNCTION__`，CTTI 得以实现。这个变量的值是当前函数完整签名的字符串。当在一个模板函数内部调用的时候，也会包含模板参数的类型名，这就达到了我们获取类型名字的目的。

注意，类似于标准中提供的[`__func__`](https://zh.cppreference.com/w/cpp/language/function#func)，`__PRETTY_FUNCTION__`是一个变量，因此它没法被用来初始化字符数组或者跟字符串字面量拼接在一起。MSVC 没有`__PRETTY_FUNCTION__`，但是有一个类似功能的宏`__FUNCSIG__`，它被替换为一个字符串字面量。

```cpp
template<typename T>
constexpr std::string_view pretty_function() {
    return __PRETTY_FUNCTION__;
}
```

在 Clang 上，调用`pretty_function<int>()`会返回`std::string_view pretty_function() [T = int]`，可以看到`T`的类型正在其中。

请注意，这里的`typename T`不能直接省略成`typename`，否则将不会出现`T`的类型名。

当我们确定了函数名之后，返回的字符串的格式就确定了。我们可以去掉无用的前后缀，从而提取出我们实际需要的类型名。

```cpp
constexpr const char PREFIX[] = "std::string_view pretty_function() [T = ";
constexpr const char SUFFIX[] = "]";

template<typename T>
constexpr std::string_view type_name() {
    auto name = pretty_function<T>();
    name.remove_prefix(sizeof(PREFIX) - 1);
    name.remove_suffix(sizeof(SUFFIX) - 1);
    return name;
}
```

这样我们就实现了[`std::type_info::name`](https://zh.cppreference.com/w/cpp/types/type_info/name)的功能。再实现一个 constexpr 的 hash 函数，我们就实现了[`std::type_index::hash_code`](https://zh.cppreference.com/w/cpp/types/type_index/hash_code)的功能。

```cpp
constexpr size_t FNV_BASIS = 14695981039346656037ull;
constexpr size_t FNV_PRIME = 1099511628211ull;

constexpr size_t fnv1a_hash(std::string_view str) {
    size_t hash = FNV_BASIS;
    for (char c: str) hash = (hash ^ c) * FNV_PRIME;
    return hash;
}

template<typename T>
constexpr size_t type_hash() {
    auto name = type_name<T>();
    return fnv1a_hash(name);
}
```

关键的部分基本都解决了。有了这些，我们就可以组装出`std::type_info`、`std::type_index`等类了。

## 可移植性

上面的代码中，`__PRETTY_FUNCTION__`是编译器相关的，其内容也是编译器相关的，需要针对不同编译器分别指定预定义变量/宏、`PREFIX`、`SUFFIX`。如果更改了相关函数的签名或者所在的命名空间，也需要相应地更改`PREFIX`和`SUFFIX`。其余的部分都是可移植的。

这里用到了 C++17 的`std::string_view`，它是可以 constexpr 构造的。而在 C++17 之前则可以直接返回`const char*`或者自己实现一个可 constexpr 构造的`std::string_view`类似物。

## 安全性

`__PRETTY_FUNCTION__`中的类型名是带有完整命名空间路径以及模板参数（如果有的话）的，在不违背 C++ 的[「单一定义规则」](https://zh.cppreference.com/w/cpp/language/definition)（One Definition Rule，ODR）的情况下，这一名字保证在每个翻译单元内唯一。若违背了 ODR，则程序非良构。

实际测试下，即使对于 C++20 的类类型模板参数（指 Class Types in Non-Type Template Parameters），GCC、Clang、MSVC 三家也均能输出其中具体的内容，从而保证名字唯一。而其他的编译器在写下这篇文章的时候都还没有支持这个特性。

## 更多

有一个名为 [CTTI](https://github.com/Manu343726/ctti) 的库是基于同样的原理写成的，本文在写作的过程中参考了它的代码。

---
title: "C++：异质查找（heterogeneous lookup）"
date: 2021-10-16
tags: [cpp]
showToc: false
---

~~太久没更新博文了，水一篇凑数~~

## 从 String View 说起

C 风格的字符串常常需要自己记录长度、管理生命周期，涉及长度变化时更是比较麻烦。于是在 C++ 中我们有了 `std::string`，并且有了与之配套的一系列函数，比如 `std::stoi`，对应 C 里面的 `atoi`。这个函数的声明如下：

```cpp
int stoi(const std::string& str, std::size_t* pos = 0, int base = 10);
```

这个接口接受一个 `const std::string&`，乍看或许是理所应当的：我是 C++ 函数，我需要读取字符串，但是我不需要修改它。实际上，C++ 中有很多接受 `const std::string&` 的函数，然而很遗憾，这个设计是失败的。

考虑这样一种情况，我们需要读取一个 `std::string` 里的子串，但我们不需要修改它。比如对一个拥有很多数字的字符串进行连续 parse ，或者在某个大文本里找到某个模板再对匹配结果进行进一步筛选。这种情况下，这些接受 `const std::string&` 的函数就变得不好用了，因为子串不是一个 `std::string` 对象。我们往往不得不把子串复制到一个新的 `std::string` 对象里，造成了额外的开销。

类似的常见情况还有，我们接收到了一个 C 风格的字符串，以 `char* str` + `size_t len` 的形式，而我们希望能在这个字符串上使用各种 C++ 函数的功能。比如跟 C 接口交互的时候，或者用 buffer 从别的地方接收字符串数据的时候。

所以要怎么解决呢？我们可以采用 C 风格的接口，即 `char* str` + `size_t len`，或者采用迭代器风格的接口，即 `char* begin` + `char* end`。而将这两个参数合起来，我们就得到了 `std::string_view` ——只读字符串接口的正确答案。它不仅比 `const std::string&` 更泛用，而且在没有发生 SSO 的情况下，它还比 `const std::string&` 减少了一次指针跳转。

Rust 很早就想明白了这个问题，一开始就提供了 `String`（对应 C++ 的 `string`）和 `&str`（对应 C++ 的 `string_view`），并且在各个接口上统一了用法。而 C++ 则是群魔乱舞，什么样的接口都有。

## 泛型的困境

假设你现在在写一个泛型容器 `map<K, V>`，你要给他添加一个 `.find` 成员来进行查找。那么 `.find` 的参数应该是什么呢？一般来说 `const K&` 就可以了，但要是 `K = std::string`，那么就会遇到前面所说的问题了。给 `std::string` 做一个特化吗？不，我们要考虑更一般的问题，即**如何在泛型里处理一个类型有多种表示的情况**。我们应该允许 `.find` 的参数是其他一些和 `K` 有关系的类型。C++ 的异质查找（heterogeneous lookup）说的就是这么样一种操作，异质在这里指的是存储的类型和查询的类型不一样。

要解决这个问题也并不是很困难。在 `std::map` 等容器中原本就有一个 `Compare` 泛型参数用于设置比较器，我们只要接收一个可以比较不同类型的比较器并且把 `.find` 的参数放宽到任意类型就可以了。

默认情况下，`Compare` 的值是 `std::less<Key>`，其 `operator()` 函数只能用于 `Key` 和 `Key` 的比较。在 C++14 后，`std::less` 等函数新增加了 `void` 特化，其 `operator()` 函数是一个模板，接受任意类型并返回 `std::forward<T>(lhs) < std::forward<U>(rhs)` 。我们只要将 `std::map` 的 `Compare` 参数设置成 `std::less<>` 即可使用异质查找。

```cpp
using namespace std::literals::string_view_literals;
std::map<std::string, int, std::less<>> foo;
foo.find("key"sv);
```

或者你也可以定制自己的比较器，只要它有个合法的成员类型 `is_transparent`，也会启用异质查找。

以上是对 `std::map` 和 `std::set` 而言的。对于 unordered 系列容器，则是要求 `Hash::is_transparent` 和 `KeyEqual::is_transparent`。另外 unordered 系列容器的异质查找在 C++20 才被加入。

Rust 里的解决方案则是用一个 `Borrow` trait 来描述这种关系，相对来说约束更大一些。有兴趣的可以自行阅读[文档](https://doc.rust-lang.org/std/borrow/trait.Borrow.html)，这里就不展开讲了。

很遗憾 C++ 太晚意识到这些问题了，导致留下了大量的失败设计。为了兼容性考虑，异质查找不能默认启用，这导致启用了异质查找的和不启用的两个 `std::map` 类型是不一样的。所以即使这项功能已经可用，你也可能受限于旧有接口。

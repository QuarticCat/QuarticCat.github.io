---
title: "C++：把字符串编码进类型里"
date: 2021-03-05
tags: [cpp]
---

又到了我第 114514 喜欢的类型体操环节。这篇文章不当正经人了，想到啥写啥。

## 这玩意有什么用

非要举一个具有实用意义的场景的话，[PEGTL](https://github.com/taocpp/PEGTL) 是我能想到的一个很好的例子。这是一个 parser combinator 库，它使用类型来组合 parser ，比如这样：

```cpp
struct separater: star<one<' ', '\t', '\r', '\n'>> {};
```

那么要 parse 一段字符串的时候当然就要把字符串信息编码进类型里面了。

我在自己的玩具 parser combinator 库里也用了这种方法，只不过我写法上使用变量来组合。

除此之外，著名的 [fmt](https://github.com/fmtlib/fmt) 库也用到了这个东西来进行大量的编译期字符串操作。但它为了兼容性，实现方式都比较原始，而且重复实现了大量标准库中后来加入或者被标记为`constexpr`的东西，也许我有时间会用 C++20 实现一个简易版的 fmt 库。

## 不就是个`char...`吗

看了上面的例子，肯定有人会这么想。但其实再想想，我们可以有好几种方案在模板参数里接收一个字符串：

```cpp
template<const char* /* , size_t N */>
struct Str1 {};

template<char...>
struct Str2 {};

// Need C++20, will explain later
template<SomeUserDefinedString>
struct Str3 {};
```

为什么这里不写数组类型呢，因为在模板参数里，数组类型会被自动替换成指针，和第一种方案实际上是一样的。总之就这么三个，我们一个一个来讲。

## 方案一

```cpp
template<const char* /* , size_t N */>
struct Str1 {};
```

把它放在第一个讲是因为它是最废物的，只要一写就发现：

```cpp
constexpr Str1<"123"> s1;
```

诶，编译器怎么拒绝了？GCC 在这里会直接告诉你 string literals can never be used in this context，而 Clang 则在打谜语。其实并非这个模板参数用不了，而是你得这么写才合法：

```cpp
constexpr const char chars[] = "123";
constexpr Str1<chars> s2;
```

这可太拉垮了，每个类型都得在某个地方先定义一个字符串才能用。而且这是两条语句而不是表达式，很多场景下没法用宏组合到一起。我想写出一套流畅的类型嵌套，或者至少，constexpr 变量表达式，怎么办？有经验的人可能会想到立即调用 lambda ：

```cpp
constexpr auto s3 = [] {
    constexpr const char chars[] = "123";
    return Str1<chars>{};
}();
```

然后咱再用宏把这 lambda 调用包一层，不就是个好用的表达式了吗？想得美，编译器立马把 error 拍你脸上。Clang 还在打谜语，就不提了，GCC 直接告诉你问题是这里模板参数需要静态存储期。于是你可能又想，咱加个`static`不就完事了：

```cpp
constexpr auto s4 = [] {
    static constexpr const char chars[] = "123";
    return Str1<chars>{};
}();
```

结果这个 lambda 就变成非 constexpr 的了，哈哈哈哈哈。总之，这条路大约的确是走不通了。

## 方案二

```cpp
template<char...>
struct Str2 {};
```

这是最常见的，也是还比较好用的。想要有字符串的体验只需要在结构体内部写个`constexpr const char chars[] = {Cs..., '\0'}`就完事了。定义也很直观，直接写 `Str2<'1', '2', '3'>`。唯一的问题就是，无关的字符太多了，字符串一长写起来就很难受。所以接下来就得想想怎么简化这个类型的书写。

### 用户定义字面量

一种可行的简化方法是用用户定义字面量。但很可惜，标准的字面量太弱了，没法取到字符串字面量的`char...`，只能取到数值字面量的。所以最多只能写出这种代码：

```cpp
template<char... Cs>
constexpr Str2<Cs...> operator""_s() {
    return {};
}

constexpr auto s5 = 123_s;  // OK
constexpr auto s6 = "abc"_s;  // Error
```

而只有依赖 GCC 与 Clang 的扩展，我们才能成功定义出期望的字面量：

```cpp
template<class CharT, CharT... Cs>
constexpr Str2<Cs...> operator""_s() {
    return {};
}

constexpr auto s7 = "123"_s;
```

当你期望取得一个类型而不是 constexpr 变量时，也只需要一个`decltype`就足够了：

```cpp
using T = decltype("123"_s);
```

如果嫌长可以用宏包一层。

### 模板展开

这个方法的灵感来自于 stackoverflow 上的[一个回答](https://stackoverflow.com/a/15912824/14258517)。原代码很长不过大部分都是废话，只不过是利用 C++11 后模板允许传局部 struct 这一点来传递临时字符串，在元函数里生成下标序列然后产生取下标表达式的展开，从而把临时字符串转化成字符参数序列。最后用个 lambda 包起来，就是个表达式了。

理解了原理以后很容易写出一个简化的版本。有了 C++14 的`std::index_sequence`，这种简单的递归完全不需要自己写：

```cpp
template<class S, size_t... Is>
constexpr auto helper(std::index_sequence<Is...>) {
    return Str2<S{}.s[Is]...>{};
}

#define STR(str)                                                       \
    [] {                                                               \
        struct S { const char* s = str; };                             \
        return helper<S>(std::make_index_sequence<sizeof(str) - 1>{}); \
    }()

constexpr auto s8 = STR("123");
```

注意这里还用到了 constexpr lambda ，这是一个 C++17 的特性。这个方法有个缺陷是在 C++20 前没法直接取得类型，因为这之前 lambda 不能用在不求值语境里，没法包个`decltype`。

而当我们来到了 C++20 ，有了 template lambda ，我们甚至可以只用四行写出这个宏：

```cpp
#define STR(str)                                   \
    []<size_t... Is>(std::index_sequence<Is...>) { \
        return Str2<str[Is]...>{};                 \
    }(std::make_index_sequence<sizeof(str) - 1>{})

constexpr auto s8 = STR("123");
```

只不过有个问题，既然都上了 C++20 ，那还不如直接看方案三。

### 宏展开

其实思路和模板展开是完全一致的，好处是可以直接获得类型，坏处就是写起来很长，以及允许的字符串长度有限制（但可以轻松扩展），而且字符串类型内部需要处理一下末尾的`0`。

```cpp
#define STR_EXPAND_1(str, i) \
    (sizeof(str) > (i) ? str[i] : 0)

#define STR_EXPAND_4(str, i) \
    STR_EXPAND_1(str, i+0),  \
    STR_EXPAND_1(str, i+1),  \
    STR_EXPAND_1(str, i+2),  \
    STR_EXPAND_1(str, i+3)

#define STR_EXPAND_16(str, i) \
    STR_EXPAND_4(str, i+0),   \
    STR_EXPAND_4(str, i+4),   \
    STR_EXPAND_4(str, i+8),   \
    STR_EXPAND_4(str, i+12)

#define STR_EXPAND_64(str, i) \
    STR_EXPAND_16(str, i+0),  \
    STR_EXPAND_16(str, i+16), \
    STR_EXPAND_16(str, i+32), \
    STR_EXPAND_16(str, i+48)

#define STR(str) Str2<STR_EXPAND_64(str, 0)>

using T = STR("123");
constexpr auto s9 = STR("123"){};
```

这个也是 PEGTL 内部使用的方法，因为 PEGTL 是用类型来组合的。

## 方案三

```cpp
template<SomeUserDefinedString>
struct Str3 {};
```

有了 C++20 对 NTTP 的扩展，我们终于可以写出来一个不需要任何宏和编译器扩展就方便好用的字符串模板参数了！原理也很简单，就是自己写一个符合 NTTP 要求的类，把字符串复制一份，这样模板参数就拥有字符串的所有权了，方案一遇到的那些问题也就不复存在。

```cpp
template<size_t N>
struct StrTP {
    char str[N];
    constexpr StrTP(const char (&s)[N]) {
        for (size_t i = 0; i < N; ++i) str[i] = s[i];
    }
};

template<StrTP S>
class Str3 {};

constexpr Str3<"123"> s10;
```

再包一层模板变量也是可以的：

```cpp
template<StrTP S>
constexpr Str3<S> str3{};

constexpr auto s11 = str3<"123">;
```

或者再包一层用户定义字面量也毫无问题：

```cpp
template<StrTP S>
constexpr Str3<S> operator""_s() {
    return {};
}

constexpr auto s12 = "123"_s;
```

我愿称之为最完美方案。

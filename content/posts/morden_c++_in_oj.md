---
title: "现代C++在算法竞赛中的应用"
date: 2021-01-20T11:40:28+08:00
draft: false
---

## 前言

C++是一门古老的编程语言，在算法竞赛中因为其优秀的运行效率和丰富的标准库，成为许多 acmer 和 oier 的
首选语言。在 C++11 面世以后，C++扩充许多特性，给 C++ 这门语言注入了新的活力。

这篇文章将讨论一些能在算法竞赛中提供方便的现代C++的特性

{{< admonition tips "编译环境" true >}}

大部分oj都已经支持了G++17, 本文所有代码都基于 GCC 9.2 。

{{< /admonition >}}

## auto

`auto`是c++11引入的新关键字，用于自动推导类型，使用auto可以让我们告别写冗长的类型名

```cpp
auto a = 1;//推导为int
auto t = std::make_tuple(1,1); // 推导为std::tuple<int,int>
```

## 区间 for 迭代

在 C++11 引入了基于范围的迭代，让可以使用类似于python中的`for x in list :`的写法对容器遍历

如果需要读取一个大小为n的数组，在以前我们可能这么做

```cpp
int n, a[maxn];
std::cin >> n;
for (int i = 1; i <= n; i++)
    cin >> a[i];
```

使用现代c++可以这么写

```cpp
int n;
cin >> n;
vector<int> a(n, 0);
for (auto &x : a)
    cin >> x;
```

## tuple

在传统C++中，我们要打包一组数据通常使用自定义`struct`，当需要打包的数据为两个时可以使用`std::pair`。
在现代C++中，你可以使用更方便的`std::tuple`.

使用`std::tuple`需要引入头文件 `<tuple>`，在cppreference中，将`std::tuple`描述为`std::pair`的推广,
将`pair`只能储存两个数值的限制去掉了

在C++17以前可以使用这种方式解包tuple

```cpp
int a, b;
auto x = std::make_tuple<int, int>(1, 1);
std::tie(a, b) = x;
auto c = std::get<0>(x);
```

在C++17中提供了一种结构化绑定的语法，可以很方便的解包tuple

```cpp
using namespace std;
auto x = tuple<int, tuple<int, int>>{1, {2, 3}};
auto &[a, x2] = x; // 还是rust的模式解构nb
auto &[b, c] = x2;
// a= 1,b= 2,c= 3
cout << "a= " << a << ",b= " << b << ",c= " << c << endl;
```

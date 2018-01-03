---
layout: post
title: Effective Modern C++ 笔记
date: 2018-01-03
author: Nicholas Huang
header-img: 
categories: C++
tags:
    - C++
--- 
# Effective Modern C++ 笔记
## Item 1: Understand template type deduction

```
    template<typename T>
    void f(ParamType param);
    
    f(expr);
```
此种情况：
>T的类型不仅和expr的类型独立，而且还和ParamType的形式独立。

三种情况：

1. ParamType是一个指针或者非universal的引用，它的推导过程：
	1. 如果expr的类型是一个引用，忽略引用的部分
	2. 然后利用expr的类型和ParamType对比判断T的类型
2. ParamType是一个universal的引用
    1. 如果expr的类型是一个左值，那么T和ParamType都被推导为左值引用
    2. 如果expr的类型是一个右值，那么和情况1一样
3. ParamType既不是指针也不是引用
    1. 如果expr的类型是一个引用，忽略引用的部分
    2. 如果在忽略了expr的引用符号后，还有const、volatile，也忽略他们
    这个主要是因为在这种情况是传值，传值就涉及拷贝。因此实际的参数和传入的参数是相互独立的。

如果实际参数是数组，那么情况要分形参是传值还是传引用，如果是传值，则推导成指针，如果是穿引用，则推导为对数组的引用。因此，可以用传引用这个方式来获得数组的大小。

```
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N;
}

int keyVals[] = {1, 3, 7, 9, 11, 22, 35};
int mappedVals[arraySize(keyVals)];
std::array<int, arraySize(keyVals)> mappedVals;
```

函数做参数时的推导同数组一样


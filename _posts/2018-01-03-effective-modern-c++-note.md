---
layout: post
title: Effective Modern C++ 笔记
date: 2018-01-03
updated: 2018-01-04
author: Nicholas Huang
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

## Item2: Understand auto type deduction
auto的推导和Item1中讲的template的推导，除了一个例外，其余都是相同的。
>As you can see, auto type deduction works like template type deduction. They're essentially two sides of the same coin.

那么例外的情况如下：

```
auto x1 = 27;   // type is int, value is 27
auto x2(27);    // ditto
auto x3 = {27}; // type is std::initializer_list<int>, value is {27}
auto x4{27};    // ditto
```
可以看到第一个和第二个auto的推导和Item1中的规则是一样，但是，第三个和第四个auto的推导是根据这么一条规则：
>When the initializer for an auto-declared variable is enclosed in braces, the deduced type is a std::initializer_list. If such a type can't be deduced(e.g., because the values in the braced initializer are of different types), the code will be rejected.

对于braced initializers在template中的运用如下：

```
template<typename T>
void f(T param);
f({1, 2, 3});//error! can't deduce type for T

template<typename T>
void f(std::initializer_list<T> initList);
f({1, 2, 3});// T deduced as int, and initList's type is std::initializer_list<int>
```

为啥对于braced initializers——也就是C++11新增的那个统一初始化——auto推导和template推导不同，大佬Scott Meyers和你一样，也想知道，摊手.jpg。
在C++14中，auto可以用在函数的返回类型上和lamda的参数声明上。实际上，这些地方的auto都是用模板类型推导规则而不是auto推导规则，也就是说braced initializer是不能用到这些地方的。

# Item3: Understand decltype
通常情况：

```
const int i = 0;        //decltype(i) is const int
bool f(const Widget& w);//decltype(w) is const Widget&
                        //decltype(f) is bool(cont Widget&)

struct Point {
    int x, y;           //decltype(Point::x) is int, same as Point::y
};

Widget w;               //decltype(w) is Widget

if(f(w))...             //decltype(f(w)) is bool

template<typename T>
class vector{
    public:
        ...
        T& operator[](std::size_t index);
        ...
};
vector<int> v;          //decltype(v) is vector<int>
if(v[0] == 0)...        //decltype(v[0]) is int&
```
在C++11中，auto单独用在表示函数返回类型的时候，不会执行类型推导，需要搭配尾随返回类型(trailing return type)，代码如下：

```
template<typename Container, typename Index>        //need refinement
auto authAndAccess(Container& c, Index i)
  -> decltype(c[i])
{
    authenticateUser();
    return c[i];
}

template<typename Container, typename Index>        //final version
auto authAndAccess(Container&& c, Index i)          //because we do not know 
-> decltype(std::forward<Container>(c)[i])          //what c will be, a rvalue or 
{                                                   //a lvalue, so we use 
    authenticateUser();                             //universal reference
    return std::forward<Container>(c)[i];
}
```


```
template<typename Container, typename Index>        //need refinement
decltype(auto)
authAndAccess(Container& c, Index i)
{
    authenticateUser();
    return c[i];
}

template<typename Container, typename Index>        //final version
decltype(auto)                                      //because we do not know
authAndAccess(Container&& c, Index i)               //what c will be, a rvalue or       
{                                                   //a lvalue, so we use 
    authenticateUser();                             //universal reference
    return std::forward<Container>(c)[i];
}
```
还有就是decltype对于变量名和比变量名复杂的左值表达式的推导是不同的，如下：

```
int x = 0;          //decltype(x) is int
int (x) = 0;        //decltype((x)) is int&
```
因此在C++14中应用decltype(auto)的时候要注意返回语句中的一个细小改变会影响度你这个函数的推导类型，如下：

```
decltype(auto) f1()
{
    int x = 0;
    ...
    return x;           //decltype(x) is int, so f1 returns int
}

decltype(auto) f2()
{
    int x = 0;
    ...
    return (x);         //decltype((x)) is int&, so f2 returns int& 
}
```



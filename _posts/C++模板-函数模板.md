---
title: c++模板-函数模板
date: 2022-06-24 09:19:00
tags: c++模板
category:
---
---
## 定义如下：
```
#include <iostream>
#include 
template<typename T>
T max(T a , T b)
{
    return b>a?b : a;
}

int main()
{
    int i = 42;
    std
}
```
---
## 两个阶段编译
### 1.  在模板定义阶段，模板的检查并不包含类型参数的检查。只包含下面几个方面:  

* 语法检查。比如少了分号。   

* 使用了未定义的不依赖于模板参数的名称（类型名，函数名，......）。  

* 未使用模板参数的 static assertions。  

### 2.  在模板实例化阶段，为确保所有代码都是有效的，模板会再次被检查，尤其是那些依赖类型参数的部分  

---
## 模板类型推断
在类型推断的时候自动的类型转换是受限制的:  
* 如果调用参数是按引用传递的，任何类型转换都不被允许。通过模板类型参数 T 定义的 两个参数，它们实参的类型必须完全一样。  
* 如果调用参数是按值传递的，那么只有退化（decay）这一类简单转换是被允许的：const 和 volatile 限制符会被忽略，引用被转换成被引用的类型，raw array 和函数被转换为相 应的指针类型。通过模板类型参数 T 定义的两个参数，它们实参的类型在退化（decay） 后必须一样  
---
## 多模板参数
```
 template<typename T1, typename T2>
 T1 max(T1 a, T2 b)
 {
    return b < a ? a : b;
 }
```
可能造成返回值与T1类型相关，如T1 = 66.66 T2 = 42 返回值为66.66；如果T1 = 42，T2 = 66.66返回值则为66
解决此问题有三种方法如下：
* **为返回类型引入第三个模板参数**
```
 template<typename T1,typename T2,typename RT>
    RT max(T1 a, T2 b)
    {
        return b < a ? a : b;
    }
    //调用如下
    ::max<int,double,int>(7.2,4); //麻烦，啰嗦
```
到目前为止，我们已经看到了显式地提到所有函数模板参数或没有函数模板参数的情况。另一种方法是只显式指定第一个参数，并允许演绎过程派生其他参数。通常，必须指定最后一个不能隐式确定的参数类型之前的所有参数类型。因此，如果你在我们的例子中改变了模板参数的顺序，调用者只需要指定返回类型,代码如下：
```
template<typename RT,typename T1,typename T2>
    RT max(T1 a, T2 b)
    {
        return b < a ? a : b;
    }
    ::max<double>(7.2,2); // 返回类型为double,输入参数类型推导;
```
* **让编译器找出返回类型**。  
需要声明返回类型为auto代码如下：
```
template<typename T1,typename T2>
    auto max(T1 a, T2 b)
    {
        return b < a ? a : b;
    }
```
不使用尾置返回类型(trailing return type)的情况下将auto 用于返回类型要求返回类型必须是能够通过函数体推导出来，因此要有这样用来推断返回类型的返回语句，并且多个返回语句之间的推断结果必须一致。
在c++14 之前，要想让编译器推断出返回类型，就必须让或多或少的函数实现成为函数声明的一部分，在c++11中，尾置返回类型(trailing return type)允许我们使用函数的调用参数。可以基于运算符?:的结果声明返回类型：
```
template<typename T1,typename T2>
    auto max(T1 a, T2 b) -> decltype(b<a?a:b)
    {
        return b < a ? a : b;
    }
```
事实上可以修改成如下代码
```
template<typename T1,typename T2>
    auto max(T1 a, T2 b) -> decltype(true?a:b)
    {
        return b < a ? a : b;
    }
```
然而，在任何情况下，这个定义都有一个显著的缺陷:返回类型可能是引用类型，因为在某些条件下T可能是引用。由于这个原因，你应该返回从T衰变的类型，如下所示:
```
 template<typename T1,typename T2>
    auto max(T1 a, T2 b) -> typename std::decay<decltype(true?a:b)>::type
    {
        return b < a ? a : b;
    }
```
在这里我们用到了类型萃取（type trait）std::decay<>，它返回其 type 成员作为目标类型， 定义在标准库<type_trait>中。由于其 type 成员是一个类型，为了获取其结果， 需要用关键字 typename 修饰这个表达式。

**这里请注意auto 初始化总是类型退化。这也适用于auto为返回值**。
```
int i = 42;
int const& ir = i;
auto a = ir ; a 退化为类型int不是int const& 
```
* **将返回类型声明为两个参数类型的“公共类型”**。  
自c++ 11以来，c++标准库提供了一种方法来指定选择“更通用的类型”。
std::common_type<>::type,如下：
``` 
template<typename T1,typename T2>
typename  std::common_type<T1, T2>::type  max(T1 a, T2 b)
{
    return b < a ? a : b;
}
```
---
## 默认模板参数
1. 如下：
```
template<typename T1,typename T2,typename RT = std::decay_t<decltype(true?T1() : T2())>>
    RT max(T1 a, T2 b)
    {
        return b < a ? a : b;
    }
```
2. 代码如下：
```
template<typename T1,typename T2,typename RT = std::common_type_t<T1,T2>>
    RT max(T1 a, T2 b)
    {
        return b < a ? a : b;
    }
```

3.代码如下：
```
 template<typename RT = long, typename T1,typename T2>
    RT max(T1 a, T2 b)
    {
        return b < a ? a : b;
    }
```
---
## 重载函数模板
在所有其他因素相同的情况下，模板解析将优先选择**非模板函数**，而不是从模板实例化出来的函数。

---

---
title: C++模板-泛型库
date: 2022-08-01 16:37:18
tags:
---
## 可调用对象
在c++中，由于一些类型既可以被作为函数调用参数使用，也可以按照f(...)的形式调用，因此也可以被用作回调参数：
*  函数指针类型；
*  重载了operator()的class类型（有时候被称为仿函数），包括lambda表达式
*  包含一个可以产生一个函数指针或者函数引用的转换函数的class类型；
这些类型统称为函数对象类型，对应的值成为函数对象。

### 一: 函数对象的支持
我们看一下标准库的for_earch()算法是如何实现的。
```
namespace _11_1_1_
{
    template<typename Iter,typename Callable>
    void foreach(Iter current, Iter end, Callable op)
    {
        while (current != end)
        {
            op(*current);
            ++current;
        }
    }

    void func(int i)
    {
        std::cout << "func() called for: " << i << std::endl;
    }

    class FuncObj
    {
    public:
        void operator()(int i) const
        {
            std::cout << "FuncObj::op() called for : " << i << std::endl;
        }

    };

}

int main()
{
    // 11.1.1
    {
        std::vector<int> primes = { 1,2,3,4,5,6,7,8 };
        _11_1_1_::foreach(primes.begin(), primes.end(),_11_1_1_::func);
        _11_1_1_::foreach(primes.begin(), primes.end(), _11_1_1_::func);
        _11_1_1_::FuncObj funcObj;
        _11_1_1_::foreach(primes.begin(), primes.end(), funcObj);
        _11_1_1_::foreach(primes.begin(), primes.end(), [](int i) {
            std::cout << "lambda func [" << i << "]" << std::endl;
            });
    }
    return 0;
}
```
详细看一下以上各情况
* 当把函数名当作函数参数传递时，并不是传递函数本体，而是传递其指针或者引用。和 数组情况类似，在按值传递时，函数参数退化为指针，如果参数类型是 模板参数，那么类型会被推断为指向函数的指针。 和数组一样，按引用传递的函数的类型不会 decay。但是函数类型不能真正用 const 限 制。如果将 foreach()的最后一个参数的类型声明为 Callable const &，const 会被省略。 （通常而言，在主流 C++代码中很少会用到函数的引用。） 
* 在第二个调用中，函数指针被显式传递（传递了一个函数名的地址）。这和第一中调用 方式相同（函数名会隐式的 decay 成指针），但是相对而言会更清楚一些。 
* 如果传递的是仿函数，就是将一个类的对象当作可调用对象进行传递。通过一个 class 类型进行调用通常等效于调用了它的 operator()。因此下面这样的调用： op(*current);
* Lambda 表达式会产生仿函数（也称闭包），因此它与仿函数（重载了 operator()的类） 的情况没有不同。不过 Lambda 引入仿函数的方法更为简便，因此它们从 C++11 开始变 得很常见。有意思的是，以[]开始的 lambdas（没有捕获）会产生一个向函数指针进行转换的运算 符。但是它从来不会被当作 surrogate call function，因为它的匹配情况总是比常规闭包 的 operator()要差。

## 二: 处理成员函数以及额外的参数
以上的例子漏掉另一种情况，成员函数。从C++17开始，标准库提供一个工具：std::invoke()包含成员函数forearch代码如下：
```

namespace _11_1_2_
{
    class MyClass
    {
    public:
        void memfunc(std::string a, int i) const
        {
            std::cout << "MyClass::memfunc() call for:[" << i << "]" << a << std::endl;
        }
    private:

    };

    template<typename Iter ,typename Callable,typename... Args>
    void foreach(Iter current, Iter end, Callable op, Args const... args)
    {
        while (current != end)
        {
            std::invoke(op, args..., *current);
            ++current;
        }
    }

    void func(int i)
    {
        std::cout << "func() called for: " << i << std::endl;
    }

    class FuncObj
    {
    public:
        void operator()(std::string a, int i) const
        {
            std::cout << "FuncObj::op(" << a << ") called for : " << i << std::endl;
        }

    };
}

int main()
{
    // 11.1.2
    std::vector<int> primes = { 1,2,3,4,5,6,7,8 };
    _11_1_2_::foreach(primes.begin(), primes.end(), [](std::string a, int i) {
        std::cout << "输出：" << a << ":" << i << std::endl;
    }, "hello");
    _11_1_2_::MyClass obj;
    _11_1_2_::foreach(primes.begin(), primes.end(), &_11_1_2_::MyClass::memfunc, obj, "world");

    _11_1_2_::foreach(primes.begin(), primes.end(), _11_1_2_::func);
    _11_1_2_::FuncObj funcObj = _11_1_2_::FuncObj();
    _11_1_2_::foreach(primes.begin(), primes.end(), funcObj,"_11_1_2_");
}
```
*  如果可调用对象是一个指向成员函数的指针，它会将 args...中的第一个参数当作 this 对 象（不是指针）。Args...中其余的参数则被当做常规参数传递给可调用对象。
* 否则，所有的参数都被直接传递给可调用对象。   
<b>注意这里对于可调用对象和 agrs..都不能使用完美转发（perfect forward）：因为第一次调用 可能会 steal(偷窃)相关参数的值，导致在随后的调用中出现错误。</b>

### 函数调用的包装

## 二: 其他一些泛型库的工具
### 1. 类型萃取
标准库提供各种各样的萃取工具，它们可以被用来计算以及修改类型。这样就可以在实例化的时候让泛型代码适用各种类型或者对不同类型做出不同的响应：

### 2. std::addressoff()
函数模板 std::addressof<>()会返回一个对象或者函数的准确地址。即使一个对象重载了运算 符&也是这样  
### 3. std::declval()
函数模板 std::declval()可以被用作某一类型的对象的引用的占位符。该函数模板没有定义， 因此不能被调用（也不会创建对象）。因此它只能被用作不会被计算的操作数（比如 decltype 和 sizeof）。也因此，在不创建对象的情况下，依然可以假设有相应类型的可用对象。
## 三: 完美转发临时变量
## 四: 作为模板参数的引用
## 五: 推迟计算
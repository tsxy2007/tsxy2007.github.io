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
```
```
如上所示，通过检查某些条件，可以在模板的不同实现之间做选择。在这里用到了编译期if特性（c++17开始可用），作为替代，也可以使用std::enable_if,部分特例化或者SFINAE。   
使用类型萃取的时候必须要额外小心：其行为可能和程序员的预期不同。   
1. 删除引用和const顺序问题
```
std::remove_const_t<std::remove_reference_t<int const&>> // int std::remove_reference_t<std::remove_const_t<int const&>> // int const
```
2. 某些情况下，结果可能会让你很意外：
```
add_rvalue_reference_t<int const> // int const&& add_rvalue_reference_t<int const&> // int const& (lvalueref remains lvalue-ref)
```
这里我们期望能够返回一个右值引用，但是c++**引用塌缩**会令左值引用和右值引用返回左值引用。
### 2. std::addressoff()
函数模板 std::addressof<>()会返回一个对象或者函数的准确地址。即使一个对象重载了运算 符&也是这样  
### 3. std::declval()
函数模板 std::declval()可以被用作某一类型的对象的引用的占位符。该函数模板没有定义， 因此不能被调用（也不会创建对象）。因此它只能被用作不会被计算的操作数（比如 decltype 和 sizeof）。也因此，在不创建对象的情况下，依然可以假设有相应类型的可用对象。
## 三: 完美转发临时变量
我们可以使用转发引用以及std::forward<>来完美转发泛型参数如下：
```
template<typename T> 
void f (T&& t) // t is forwarding reference 
{ 
    g(std::forward<T>(t)); // perfectly forward passed argument t to g() 
}
```
但是某些情况下，在泛型代码中我们需要转发一些不是通过参数传递进来的数据。此时我们可以使用auto&& 创建一个可以被转发的变量。比如，假设我们需要相继调用get()和set()两个函数，并且需要将Get()的返回值完美转发给set()
```
template<typename T> 
void foo (T&& t) //
{ 
    set(get(x));
}
```
假设以后需要我们更新代码对get()的返回值进行某些操作，可以通过将get()的返回值存储在一个被声明为auto&& 的变量中实现：
```
template<typename T> 
void foo(T x) 
{ 
auto&& val = get(x); …// perfectly forward the return value of get() to set(): 
set(std::forward<decltype(val)>(val)); 
}
```
## 四: 作为模板参数的引用
不是很常见，但是模板参数的类型依然可以是引用类型；代码如下：
```
namespace _11_4_
{
    template<typename T>
    void tmplParamIsReference(T)
    {
        std::cout << "T is reference: " << std::is_reference_v<T> << std::endl;
    }
}

int main()
{
 using namespace _11_4_;
 std::cout << std::boolalpha;
 int i;
 int& r =  
 tmplParamIsReference(i); // false
 tmplParamIsReference(r);// false
 tmplParamIsReference<int&>(i);// true
 tmplParamIsReference<int&>(r);// true
 return 0;
}
```
这样做可以从根本上改变模板的行为，不过由于这并不是模板最初设计的目的，这样做可能触发错误或者不可预知的行为.如下的例子：
```
namespace _11_4_
{

    template<typename T, T Z = T{} >
    class RefMem
    {
    public:
        RefMem() : zero{ Z } {}
    private:
        T zero;
    };
}
int main()
{
     RefMem<int> rm1, rm2;
     rm1 = rm2 
     //RefMem<int&> rm3; // 需要左值
     //RefMem<int&, 0>rm4; // 不能从int 转换成int&
}
```
此处模板的模板参数为T，其非类型模板参数Z被进行了零初始化。用int实例化该模板会获得预期的行为。
* 非模板参数的默认初始化不在可行。 
* 不再能够直接用 0 来初始化非参数模板参数。 
* 最让人意外的是，赋值运算符也不再可用，因为对于具有非 static 引用成员的类，其默 赋值运算符会被删除掉。

而且将引用类型用于非类型模板参数同样会变的复杂和危险，考虑如下例子：
```
template<typename T,int& SZ>
    class Arr
    {
    private:
        std::vector<T> elems;
    public:
        Arr() : elems(SZ)
        { //use current SZ as initial vector size 
        }

        void print() const
        {
            for (int i = 0; i < SZ; ++i)
            {
                //loop over SZ elements std::cout << elems[i] << ’ ’; 
            }
        }
    };

int main()
{
    //Arr<int&, size> x;// error 编译错误在vector;
      Arr<int, size> y;
      y.print();
      size += 100; //modifies SZ in Arr<>
      y.print();// run-time ERROR: invalid memory access: loops over 120 elements
}
```
注意这里并不能通过将 SZ 声明为 int const &来修正这一错误，因为 size 本身依然是可变的。 看上去这一类问题根本就不会发生。但是在更复杂的情况下，确实会遇到此类问题。比如在 C++17 中，非类型模板参数可以通过推断得到：   
```
template<typename T, decltype(auto) SZ>   
class Arr;  
``` 
使用 decltype(auto)很容易得到引用类型，因此在这一类上下文中应该尽量避免使用 auto。  
基于这一原因，C++标准库在某些情况下制定了很特殊的规则和限制。
* 在模板参数被用引用类型实例化的情况下，为了依然能够正常使用赋值运算符， std::pair<>和 std::tuple<>都没有使用默认的赋值运算符，而是做了单独的定义。
* 由于这些副作用可能导致的复杂性，在 C++17 中用引用类型实例化标准库模板 std::optional<>和 std::variant<>的过程看上去有些古怪。
## 五: 推迟计算
在实现模板的过程中，有时候需要面对是否考虑不完整类型的问题。
代码如下：
```

namespace _11_5_
{
    
    template<typename T>
    class Cont
    {
    private:
        T* elems;
    public:
        typename std::conditional_t<std::is_move_constructible_v<T>,T&&,T&> foo() // 编译报错不允许使用不完整类型。
        {

        }
    };

    struct Node
    {
        std::string value;
        Cont<Node> next;
    };
}
```
解决办法就是延迟计算代码如下
```

namespace _11_5_
{
    
    template<typename T>
    class Cont
    {
    private:
        T* elems;
    public:
        template<typename D = T> // 类型萃取依赖于模板参数 D
        typename std::conditional_t<std::is_move_constructible_v<D>,T&&,T&> foo() //
        {

        }
    };

    struct Node
    {
        std::string value;
        Cont<Node> next;
    };
}
```
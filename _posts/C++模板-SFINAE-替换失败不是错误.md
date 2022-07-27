---
title: C++模板-SFINAE/替换失败不是错误
date: 2022-07-27 11:26:48
tags:
---
## 一: 模板元编程
C++模板的实例化发生在编译期间(而动态语言的泛型实在程序运行期间决定的)。事实证明c++模板的某些特性可以和实例化过程相结合，这样就产生了一种c++自己内部的原始递归的“编程语言”。因此模板可以用来“计算一个程序的结果”。
```
namespace _8_1_1_
{
    //模板开始递归
    template<unsigned p,unsigned d>
    struct DoIsPrime
    {
        static constexpr bool value = (p % d != 0) && DoIsPrime<p, d - 1>::value;
    };

    //模板递归终止条件
    template<unsigned p>
    struct DoIsPrime<p,2>
    {
        static constexpr bool value = (p % 2 != 0);
    };

    //计算质数入口
    template<unsigned p>
    struct IsPrime
    {
        static constexpr bool value = DoIsPrime<p, p / 2>::value;
    };
    // <=3 数字直接给出答案;
    template<> struct IsPrime<0> { static constexpr bool value = false; };
    template<> struct IsPrime<1> { static constexpr bool value = false; };
    template<> struct IsPrime<2> { static constexpr bool value = true; };
    template<> struct IsPrime<3> { static constexpr bool value = true; };

}

int main()
{
    std::cout << "模板元编程:" << _8_1_1_::IsPrime<11>::value <<std::endl;
    std::cout << "模板元编程:" << _8_1_1_::IsPrime<10>::value << std::endl;
}
```
## 二: 通过constexpr进行计算
c++11引入了一个叫做constexpr的新特性，它大大简化了各种类型的编译期计算。如果给定合适的输入，constexpr就可以在编译期完成相应的计算。虽然c++11对constexpr函数的使用者有诸多限制，但是在c++14中这些限制中单独大部分都被移除了。不能使用场景：
* 堆内存分配
* 异常抛出   
质数的实现的constexpr方式如下：
### c++11实现方式
```
namespace _8_2_1_
{
    constexpr bool doIsPrime(unsigned p, unsigned d)
    {
        return d != 2 ? (p % d != 0) && doIsPrime(p, d - 1)  : (p%2!=0);
    }

    constexpr bool IsPrime(unsigned p)
    {
        return p < 4 ? !(p < 2) : doIsPrime(p, p / 2);
    }
    
    const bool b1 = isPrime(13); // 编译期

     // 运行期
    bool fiftySevenIsPrime()
    {
        return IsPrime(57);
    }
}

int main()
{
    unsigned a = 10;
    constexpr bool isprime1 = _8_2_1_::IsPrime(11);
    std::cout << "constexpr:" << isprime1 << std::endl; //编译期
    std::cout << "constexpr:" << _8_2_1_::IsPrime(11) << std::endl; //运行期
    std::cout << "constexpr helper:" << _8_2_1_::Helper<11>::value << std::endl; //编译期
    std::cout << "constexpr:" << _8_2_1_::IsPrime(a) << std::endl; // 运行期
    std::cout << "constexpr helper:" << _8_2_1_::Helper<10>::value << std::endl;
    return 0;
}
```
### c++14实现方式如下:
```
namespace _8_2_2_
{
    constexpr bool isPrime(unsigned int p)
    {
        for (unsigned i = 2; i <= p/2; i++)
        {
            if (p % i == 0)
            {
                return false;
            }
        }
        return p > 1;
    }
    const bool b1 = isPrime(13); // 编译期

    // 运行期
    bool fiftySevenIsPrime() 
    {
        return isPrime(57);
    }
}

int main()
{
    unsigned a = 10;
    constexpr bool isprime1 = _8_2_2_::isPrime(11);//编译期
    bool isprime2 = _8_2_2_::isPrime(a);
    std::cout << "_8_2_2_constexpr:" << isprime1 << std::endl; 
    std::cout << "_8_2_2_constexpr:" << _8_2_2_::isPrime(11) << std::endl; //运行期
    std::cout << "_8_2_2_constexpr:" << isprime2 << std::endl; // 运行期
}
```
<b>constexpr 可以在编译期执行，但是不一定会在编译期执行</b>

总结下就是：
1. constexpr + 参数是常量 执行在编译期
2. 参数常量 执行在运行期
3. 如果 b1 被定义于全局作用域或者 namespace 作用域，也会在编译期进行计算。
4. 参数是变量都会在执行期执行

## 三: 通过部分特例化进行路径选择
可以利用isPrime()在编译期通过部分特例化在不同的实现方案之间做选择代码如下：

### 模板元编程
```
    template<int SZ,bool = IsPrime<SZ>::value>
    struct Helper;

    template<int SZ>
    struct Helper<SZ, false>
    {
        static constexpr bool value = false;
    };

    template<int SZ>
    struct Helper<SZ, true>
    {
        static constexpr bool value = true;
    };
```

### constexpr方式:
```
    template<int SZ, bool = IsPrime(SZ)>
    struct Helper;

    template<int SZ>
    struct Helper<SZ,false>
    {
        static constexpr bool value = false;
    };

    template<int SZ>
    struct Helper<SZ, true>
    {
        static constexpr bool value = true;
    };
```

<b>由于函数模板不支持部分特例化</b>，当基于一些限制在不同的函数实现之间做选择时，必须要使用其它一些方法： 
* 使用有 static 函数的类;
* 使用 6.3 节中介绍的 std::enable_if; 
* 使用下一节将要介绍的 SFINAE 特性;
* 或者使用从 C++17 开始生效的编译期的 if 特性;

## 四: SFINAE(替换 失败不是错误)
C++中，重载函数以支持不同类型的参数是很常规的操作。当编译期遇到一个重载函数的调用时，它必须分别考虑每个重载版本，已选择其中类型最匹配的那一个。     
在一个函数调用的备选方案中包含函数模板时，编译器首先要决定应该将什么样的模板参数 用于各种模板方案，然后用这些参数替换函数模板的参数列表以及返回类型，最后评估替换 后的函数模板和这个调用的匹配情况（就像常规函数一样）。但是这一替换过程可能会遇到 问题：替换产生的结果可能没有意义。不过这一类型的替换不会导致错误，C++语言规则要求忽略掉这一类型的替换结果。 这一原理被称为 SFINAE（发音类似 sfee-nay），代表的是“substitution failure is not an error”。  
但是上面讲到的替换过程和实际的实例化过程不一样（参见 2.2 节）：即使对那些最终被证 明不需要被实例化的模板也要进行替换（不然就无法知道到底需不需要实例化）。不过它只 会替换直接出现在函数模板声明中的相关内容（不包含函数体）。
考虑如下例子：
```
namespace _8_4_0_
{
    template<typename T,unsigned N>
    std::size_t len(T(&)[N])
    {
        return N;
    }

    template<typename T>
    typename T::size_type len(T const& t)
    {
        return t.size();
    }
}

int main()
{
     // 数组
    int a[10];
    std::cout << "_8_4_0_ no size():" << _8_4_0_::len(a) << std::endl;
    std::cout << "_8_4_0_ no size():" << _8_4_0_::len("tmp") << std::endl;
    // 其他数据结构
    std::vector<int> v;
    v.push_back(1);
    std::cout << "_8_4_0_ has size():" << _8_4_0_::len(v) << std::endl;
    // 指针 
    int* p = new int(0);
    //std::cout << "_8_4_0_ has point:" << _8_4_0_::len(p) << std::endl;
    // allocator 没有匹配项
    std::allocator<int> x;
    //std::cout << "_8_4_0_ allocator:" << _8_4_0_::len(x) << std::endl;
    return 0;
}
```
1. 第一个函数模板的参数类型是T(&)[N],也就是说他是一个包含了N个T型元素的数组。
2. 第二个函数模板的参数类型就是简单的 T，除了返回类型要是 T::size_type 之外没有别的 限制，这要求被传递的参数类型必须有一个 size_type 成员。

* 当传入参数为int a[10]，"tmp"这类数组时只有裸数组的函数模板能够匹配，如果只从函数签名来看的话，对第二函数模板也可以分别用int[10]和const char[4]替换类型参数T，但是这种替换在处理返回类型T::size_type时会导致错误。因此对于这两个调用，第二个函数模板会被忽略掉。
* 当传递的是裸指针话，以上两个模板都不会被匹配上。此时编译期会报错没有发现合适len函数。
* std::allocator<int> x;此时会匹配到第二个函数模板。因此不会报错说没有发现合适的len()函数，而是会报一个编译期错误说对std::allocator<int>而言size()是个无效调用。此时第二个模板函数不会被忽略掉。

###  对所有类型的应急选项
代码如下：
```
std::size_t len(...)
{
    return 0;
}
```

此处额外提供了一个通用函数 len()，它总会匹配所有的调用，但是其匹配情况也总是所有 重载选项中最差的（通过省略号...匹配）  
此时对于裸数组和 vector，都有两个函数可以匹配上，但是其中不是通过省略号（...）匹配 的那一个是最佳匹配。对于指针，只有应急选项能够匹配上，此时编译器不会再报缺少适用 于本次调用的 len()。不过对于 std::allocator<int>的调用，虽然第二个和第三个函数都能匹配 上，但是第二个函数依然是最佳匹配项。因此编译器依然会报错说缺少 size()成员函数

### 通过 decltype 进行 SFINAE（此处是动词）的表达式
对于有 size_type 成员但是没有 size()成员函数的参数类型，我们想要保证会忽略掉函 数模板 len()。如果没有在函数声明中以某种方式要求 size()成员函数必须存在，这个函数模 板就会被选择并在实例化过程中导致错误：
```
 template<typename T>
    typename T::size_type len(T const& t)
    {
        return t.size();
    }
```
处理这一情况有一种常用模式或者说习惯用法： 
* 通过尾置返回类型语法（trailing return type syntax）来指定返回类型（在函数名前使用 auto，并在函数名后面的->后指定返回类型）。 
* 通过 decltype 和逗号运算符定义返回类型。 
* 将所有需要成立的表达式放在逗号运算符的前面（为了预防可能会发生的运算符被重载 的情况，需要将这些表达式的类型转换为 void）。 
* 在逗号运算符的末尾定义一个类型为返回类型的对象。
```
template<typename T>
auto len(T const& t)->decltype((void)t.size(),T::size_type())
{
    return t.size();
}
```
类型指示符 decltype 的操作数是一组用逗号隔开的表达式，因此最后一个表达式 T::size_type() 会产生一个类型为返回类型的对象（decltype 会将其转换为返回类型）。而在最后一个逗号 前面的所有表达式都必须成立，在这个例子中逗号前面只有 t.size()。之所以将其类型转换为 void，是为了避免因为用户重载了该表达式对应类型的逗号运算符而导致的不确定性。 注意 decltype 的操作数是不会被计算的，也就是说可以不调用构造函数而直接创建其 “dummy”对象，
## 五: 编译期 if
   部分特例化，SFINAE 以及 std::enable_if 可以一起被用来禁用或者启用某个模板。而 C++17 又在此基础上引入了同样可以在编译期基于某些条件禁用或者启用相应模板的编译期 if 语 句。通过使用 if constexpr(...)语法，编译器会使用编译期表达式来决定是使用 if 语句的 then 对应的部分还是 else 对应的部分。
   代码如下：
```
namespace _8_5_
{
    template <typename T,typename ... Types>
    void print(T const& firstArg, Types const& ... args)
    {
        std::cout << firstArg ;
        if constexpr(sizeof...(args))
        {
            std::cout<< ';';
            print(args...);
        }
        else // 无论如何都会被执行;
        {
            std::cout << std::endl;
        }
    }
}
```
如果只给print()传递一个参数，那么args...就是一个空的参数包，此时sizeof...(args)==0.这样if语句里面的语句就会被丢弃吊，也就是说这部分代码不会被实例化。因此也就不再需要一个单独的函数来终结递归。
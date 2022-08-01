---
title: C++模板-移动语义和enable_if<>
date: 2022-07-18 23:53:50
tags:
---
## 一：完美转发
我们希望实现的泛型代码可以将被传递参数的基本特性转发出去：
* 可变对象转发之后依然可变
* const对象转发之后依然是const
* 可移动对象(可以从窃取资源的对象)被转发之后依然可以是可移动的。
不使用模板，就需要针对以上三种情况分别编程，代码如下：
```

namespace _6_1_0_
{
    class X
    {
    public:
    private:

    };

    void g(X&)
    {
        std::cout << "g() for variable" << std::endl;
    }

    void g(X const&)
    {
        std::cout << "g() for constant" << std::endl;
    }

    void g(X&&)
    {
        std::cout << "g() for movable object" << std::endl;
    }

    void f(X& val)
    {
        g(val);
    }

    void f(X const& val)
    {
        g(val);
    }

    void f(X&& val)
    {
        g(std::move(val));// 必须使用std::move因为移动语义不会一起传递
    }
}
```

如果使用一下模板
```
template<Typename T>
void f(T t)
{
    g(val);
}
```
这种只适用于前两种情况，对于第三种移动对象的情况无效。为了解决这种情况c++11引入了完美转发(perfect forwarding)。代码如下：
```
namespace _6_1_1_
{
    class X
    {

    };

    void g(X&)
    {
        std::cout << "g() for variable" << std::endl;
    }

    void g(X const&)
    {
        std::cout << "g() for constant" << std::endl;
    }

    void g(X&&)
    {
        std::cout << "g() for movable object" << std::endl;
    }

    template<typename T>
    void f(T&& val)
    {
        g(std::forward<T>(val));
    }
}
```
模板的T&&和具体类型X的X&&看起来是一样的。但是它们适用于不同的规则
* 具体类型X的X&&声明了一个右值引用参数。只能被绑定到一个可移动的对象上(一个prvalue，临时对象。一个xvalue ，比如通过std::move()传递的参数)。
* 模板参数T的T&&声明一个转发引用(万能引用)。可以被绑定到可变，不可变或者可移动对象上。在函数内部这个参数也可以是可变，不可变或者指向一个可以被窃取的内部数据的值。
---
## 二：特殊成员函数
### 1. 基本写法
特殊成员函数也可以是模板，比如构造函数。考虑一下例子：
```
namespace _6_2_0_
{
    class Person
    {
    public:
        explicit Person(std::string const& n) : name(n)
        {
            std::cout << "copying string-CONSTR for '" << name << "'" << std::endl;
        }

        explicit Person(std::string&& n) : name(std::move(n))
        {
            std::cout << "moving string-CONSTR for '" << name << "'" << std::endl;
        }

        Person(Person const& p) : name(p.name)
        {
            std::cout << "COPY-CONSTR Person'" << name << "'\n";
        }

        Person(Person&& p) :name(std::move(p.name))
        {
            std::cout << "MOVE-CONSTR Person ’" << name << "’\n";
        }
    private:
        std::string name;
    };
}
```
一个以std::string类型的name成员和一个初始化构造函数。为了支持移动语义，重载了接受std::string作为参数的构造函数。
* 一个以std::string对象为参数，并用其副本来初始化name成员
* 一个以可移动std::string对象作为参数，并通过std::move()从中窃取值来初始化name
### 2. 使用基础模板实现
现在我们将使用模板方法重构参数string的构造函数。代码如下:
```
namespace _6_2_1_
{
    class Person
    {
    private:
        std::string name;
    public:
        template<typename STR>
        explicit Person(STR&& n) :name(std::forward<STR>(n))
        {
            std::cout << "TMPL-CONSTR for '" << name << "'\n";
        }

        Person(Person const& p) : name(p.name)
        {
            std::cout << "COPY-CONSTR Person'" << name << "'\n";
        }

        Person(Person&& p) : name(std::move(p.name))
        {
            std::cout << "Move-CONSTR Person'"<< name << "'\n";
        }
    };
}
```
以上代码绝大部分情况都正确除了一下情况
```
std::string s ="name";
_6_2_1_::Person p1 {s};
_6_2_1_::Person p2{p1}; // 错误
```
问题出在根据c++重载解析规则，对于一个非const 左值Person p，成员模板   
template<typename STR>  
Person(STR&& n);  
通常比预定义的拷贝构造函数更匹配  
 Person(Person const& p)   
## 三：通过std::enable_if<>禁用模板
c++11开始，c++标准提供了辅助模板std::enable_if<>，可以在某些编译期条件下忽略函数模板。比如：
```
template<typename T>
typename std::enable_if<sizeof(T) > 4>::Type foo ()
{
    //...
}
```
也就是说std::enable_if<>是一种类型萃取(type trait),它会根据一个作为其（第一个）模板参数的编译期表达式决定其行为：
* 如果这个表达式结果为true，它的成员会返回一个类型：  
 -- 如果这个表达式没有第二个模板，返回类型为void。  
 -- 否则返回类型是其第二个参数的类型
 * 如果表达式结果是false，则其成员类型是未定义的。根据模板的一个叫做SFINAE（替换失败不是错误）这会导致包含std::enable_if<>表达式的函数模板被忽略掉。

 ## 四：使用enable_if<>
使用enable_if<>解决构造函数模板问题： 
当传递的模板参数类型不正确的时候，禁用以下构造函数
```
template<typename STR>
explicit Person(STR&& n) :name(std::forward<STR>(n))
{
    std::cout << "TMPL-CONSTR for '" << name << "'\n";
}
```
改造后代码
```
namespace _6_2_1_
{
    class Person;
    template<typename T>
    using EnableIfString = std::enable_if_t<std::is_convertible_v<T, std::string>>;

    template<typename T>
    using EnableIfPerson = std::enable_if_t<std::is_convertible_v<T, Person>>;

    class Person
    {
    private:
        std::string name;
    public:
        template<typename STR,typename = EnableIfString<STR> /*std::enable_if_t<std::is_convertible_v<STR,std::string>>*/> // 可以直接写在这里。
        explicit Person(STR&& n) :name(std::forward<STR>(n))
        {
            std::cout << "TMPL-CONSTR for '" << name << "'\n";
        }

        Person(Person const& p) : name(p.name)
        {
            std::cout << "COPY-CONSTR Person'" << name << "'\n";
        }

        Person(Person&& p) : name(std::move(p.name))
        {
            std::cout << "Move-CONSTR Person'"<< name << "'\n";
        }
    };
}
```
### 禁用某些成员函数
注意我们不能通过使用enable_if<>来禁用copy/move构造函数以及赋值构造函数。这是因为成员函数模板不会算作特殊成员函数（依然会生成默认构造函数），而且在需要使用copy构造函数的地方，相应的成员函数模板会被忽略掉。
```
template<typename T,std::enable_if_t<std::is_convertible_v<T, Person>> >
Person(T&& p) :name(p.name)
{
    std::cout << "TMPL-Copy Person for '" << name << "'\n";
}
int main()
{
    std::string s ="name";
_6_2_1_::Person p1 {s};
_6_2_1_::Person p2{p1};
}
```
p2 在构造的时候并不会调用模板构造函数。而是调用默认构造函数。

## 五：使用concenpt简化enable_if<>表达式

即使使用了模板别名，enable_if 的语法依然显得很蠢，因为它使用了一个变通方法：为了达 到目的，使用了一个额外的模板参数，并且通过“滥用”这个参数对模板的使用做了限制。 这样的代码不容易读懂，也使模板中剩余的代码不易理解。
原则上我们所需要的只是一个能够对函数施加限制的语言特性，当这一限制不被满足的时 候，函数会被忽略掉。 这个语言特性就是人们期盼已久的 concept，可以通过其简单的语法对函数模板施加限制条 件。不幸的是，虽然已经讨论了很久，但是 concept 依然没有被纳入 C++17 标准。一些编译 器目前对 concept 提供了试验性的支持，不过其很有可能在 C++17 之后的标准中得到支持（目 前确定将在 C++20 中得到支持）。通过使用 concept 可以写出下面这样的代码：
```

namespace _6_5_0_
{
    class Person;
    template<typename T>
    concept ConvertibleToString = std::is_convertible_v<T, std::string>;

    class Person
    {
    private:
        std::string name;
    public:
        template<typename STR>
        requires ConvertibleToString<STR>
        explicit Person(STR&& n) :name(std::forward<STR>(n))
        {
            std::cout << "TMPL-CONSTR for '" << name << "'\n";
        }

        Person(Person& p) : name(p.name)
        {
            std::cout << "normal Copy Person for '" << name << "'\n";
        }

        Person(Person&& p) : name(p.name)
        {
            std::cout << "rvalue Copy Person for '" << name << "'\n";
        }

        Person(const Person& p) : name(p.name)
        {
            std::cout << "const Copy Person for '" << name << "'\n";
        }

    };
}

```
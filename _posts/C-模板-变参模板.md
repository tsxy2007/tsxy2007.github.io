---
title: C++模板-变参模板
date: 2022-07-09 16:42:51
tags:
---

## 一:变参模板
### 1.实例
可以将模板参数定义成能够接受任意多个模板参数的情况。这类模板被称为变参模板。
```
void print()
{
}

template<typename T, typename ... Types>
void print(T FirstArg, Types... args)
{
    std::cout << FirstArg << std::endl;
    print(args...);
}
```
### 2.变参和非变参模板的重载
```
template<typename T>
void print(T arg)
{
    std::cout << arg << std::endl;
}
template<typename T, typename ... Types>
void print(T FirstArg, Types... args)
{
    print(FirstArg);
    print(args...);
}
```
### 3.sizeof... 运算符
sizeof...既可以用于模板参数包，也可以用于函数参数包
```
template<typename T, typename ... Types>
void print(T FirstArg, Types... args)
{
    std::cout << sizeof...(Types) << " "<< sizeof...(args) << std::endl;
}
```
## 二: 折叠表达式
从C++17开始，提供一种可以用来计算参数包（可以有初始值）中所有参数运算结果的二元运算符。
几乎所有的二元运算符都可以用于折叠表达式。  

```
    template<typename T>
    T Test(const T& t)
    {
        return t;
    }
    template<typename... T>
    auto foldSum(T ... s)
    {
        return (... + Test(s));
    }

    template<typename T>
    class AddSpace
    {
    private:
        T const& ref;
    public:
        AddSpace(T const& r) : ref(r) {}

        friend std::ostream& operator<<(std::ostream& os, AddSpace<T> s)
        {
            return os << s.ref << ' ';
        }
    };

    template<typename... Args>
    void print(Args... args)
    {
        (std::cout << ... << AddSpace<Args>(args)) << std::endl;
    }
```
## 三: 变参类模板和变参表达式
### 1.变参表达式
除了转发所有参数之外，还可以做些别的事情。如下：
```
    template<typename T>
    class AddSpace
    {
    private:
        T const& ref;
    public:
        AddSpace(T const& r) : ref(r) {}

        friend std::ostream& operator<<(std::ostream& os, AddSpace<T> s)
        {
            return os << s.ref << ' ';
        }
    };

    template<typename... Types>
    void print(Types... args)
    {
        (std::cout<< ... << AddSpace<Types>(args)) << std::endl;
    }

    template<typename ...Types>
    void printDoubled(Types const& ... args)
    {
        print(args + args...);
    }

    template<typename T1,typename... TN>
    constexpr bool isHomogeneous(T1, TN...)
    {
        return (std::is_same<T1, TN>::value && ...);
    }
```

### 2.变参下标
可以通过一组变参下标来访问第一个参数中相应的元素
```
    template<typename C, typename... Idx>
    void printElems(C const& coll, Idx... idx)
    {
        print(coll[idx]...);
    }

    template<std::size_t... Idx,typename C>
    void printIdx(C const& Coll)
    {
        print(Coll[Idx]...);
    }
```
### 3.变参类模板
类模板也可以是变参的。一个重要的例子是，通过任意多个模板参数指定了 class 相应数据 成员的类型：
```
template<typename… Elements>class Tuple; Tuple<int, std::string, char> t; 
```
### 4.变参推断指引
```
namespace std { template<typename T, typename… U> array(T, U…) -> array<enable_if_t<(is_same_v<T, U> && …), T>, (1 + sizeof…(U))>; }
```
### 5. 变参基类及其使用

```
namespace _4_4_5_
{
    class Customer
    {
    public:
        Customer(std::string const& n) : name(n){}
        std::string getName()const { return name; }
    private:
        std::string name;
    };

    struct CustomerEq
    {
        bool  operator()(Customer const& c1, Customer const& c2) const
        {
            return c1.getName() == c2.getName();
        }
    };

    struct CusomerHash
    {
        std::size_t operator()(Customer const& c)const
        {
            return std::hash<std::string>()(c.getName());
        }
    };

    template<typename... Bases>
    struct Overloader : Bases...
    {
        using Bases::operator()...;
    };
}
```
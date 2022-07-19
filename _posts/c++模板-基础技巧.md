---
title: C++模板-基础技巧
date: 2022-07-13 15:25:00
tags: c++模板
category:
---

## 一: typename关键字
typename用来澄清模板内部的一个标识符代表的某种类型，而不是数据成员。如下：
```
namespace _5_1_
{
    class A
    {
    public:
        class B
        {
        public:
            B(){}
            ~B(){}

        private:

        };
    public:
        A(){}
        ~A(){}

    private:

    };

    template<typename T>
    class MyClass
    {
    public:
        typename T::B* Ptr;

    private:

    };
}
```
其中第二个typename被用来澄清SubType是定义在ClassT中的一个类型。因此这里Ptr是一个指向T:SubType类型的指针。

## 二: 零初始化
对于基础类型，比如int,double以及指针类型，由于他们没有默认构造函数，因此它们不会默认初始化成一个有意义的值。比如一下：
```
void foo()
{
    int x;      //x 为未定义值
    int* ptr;   //ptr 指针可以是任何地址；
}
```
因为在定义模板时，如果想让一个模板类型的变量被初始化成一个默认值，可以通过下面的写法就可以保证得到适量的初始化
```
template<typename T>
    void foo()
    {
        T x{}; // c++11 after
        //T x = T();// c++ 11 Before
        std::cout << x << std::endl;
    }
```

## 三: 使用this->
对于类模板，如果它的基类也是依赖于模板参数的，那么对它而言即使x是继承而来的，使用this->x和x也不一定是等效的。
```
namespace _5_3_
{
    template<typename T>
    class Base
    {
    public:
        void bar()
        {
            std::cout << "this is base bar()" << std::endl;
        }
    private:

    };

    template<typename T>
    class Derived : public Base<T>
    {
    public:
        void foo()
        {
            //bar(); // error Derived没有bar方法
            //this->bar(); // 正确 通过this调用Base类方法；
            Base<T>::bar();// 正确 通过Base<T>::调用Base类方法；
        }
    };
}
```

## 四: 使用裸数组或者字符串常量的模板
如果参数是按引用传递的，那么参数类型不会退化（decay）。只有当按值传递参数时，模板类型才会退化，我们可以定义专门用来处理裸数组或者字符串常量的模板
也可以边界未知的数组做重载或者部分特化：
```
namespace _5_4_
{
    template<typename T,int N,int M>
    bool less(T(&a)[N], T(&b)[M]) // 引用未退化
    {
        for (int i = 0; i < N && i < M; i++)
        {
            if (a[i]<b[i])
            {
                return true;
            }
            if (b[i] < a[i])
            {
                return false;
            }
        }
        return N < M;
    }

    template<int N,int M>
    bool less(char(&a)[N], char(&b)[M])
    {
        for (int i = 0; i < N && i < M; i++)
        {
            if (a[i] < b[i])
            {
                return true;
            }
            if (b[i] < a[i])
            {
                return false;
            }
        }
        return N < M;
    }

    template<typename TT>
    struct MyClass
    {

    };

    template<typename T,std::size_t SZ>
    struct MyClass<T[SZ]>
    {
        static void print() 
        { 
            std::cout << "print() for T[" << SZ << "]\n"; 
        }
    };

    template<typename T,std::size_t SZ>
    struct MyClass<T(&)[SZ]>
    {
        static void print() 
        { 
            std::cout << "print() for T(&)[" << SZ << "]\n"; 
        }
    };

    template<typename T>
    struct MyClass<T[]>
    {
        static void print()
        {
            std::cout << "print() for T[]" << std::endl;
        }
    };

    template<typename T>
    struct MyClass<T(&)[]>
    {
        static void print()
        {
            std::cout << "print() for T(&)[]" << std::endl;
        }
    };

    template<typename T>
    struct MyClass<T*>
    {
        static void print()
        {
            std::cout << "print() for T*" << std::endl;
        }
    };


    template<typename T1,typename T2,typename T3>
    void foo(int a1[7], int a2[], int(&a3)[42], int(&x0)[], T1 x1, T2& x2, T3&& x3)
    {
        MyClass<decltype(a1)>::print(); // 使用MyClass<T*> 退化成指针
        MyClass<decltype(a2)>::print();// 使用MyClass<T*> 退化成指针
        MyClass<decltype(a3)>::print();// 使用MyClass<T(&)[SZ]> 
        MyClass<decltype(x0)>::print(); //  使用MyClass<T(&)[]> 
        MyClass<decltype(x1)>::print();//  使用MyClass<T*>
        MyClass<decltype(x2)>::print();//  使用MyClass<T(&)[]>
        MyClass<decltype(x3)>::print();//  使用MyClass<T(&)[]> 万能引用， 引用折叠

    }
}
```

## 五: 成员模板
### 1.基本定义
类的成员也可以是模板，对嵌套类和成员函数都是这样。作用和优点同样通过stack<>类模板得到展现。通常只有当两个Stack类型相同的时候才可以相互赋值。即便两个元素之间可以相互隐士转会，也不能赋值。但是可以将赋值运算符定义成模板，就可以将两个元素类型做转换赋值。如下：
```

namespace _5_5_
{
    template<typename T>
    class Stack
    {
    private:
        std::deque<T> elems;
    public:
        void push(T const&);
        void pop();
        T const& top() const;
        bool empty()const
        {
            return elems.empty();
        }

        template<typename T2>
        Stack& operator= (Stack<T2> const&);

        void print()
        {
            for (size_t i = 0; i < elems.size(); i++)
            {
                std::cout << elems[i] << std::endl;
            }
        }
    };

    template<typename T>
    void Stack<T>::push(T const& a)
    {
        elems.push_back(a);
    }


    template<typename T>
    void Stack<T>::pop()
    {
        assert(!elems.empty());
        elems.pop_back();
    }

    template<typename T>
    T const& Stack<T>::top()const
    {
        assert(!elems.empty());
        return elems.back();
    }

    template<typename T>
    template<typename T2>
    Stack<T>& Stack<T>::operator= (Stack<T2> const& op2)
    {
        Stack<T2> tmp(op2);
        elems.clear();
        while (!tmp.empty())
        {
            elems.push_front(tmp.top);
            tmp.pop();
        }
        return *this;
    }
 }
```
以上代码有两处改动
1. 赋值运算符的参数是一个元素类型为T2的stack
2. 新的模板使用std::deque<>作为内部容器，方便赋值运算符的定义
   
在模板函数内部，你希望简化op2中相关元素的访问。但是由于op2属于另一种类型。所以只能只能访问公共成员，为了可以访问op2的私有成员，可以将其他类型的stack模板的实例都定义成友元如下：
```
namespace _5_5_
{
    template<typename T>
    class Stack
    {
    private:
        std::deque<T> elems;
    public:
        void push(T const&);
        void pop();
        T const& top() const;
        bool empty()const
        {
            return elems.empty();
        }

        template<typename T2>
        Stack& operator= (Stack<T2> const&);

        void print()
        {
            for (size_t i = 0; i < elems.size(); i++)
            {
                std::cout << elems[i] << std::endl;
            }
        }
        
        template<typename> friend class Stack; // 友元
    };

    template<typename T>
    void Stack<T>::push(T const& a)
    {
        elems.push_back(a);
    }


    template<typename T>
    void Stack<T>::pop()
    {
        assert(!elems.empty());
        elems.pop_back();
    }

    template<typename T>
    T const& Stack<T>::top()const
    {
        assert(!elems.empty());
        return elems.back();
    }

    template<typename T>
    template<typename T2>
    Stack<T>& Stack<T>::operator= (Stack<T2> const& op2)
    {
        // 直接访问op2的私有成员
        elems.clear();
        elems.insert(elems.begin(),
            op2.elems.begin(),
            op2.elems.end());
        return *this;
    }
 }
```
也可以将内部的容器类型参数化如下：
```
namespace _5_5_0_
{
    template<typename T, typename Cont = std::deque<T>>
    class Stack
    {
    private:
        Cont elems;
    public:
        void push(T const&);
        void pop();
        T const& top()const;
        bool empty()const
        {
            return elems.empty();
        }

        template<typename T2,typename Cont2>
        Stack& operator=(Stack<T2, Cont2> const&);

        template<typename, typename> friend class Stack;
    };

    template<typename T, typename Cont>
    void Stack<T, Cont>::push(T const& item)
    {
        elems.push_back(item);
    }

    template<typename T, typename Cont>
    void Stack<T, Cont>::pop()
    {
        ASSERT(!elems.empty());
        elems.pop_back();
    }

    template<typename T, typename Cont>
    T const& Stack<T, Cont>::top()const
    {
        ASSERT(!elems.empty());
        return elems.back();
    }

    template<typename T, typename Cont>
    template<typename T1, typename Cont1>
    Stack<T,Cont>& Stack<T, Cont>::operator=(Stack<T1, Cont1> const& op1)
    {
        elems.clear();
        elems.insert(elems.begin(),
            op1.elems.begin(),
            op1.elems.end());
        return *this;
    }
}
```
### 2.成员模板的特例化
成员函数模板也可以被全部或者部分特例化
```
class BoolString
    {
    private:
        std::string value;
    public:
        BoolString(std::string const& s) : value(s)
        {}

        template<typename T = std::string>
        T get() const {
            // get value (converted to T) 
            return value;
        }
    };

    //为防止多个文件引入，避免重复定义的错误，必须将它定义成inline的。
    template<>
    inline bool BoolString::get<bool>() const 
    { 
        return value == "true" || value == "1" || value == "on"; 
    }
```
### 3.特殊成员函数的模板
如果能够通过特殊成员函数 copy 或者 move 对象，那么相应的特殊成员函数（copy 构造函 数以及 move 构造函数）也将可以被模板化。和前面定义的赋值运算符类似，构造函数也可 以是模板。但是需要注意的是，构造函数模板或者赋值运算符模板不会取代预定义的构造函 数和赋值运算符。成员函数模板不会被算作用来 copy 或者 move 对象的特殊成员函数。在 上面的例子中，如果在相同类型的 stack 之间相互赋值，调用的依然是默认赋值运算符。 这种行为既有好处也有坏处： 
* 某些情况下，对于某些调用，构造函数模板或者赋值运算符模板可能比预定义的 copy/move 构造函数或者赋值运算符更匹配，虽然这些特殊成员函数模板可能原本只打 算用于在不同类型的 stack 之间做初始化. 
*  想要对 copy/move 构造函数进行模板化并不是一件容易的事情，比如该如何限制其存 在的场景。

### 4.template的使用
某些情况下，在调用成员模板的时候需要显式的指定其模板参数类型。这时候需要使用关键字template确保模板符号<会被理解为模板参数列表的开始，而不是 比较运算符。
如下：
```
 template<unsigned long N>
    void printBitset(std::bitset<N> const& bs)
    {
        std::cout << bs.template to_string<char, std::char_traits<char>,
            std::allocator<char>>();
    }
```
### 4.泛型lambdas和成员模板
如下lambdas表达式
```
[] (auto x, auto y) { return x + y; }
```
编译器会默认为它构造下面这样一个类
```
 class SomeCompilerSpecificName
    {
    public:
        template<typename T1,typename T2>
        auto operator()(T1  x, T2 y) const
        {
            return x + y;
        }
    };
```
## 六:变量模板
从c++14开始，变量也可以被某种类型参数化，称为变量模板
如下：
```
    template<typename T = long double>
    constexpr T pi{ 3.1415926535897932385 };
```
### 数据成员的变量模板
```
template<typename T> 
class MyClass 
{ 
    public: 
    static constexpr int max = 1000; 
};
```
### 类型萃取 Suffix_v
从 C++17 开始，标准库用变量模板为其用来产生一个值（布尔型）的类型萃取定义了简化方 式。比如为了能够使用：
```
namespace std 
{ 
    template<typename T> 
    constexpr bool is_const_v = is_const<T>::value; 
}
```
## 七:模板参数模板
模板的参数也可以是个模板，会有不少好处，依然使用stack类模板举例：对于如下
```
template<typename T,typename Cont = std::deque>
class Stack
{

}
```
如果不想使用默认容器，那么就需要我们两次指出stack元素的类型。如下：
```
Stack<int,std::vector<int>> intStack;
```
如果我们在声明stack类模板的时候如下定义
```
Staack<int,std::vector> intStack;
```
就需要我们定义stack的时候将第二个模板参数声明为模板参数模板。代码如下：
```
namespace _5_7_
{
    template<typename T,template<typename Elem, typename _Alloc = std::allocator<Elem>> typename Cont = std::deque>
    class Stack
    {
    private:
        Cont<T> elems;
    public:
        void push(T const&);
        void pop();
        T const& top()const;
        bool empty()const
        {
            return elems.empty();
        }

        template<typename T2, template<typename Elem2, typename _Alloc = std::allocator<Elem2>> typename Cont2>
        Stack<T, Cont>& operator=(Stack<T2, Cont2> const& op2);

        template<typename, template<typename, typename>class>
        friend class Stack;
    };

    template<typename T, template<typename, typename>class Cont>
    void Stack<T, Cont>::push(T const& elem)
    {
        elems.push_back(elem);
    }

    template<typename T, template<typename, typename>class Cont>
    void Stack<T, Cont>::pop()
    {
        assert(!elems.empty());
        elems.pop_back();
    }

    template<typename T, template<typename, typename>class Cont>
    T const& Stack<T, Cont>::top()const
    {
        assert(!elems.empty());
        return elems.back();
    }

    template<typename T, template<typename, typename>class Cont>
    template<typename T2, template<typename, typename> typename Cont2>
    Stack<T, Cont>& Stack<T, Cont>::operator=(Stack<T2, Cont2> const& op1)
    {
        elems.clear();
        elems.insert(elems.begin(),
            op1.elems.begin(),
            op1.elems.end());
        return *this;
    }

}

```
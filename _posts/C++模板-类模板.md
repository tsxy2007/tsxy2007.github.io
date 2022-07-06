---
title: c++模板-类模板
date: 2022-07-04 08:58:00
tags: c++模板
category:
---
---
## 一:定义如下
```
namespace _2_1_
{
    template<typename T>
    class Stack
    {
    public:
        void push(T const& elem);
        void pop();
        T const& top()const;
        //1. 在模板类里定义
        bool empty() const
        {
            return elems.empty();
        }
    private:
        std::vector<T> elems;
    };
    //2.第二种方法 类外
    template<typename T>
    void Stack<T>::push(T const& elem)
    {
        elems.push_back(elem);
    }

    template<typename T>
    void Stack<T>::pop()
    {
        assert(!elems.empty());
        T elem = elems.back();
        elems.pop_back();
        return elem;
    }

    template<typename T>
    T const& Stack<T>::top() const
    {
        assert(!elems.empty());
        return elems.back();
    }

    template <typename T>
    class C
    {
        static_assert(std::is_default_constructible<T>::value, "Class c Requaires default constructible elements");
    public:
        //C(){}
        //~C(){}

    private:

    };

    class Test
    {
    public:
        Test() = default;
    };

}
```
---

## 二:Concept
用来表示一组反复被模板库要求的限制条件。（可随机进入的迭代器）和（可默认构造的）
代码如下：
```
  template <typename T>
    class C
    {
        static_assert(std::is_default_constructible<T>::value, "Class c Requaires default constructible elements");
    public:
        //C(){}
        //~C(){}

    private:

    };
    class Test
    {
    public:
        Test() = default; 
    };
    int main(  )
    {
        _2_1_::C <_2_1_::Test> _2_1_C;// 如果添加参数会报错
    }
```
---
## 三:友元

### 1. 可以隐式的声明一个新的函数模板，但是必须使用一个不同于类模板的模板参数，比如 用 U：
```
 template<typename T>
    class Stack
    {
    public:
        void push(T const& elem);
        void pop();
        T const& top()const;
        //1. 在模板类里定义
        bool empty() const
        {
            return elems.empty();
        }

        void printOn(std::ostream& strm) const
        {
            size_t Num = elems.size();
            for (size_t i = 0; i < Num; i++)
            {
                elems[i].printOn(strm);
            }
        }

        // 可以如此定义
        template<typename U>
        friend std::ostream& operator<<(std::ostream& strm, Stack<U> const& s)
        {
            s.printOn(strm);
            return strm;
        }

        const T& operator[](size_t Index) const
        {
            assert(Index < elems.size());
            return elems[Index];
        }
    private:
        std::vector<T> elems;

    };

    template<typename T>
    void Stack<T>::push(T const& elem)
    {
        elems.push_back(elem);
    }

    template<typename T>
    void Stack<T>::pop()
    {
        assert(!elems.empty());
        T elem = elems.back();
        elems.pop_back();
        return elem;
    }

    template<typename T>
    T const& Stack<T>::top() const
    {
        assert(!elems.empty());
        return elems.back();
    }

```
### 2. 也可以先将 Stack<T>的 operator<<声明为一个模板，这要求先对 Stack<T>进行声明：
```
namespace _2_4_
{

    template<typename T>
    class Stack;
    template<typename T>
    std::ostream& operator<< (std::ostream&, Stack<T> const&);

    template<typename T>
    class Stack
    {
    public:
        void push(T const& elem);
        void pop();
        T const& top()const;
        //1. 在模板类里定义
        bool empty() const
        {
            return elems.empty();
        }

        void printOn(std::ostream& strm) const
        {
            size_t Num = elems.size();
            for (size_t i = 0; i < Num; i++)
            {
                elems[i].printOn(strm);
            }
        }

        friend std::ostream& operator<<(std::ostream& strm, Stack<T> const& s)
        {
            s.printOn(strm);
            return strm;
        }

        const T& operator[](size_t Index) const
        {
            assert(Index < elems.size());
            return elems[Index];
        }
    private:
        std::vector<T> elems;

    };

    template<typename T>
    void Stack<T>::push(T const& elem)
    {
        elems.push_back(elem);
    }

    template<typename T>
    void Stack<T>::pop()
    {
        assert(!elems.empty());
        T elem = elems.back();
        elems.pop_back();
        return elem;
    }

    template<typename T>
    T const& Stack<T>::top() const
    {
        assert(!elems.empty());
        return elems.back();
    }


    class Test
    {
    public:
        Test() = default;

        void printOn(std::ostream& strm) const
        {
            strm << "hello" << std::endl;
        }
    };
}
```
---
## 四:模板类的特化
模板类的特化类似函数模板的重载,允许我们对某一特定类型做优化，或者修正类模板针对某一特定类型实例化之后的行为。
一下为代码例子
```
template<>
    class Stack<std::string>
    {
    public:
        void push(std::string const& str);
        void pop();
        std::string const& top()const;
        bool empty()const
        {
            return elems.empty();
        }

        friend std::ostream& operator<< (std::ostream& os, Stack<std::string> const& s)
        {
             size_t Num = s.elems.size();
            for (size_t i = 0; i < Num; i++)
            {
                 os << s.elems[i] << ";" ;
            }
            return os;
        }
    private:
        std::deque<std::string> elems;
    };

    void Stack<std::string>::push(std::string const& elem)
    {
        elems.push_back(elem);
    }

    void Stack<std::string>::pop()
    {
        assert(!elems.empty());
        elems.pop_back();
    }

    std::string const& Stack<std::string>::top()const
    {
        assert(!elems.empty());
        return elems.back();
    }
}

```
---
## 五:部分特化
类模板可以只被部分的特例化。这样就可以为某些特殊情况提供特殊的实现，不过使用者还 是要定义一部分模板参数。
代码如下：
```
template<typename T>
    class Stack<T*>
    {
    private:
        std::vector<T*> elems;
    public:
        void push(T* elem);
        void pop();
        T* top()const;
        bool empty()const
        {
            return elems.empty();
        }
    };

    template<typename T>
    void Stack<T*>::push(T* elem)
    {
        elems.push_back(elem);
    }

    template<typename T>
    void Stack<T*>::pop()
    {
        assert(!elems.empty());
        elems.pop_back();
    }

    template<typename T>
    T* Stack<T*>::top()const
    {
        assert(!elems.empty()); 
        return elems.back();
    }

```
---
## 六:多模板的部分特例化
```
namespace _2_5_ 
{
    template<typename T1,typename T2>
    class MyClass
    {

    };

    template<typename T>
    class MyClass<T, T>
    {

    };

    template<typename T>
    class MyClass<T, int>
    {

    };

    template<typename T1,typename T2>
    class MyClass<T1*, T2*>
    {

    };
}
```
---
## 七:默认模板参数
和函数模板一样，也可以给类模板的模板参数指定默认值。
```

namespace _2_7_
{
    // 模板参数Cont为默认模板参数
    template<typename T, typename Cont = std::vector<T>>
    class Stack
    {
    private:
        Cont elems;
    public:
        void push(T const& elem);
        void pop();
        T const& top()const;
        bool empty()const
        {
            return elems.empty();
        }
    };

    template<typename T,typename Cont>
    void Stack<T, Cont>::push(T const& elem)
    {
        elems.push_back(elem);
    }

    template<typename T, typename Cont>
    void Stack<T, Cont>::pop()
    {
        assert(!elems.empty());
        elems.pop_back();
    }

    template<typename T, typename Cont>
    T const& Stack<T, Cont>::top()const
    {
        assert(!elems.empty());
        return elems.back();
    }
}
```
---
## 八:类型别名
通过给类模板定义一个新的名字，可以使类模板的使用变得更方便。两种方式
### 1.typedef
```
typedef Stack<int> IntStack;
```
### 2. Alis声明
```
using IntStack = Stack <int>; 
```
### 3. 别名模板
```
template<typename T> 
using DequeStack = Stack<T, std::deque<T>>;
```
### 4.成员模板别称
例如各种容器类的Iterator
### 5. 类型萃取
```
namespace std 
{ 
    template<typename T>
    using add_const_t = typename add_const<T>::type;
}
```
---
## 九: 类模板的类型推导
从c++17开始如果构造函数能够推断出所有模板参数的类型，就不再显式的指明模板参数的类型
代码如下：
```

namespace _2_9_
{
    template<typename T>
    class Stack
    {
    private:
        std::vector<T> elems;
    public:
        Stack() = default;
        Stack(T const& elem)
            :elems({ elem })
        {}

        T const& top()const
        {
            assert(!elems.empty());
            return elems.back();
        }
    };
    // 推断指引
    _2_9_::Stack(const char*)->_2_9_::Stack<std::string>;
    Stack(const bool)->Stack<int>;
}
```
请注意一下细节
#### 1.由于定义了接受 int 作为参数的构造函数，要记得向编译器要求生成默认构造函数及其 全部默认行为，这是因为默认构造函数只有在没有定义其它构造函数的情况下才会默认 生成，方法如下： Stack() = default

#### 2.在初始化 Stack 的 vector 成员 elems 时，参数 elem 被用{}括了起来，这相当于用只有一 个元素 elem 的初始化列表初始化了 elems: : elems({elem})

### 类模板对字符串常量参数的类型推断
原则上可以通过字符串常量来初始化Stack:
```
Stack stringStack = "bottom";//推断为char const[7]
```
不过这样会带来一堆问题：当参数是按照 T 的引用传递的时候（上面例子中接受一个参数的 构造函数，是按照引用传递的），参数类型不会被 decay，也就是说一个裸的数组类型不会 被转换成裸指针。这样我们就等于初始化了一个这样的 Stack:
```
Stack< char const[7]>
```
不过如果参数是按值传递的，参数类型就会被 decay，也就是说会将裸数组退化成裸指针。 这样构造函数的参数类型 T 会被推断为 char const *，实例化后的类模板类型会被推断为 Stack<char const *>。

### 推断指引
针对以上问题，除了将构造函数声明成按值传递的，还有一个解决方案：由于在容器中处理 裸指针容易导致很多问题，对于容器一类的类，不应该将类型推断为字符的裸指针（char const *）。
代码如上；

## 十: 聚合类的模板化
同时还可以推断指引代码如下：
```
namespace _2_10_
{
    template<typename T>
    struct ValueWithComment
    {
        T value;
        std::string comment;
    };

    ValueWithComment(char const*, char const*)->ValueWithComment<std::string>;
}
```
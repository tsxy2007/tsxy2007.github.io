---
title: C++模板-模板的多态性
date: 2022-08-11 10:16:18
tags:
---

多态是一种用单个统一符号将多种特定行为关联起来的能力。多态也是面向对象编程范式的基石，在C++中它主要由继承和虚函数实现。由于这一机制主要在运行期间起作用，因此我们称之为动态多态。C++模板也允许我们用单一符号将不同特定行为关联起来，不过该关联发生在编译期间，我们称之为静态多态。
## 一: 动态多态
C++ 最开始只支持通过继承和虚函数实现的多态。在此情况下，多态的设计主要体现在从一些相关的对象类型中提炼出一组统一的功能，然后将它们声明成一个基类的虚函数接口。代码如下：
```
namespace _18_1_
{
    struct Coord
    {
        int Type = 0;
    };

    class GeoObj
    {
    public:
        virtual void draw() const = 0;
        virtual Coord center_of_gravity() const = 0;
        virtual ~GeoObj() = default;
    };

    class Circle : public GeoObj
    {
    public:

        virtual void draw() const override
        {
            std::cout << "draw() Circle" << std::endl;
        }
        virtual Coord center_of_gravity() const override
        {
            return {1};
        }
    };

    class Line : public GeoObj
    {
    public:

        virtual void draw() const override
        {
            std::cout << "draw() Line" << std::endl;
        }
        virtual Coord center_of_gravity() const override
        {
            return { 2 };
        }
    };

    void myDraw(GeoObj const& obj)
    {
        obj.draw();
    }

    Coord distance(GeoObj const& x1, GeoObj const& x2)
    {
        Coord c1 = x1.center_of_gravity();
        Coord c2 = x2.center_of_gravity();
        return { c2.Type - c1.Type };
    }

    void drawElems(std::vector<GeoObj*> const& elems)
    {
        for (size_t i = 0; i < elems.size(); i++)
        {
            elems[i]->draw();
        }
    }
}
```
关键的多态接口时函数draw()和center_of_gravity()。都是虚成员函数。上述例子展现了这两个虚函数的用法。后边的这几个函数使用的都是公共基类GeoObj。结果就是，在编译期间并不能知道将要真正调用的函数，会在运行期间基于各个对象的完整类型来决定将要调用的函数。

## 二: 静态多态
模板也可以用来实现多态。不同的是，它们不依赖于对基类中公共行为的分解。取而代之的是，这一“共性”隐式的要求不同类型，必须支持使用了相同的语法操作。在定义上具体的class之间彼此相互独立。代码如下：
```
namespace _18_2_
{
    struct Coord
    {
        int Type = 0;
    };

    class Circle 
    {
    public:

        virtual void draw() const 
        {
            std::cout << "draw() Circle" << std::endl;
        }
        virtual Coord center_of_gravity() const 
        {
            return { 1 };
        }
    };

    class Line 
    {
    public:

        virtual void draw() const 
        {
            std::cout << "draw() Line" << std::endl;
        }
        virtual Coord center_of_gravity() const 
        {
            return { 2 };
        }
    };

    template <typename T>
    void myDraw(T const& obj)
    {
        obj.draw();
    }

    template<typename T>
    void drawElems(std::vector<T> const& elems)
    {
        for (size_t i = 0; i < elems.size(); i++)
        {
            elems[i].draw();
        }
    }
}
```
使用这种方式，我们将不再能够透明地处理异质容器。这也正是 static 多态中的 static 部分带来的限制：所有的类型必须在编译期可知。不过，我们可以很容易的为不同的集合对 象类型引入不同的集合。这样就不再要求集合的元素必须是指针类型，这对程序性能和类型 安全都会有帮助。

## 三: 动态多态和静态多态
Static 和 dynamic 多态提供了对不同 C++编程术语的支持： 
* 通过继承实现的多态是有界的（bounded）和动态的（dynamic）： 
  1. 有界的意思是，在设计公共基类的时候，参与到多态行为中的类型的相关接口就已 经确定（该概念的其它一些术语是侵入的（invasive 和 intrusive））。    
  2. 动态的意思是，接口的绑定是在运行期间执行的。    
* 通过模板实现的多态是无界的（unbounded）和静态的（static）： 
    1. 无界的意思是，参与到多态行为中的类型的相关接口是不可预先确定的（该概念的 其它一些术语是非侵入的（noninvasive 和 nonintrusive）） 
    2. 静态的意思是，接口的绑定是在编译期间执行的

### C++中的动态多态有如下优点： 
* 可以很优雅的处理异质集合。 
* 可执行文件的大小可能会比较小（因为它只需要一个多态函数，不像静态多态那样，需 要为不同的类型进行各自的实例化）。 
*  代码可以被完整的编译；因此没有必须要被公开的代码（在发布模板库时通常需要发布 模板的源代码实现）。 
### C++中 static 多态的优点： 
*   内置类型的集合可以被很容易的实现。更通俗地说，接口的公共性不需要通过公共基类 实现。 
*   产生的代码可能会更快（因为不需要通过指针进行重定向，先验的（priori）非虚函数 通常也更容易被 inline）。 
*   即使某个具体类型只提供了部分的接口，也可以用于静态多态，只要不会用到那些没有 被实现的接口即可。

## 四: 使用concepts
使用模板的静态多态的一个争议是，接口的绑定是通过实例化相应的模板执行的。也就是说没有可供编程的公共接口或者公共class。取而代之的是，如果所有实例化的代码都是有效的，纳闷对模板的任何使用也都是有效的。否则，就会导致难以理解的错误信息，或者产生了有效代码却导致意料之外的行为。   
基于这一原因，C++语言的设计者们一直在致力于实现一种能够为模板参数显式地提供（或 者是检查）接口的能力。在 C++中这一接口被称为 concept。它代表了为了能够成功的实例 化模板，模板参数必须要满足的一组约束条件。代码如下：
```
namespace _18_4_
{
    struct Coord
    {
        int Type = 0;
    };

    template<typename T>
    concept GeoObj = requires(T x)
    {
        { x.draw() }-> std::convertible_to<void>;
        { x.center_of_gravity() } ->std::convertible_to<Coord>;
    };

    template<typename T>
    requires GeoObj<T>
        void myDraw(T const& obj)
    {
        obj.draw();
    }


    class Circle
    {
    public:

        virtual void draw() const
        {
            std::cout << "draw() Circle" << std::endl;
        }
        virtual Coord center_of_gravity() const
        {
            return { 1 };
        }
    };

    class Line
    {
    public:

        virtual void draw() const
        {
            std::cout << "draw() Line" << std::endl;
        }
        virtual Coord center_of_gravity() const
        {
            return { 2 };
        }
    };

    class Test
    {
    public:

        virtual void draw() const
        {
            std::cout << "draw() Test" << std::endl;
        }

        virtual Coord center_of_gravity() const
        {
            return { 3 };
        }
    private:

    };

}
``` 
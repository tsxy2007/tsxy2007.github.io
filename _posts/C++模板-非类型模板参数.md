---
title: C++模板-非类型模板参数
date: 2022-07-06 18:58:00
tags: c++模板
category:
---
## 一: 非类型参数限制
模板参数不一定非得是某种具体的类型，也可以是常规数值
### 限制
1. 整形常量（包含枚举）
2. 指向objects/functions/members的指针
3. objects或者functions的左值引用，
4. std::nullptr_t   

<b>浮点型数值或者class类型的对象不能作为非类型模板参数使用</b>
### 其他限制  
1. 在 C++11 中，对象必须要有外部链接。
2. 在 C++14 中，对象必须是外部链接或者内部链接。
3. 而对 C++17，即使对象没有链接属性也是有效的。

### 无效表达式
非类型模板参数可以是任何编辑器表达式。
```
template<int I,bool B>
class C;

C<sizeof(int) + 4,sizeof(int) == 4> c;
C<42, sizeof(int) > 4> c; // 错误需要在比大小添加括号
//正确如下
C<42, (sizeof(int) > 4)> c; 
```

## 二: 类模板的非类型参数
代码如下：

```

namespace _3_1_
{
    template<typename T, std::size_t Maxsize>
    class Stack
    {
    public:
        Stack();
        void push(T const& elem);
        void pop();
        T const& top()const;
        bool empty()const
        {
            return numElems == 0;
        }

        std::size_t size()const
        {
            return numElems;
        }
    private:
        std::array<T, Maxsize> elems;
        std::size_t numElems;
    };

    template<typename T, std::size_t Maxsize>
    Stack<T, Maxsize>::Stack() : 
        numElems(0)
    {

    }

    template<typename T,std::size_t Maxsize>
    void Stack<T, Maxsize>::push(T const& elem)
    {
        assert(numElems < Maxsize);
        elems[numElems] = elem;
        ++numElems;
    }

    template<typename T, std::size_t Maxsize>
    void Stack<T, Maxsize>::pop()
    {
        assert(!elems.empty());
        --numElems;
    }

    template<typename T, std::size_t Maxsize>
    T const& Stack<T, Maxsize>::top() const
    {
        assert(!elems.empty());
        return elems[numElems - 1];
    }
}
```
---
## 三: 函数模板的非类型参数
代码如下：
```
namespace _3_2_
{
    template<int Val,typename T>
    T addValue(T x)
    {
        return x + Val;
    }
}

int main()
{
      std::array<int, 10> bArray = { 1,2,3,4,5,6,7,8,9,10 };
    std::array<int, 10> aArray;
    std::transform(bArray.begin(), bArray.end(), aArray.begin(), _3_2_::addValue<10, int>);
    for (size_t i = 0; i < aArray.size(); i++)
    {
        std::cout << aArray[i] << std::endl;
    }
}
```
## 四: 用atuo作为非模板类型参数的类型
从 C++17 开始，可以不指定非类型模板参数的具体类型（代之以 auto），从而使其可以用 于任意有效的非类型模板参数的类型。通过这一特性，可以定义如下更为泛化的大小固定的 Stack 类：
```
namespace _3_3_
{
    template<typename T, auto Maxsize>
    class Stack
    {
    public :
        using size_type = decltype(Maxsize); // 推断非类型参数的类型
    private:
        std::array<T, Maxsize> elems;
        size_type numElems;
    public:
        Stack();
        void push(T const& elem);
        void pop();
        T const& top()const;
        bool empty()const
        {
            return numElems == 0;
        }
        size_type size()const
        {
            return numElems;
        }
    };

    template<typename T,auto Maxsize>
    Stack<T,Maxsize>::Stack()
        :numElems(0)
    {}

    template<typename T, auto Maxsize>
    void Stack<T, Maxsize>::push(T const& elem)
    {
        assert(numElems < Maxsize);
        elems[numElems] = elem;
        ++numElems;
    }

    template<typename T, auto Maxsize>
    void Stack<T, Maxsize>::pop()
    {
        assert(!elems.empty());
        --numElems;
    }

    template<typename T, auto Maxsize>
    T const& Stack<T, Maxsize>::top()const
    {
        assert(!elems.empty());
        return elems[numElems - 1];
    }
}

```
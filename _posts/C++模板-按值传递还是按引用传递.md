---
title: C++模板-按值传递还是按引用传递
date: 2022-07-23 18:40:20
tags:
---
从c++11开始引入移动语义(move semantics),也就是说多了一种按引用传递的方式：  
1. X const &(const 左值引用)  
   参数引用了被传递的对象，并且参数不能被更改。 
2. X &（非 const 左值引用）  
   参数引用了被传递的对象，但是参数可以被更改。   
3. X &&（右值引用）   
   参数通过移动语义引用了被传递的对象，并且参数值可以被更改或者被“窃取”。

一版在函数模板中应该优先使用按值传递，除非遇到以下情况：

* 对象不允许copy。
* 参数被用于返回数据。
* 参数以及其所有属性需要被模板转发到别的地方。
* 可以获得明显的性能提升。

## 一: 按值传递
当按值传递参数的时候，原则上所有的参数都会被拷贝，因此每个参数都会是被传递实参的一个拷贝。对于class对象，参数会通过class的拷贝构造函数来做初始化。  
调用拷贝构造函数的成本可能很高，但是有很多中方法可以避免按值传递的高昂成本：事实上我们可以通过移动语义来优化掉对象的拷贝。比如如下简单的函数模板：
```
namespace _7_1_
{
    template<typename T>
    void printV(T arg)
    {

    }
}
```
绝大部分情况arg都会变成参数的一份拷贝。但是并不是所有情况都会调用拷贝构造函数如下代码：
```
std::string returnString(); 
std::string s = "hi"; 
printV(s); //copy constructor 
printV(std::string("hi")); //copying usuallyoptimized away (if not, move constructor) 
printV(returnString()); // copying usually optimized away (if not, move constructor) 
printV(std::move(s)); // move constructor
```
咱们逐一看一下上面的4个调用：c++

* 第一个 : 咱们传递了一个lvalue，这会使用std::string的copy constructor。
* 第二个，第三个函数：被传递的参数是纯右值（prvalue，pure right value，临时对象或者某个 函数的返回值），此时编译器会优化参数传递，使得拷贝构造函数不会被调用。 从 C++17 开始，C++标准要求这一优化方案必须被实现。在 C++17 之前，如果编译器没有优 化掉这一类拷贝，它至少应该先尝试使用移动语义，这通常也会使拷贝成本变得比较低廉。
* 第四个 : 传递的是xvalue(一个使用过std::move后的对象)，这会调用move constructor。

[C++17的新特性：Mandatory Copy Elision or Passing Unmaterialized Objects](www.javashuo.com/link?url=http://www.cplusplus2017.info/c17-guaranted-copy-elision/)
### 按值传递会导致类型退化（decay）
关于按值传递，还有一个必须被讲到的特性：当按值传递参数时：
* 参数类型会退化（decay）。 
* 裸数组会退化成指针。
* const 和 volatile 等限制符会被删除

## 二: 按引用传递
按引用传递参数不会拷贝对象，而且传递参数时也不会造成类型退化。不过并不是所有情况下都能使用按引用传递。
### 1.按const引用传递
在传递非临时对象作为参数时，可以使用const引用传递代码如下：
```
    template<typename T>
    void printR(T const& args)
    {

    }

   int main()
   {
    std::string s = "Hi";
    int i = 3;
    printR(s);
    printR(i);
   } 
```
基本类型（int，float...）按引用传递变量，不会提高性能！这是因为在底层实现上，按引用传递还是通过传递参数的地址实现的。不过按地址 传递可能会使编译器在编译调用者的代码时有一些困惑：被调用者会怎么处理这个地址？理 论上被调用者可以随意更改该地址指向的内容。这样编译器就要假设在这次调用之后，所有 缓存在寄存器中的值可能都会变为无效。而重新载入这些变量的值可能会很耗时（可能比拷 贝对象的成本高很多）。你或许会问在按 const 引用传递参数时：为什么编译器不能推断出 被调用者不会改变参数的值？不幸的是，确实不能，因为调用者可能会通过它自己的非 const 引用修改被引用对象的值（这个解释太好，另一种情况是被调用者可以通过 const_cast 移除 参数中的 const）。
#### 按引用传递不会类型退化
* 参数类型不会退化（decay）。 
* 裸数组不会退化成指针。
* const 和 volatile 等限制符不会被删除
### 2.按非const引用传递
代码如下：
```
template<typename T>
    void printR(T& args)
    {
       
    }
```
如果想通过调用参数来返回变量值（比如修改被传递变量的值），就需要使用非 const 引用 （要么就使用指针）。同样这时候也不会拷贝被传递的参数。被调用的函数模板可以直接访 问被传递的参数。
```
int main()
{
   
   using namespace _7_2_1_;
   std::string s = "hi";
   std::string returnString1();
   
   printR(s); // 左值 模板里可以被修改
   printR(std::string("hi"));// 不允许临时变量prvalue:不具名且可被移动
   printR(returnString());// 不允许临时变量prvalue:不具名且可被移动
   printR(std::move(s));// 不允许xvalue
   return 0;
}
```
如果此时传递的参数时const的，arg的类型就有可能被推断为const引用，也就是说这时可以传递一个右值作为参数，但是模板所期望的参数确实左值。代码如下：
```
   const std::string& s1 = "";
   printR(s1);
   printR(std::move(s1));
   printR(returnConstString());
   printR("hi");
```
如果想要禁止非const引用传递const对象，有三种选择
* 使用static_assert触发一个编译期错误代码如下：
```
 template<typename T>
    void printR(T& args)
    {
        static_assert(!std::is_const<T>::value, "out parameter of foo<T>(T&) is const");
        }
    }
```
* 通过使用std::enable_if<>禁用该情况下模板
```
 template<typename T,typename = std::enable_if_t<!std::is_const_v<T>>>
    void printR(T& args)
    {
      
    }
```
* 通过concepts来禁用该模板
```
  template<typename T>
    requires (!std::is_const_v<T>)
    void outR(T& args)
    {
        
    }
```

### 3.按转发引用传递参数
使用引用调用（call-by-reference）的一个原因是可以对参数进行完美转发。它有自己的规则
```
template<typename T>
    void passR(T&& arg)
    {
    }


    std::string s = "hi"; 
    passR(s); // OK: T deduced as std::string& (also the type of arg) 
    passR(std::string("hi")); // OK: T deduced as std::string, arg is std::string&& 
    passR(returnString()); // OK: T deduced as std::string, arg is std::string&& 
    passR(std::move(s)); // OK: T deduced as std::string, arg is std::string&& 
    passR(arr); // OK: T deduced as int(&)[4] (also the type of arg)
```
## 三: 使用std::ref()和std::cref()限于模板
从c++11开始，可以让调用者自行决定向函数模板传递参数的方式。如果模板函数被声明成按值传递的，调用者可以使用定义在头文件<functional>的std::ref()和std::cref()将参数按引用传递给函数模板。
```
template<typename T>
void printT(T arg)
{

}
int main()
{
   std::string s = "hello";
   printT(s);
   printT(std::cref(s));
}
```
std::cref()并没有改变函数模板内部处理参数的方式。只是使用了一个技巧：它用一个行为和引用类似的对象对参数进行封装（std::reference_wrapper）.注意：编译器必须知道需要将std::reference_wrapper对象转换为原始参数类型，才会进行隐式转换。因此std::ref()和std::cref()通常只有在通过泛型代码传递对象时才能正常工作，如果直接使用传递进来的对象，有可能出现错误如下：
```
template<typename T>
void printT(T arg)
{
   std::cout<<arg<<std::endl; //报错，std::cref没有实现输出函数。
}

int main()
{
   std::string s = "hello";
   printT(std::cref(s));
}
```
正确应将模板函数作为传递参数的函数使用如下：
```
 void printString(std::string const& s)
    {
        std::cout << s << std::endl;
    }

template<typename T>
void printT(T arg)
{
   printString(arg);
}

int main()
{
   std::string s = "hello";
   printT(std::cref(s));
}
```
## 四: 处理字符串常量和裸数组
到目前为止，我们看到了将字符串常量和裸数组用作模板参数时的不同效果： 
* 按值传递时参数类型会 decay，参数类型会退化成指向其元素类型的指针。 
* 按引用传递是参数类型不会 decay，参数类型是指向数组的引用。  
两种情况各有其优缺点。将数组退化成指针，就不能区分它是指向对象的指针还是一个被传 递进来的数组。另一方面，如果传递进来的是字符串常量，那么类型不退化的话就会带来问 题，因为不同长度的字符串的类型是不同的。
### 关于字符串常量和裸数组的特殊实现
* 可以将模板定义成只接受数组作为参数：
```
template<typename T, std::size_t L1, std::size_t L2> 
void foo(T (&arg1)[L1], T (&arg2)[L2]) 
{
}
```
* 可以使用类型萃取来检测参数是不是一个数组
```
template<typename T, typename = std::enable_if_t<std::is_array_v<T>>> void foo (T&& arg1, T&& arg2) 
{
}
```
## 五: 处理返回值

返回值也可以被按引用或者按值返回。但是按引用返回可能会带来一些麻烦，因为它所引用 的对象不能被很好的控制。不过在日常编程中，也有一些情况更倾向于按引用返回： 
* 返回容器或者字符串中的元素（比如通过[]运算符或者 front()方法访问元素） 
* 允许修改类对象的成员 
* 为链式调用返回一个对象（比如>>和<<运算符以及赋值运算符）  

<b>需要注意防止返回引用，有两种方式</b>
* 用类型萃取std::remove_reference<>将T转为非引用类型
```
template<typename T>
typename std::remove_reference<T>::Type retV(T p)
{
   return T{};
}
```
* 将返回类型声明为 auto，从而让编译器去推断返回类型，这是因为 auto 也会导致类型 退化：
```
template<typename T> 
auto retV(T p) // by-value return type deduced by compiler 
{
    return T{…}; // always returns by value
}
```
## 六: 关于模板参数声明的推荐方法

* 将参数声明成按值传递： 这一方法很简单，它会对字符串常量和裸数组的类型进行退化，但是对比较大的对象可能 会受影响性能。在这种情况下，调用者仍然可以通过 std::cref()和 std::ref()按引用传递参数， 但是要确保这一用法是有效的。 
* 将参数声明成按引用传递： 对于比较大的对象这一方法能够提供比较好的性能。尤其是在下面几种情况下：  

   - 将已经存在的对象（lvalue）按照左值引用传递， 
   - 将临时对象（prvalue）或者被 std::move()转换为 可移动的对象（xvalue）按右值引 用传递， 
   - 或者是将以上几种类型的对象按照转发引用传递。
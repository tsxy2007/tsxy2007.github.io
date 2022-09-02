---
title: C++-模板-萃取的实现(二)
date: 2022-09-02 08:52:18
tags:
---
传统上我们在c和c++里定义的函数可以被更明确的称为值函数，对于模板我们还可以定义类型函数：它们接收一些类型作为参数并返回类型或者常量作为结果。   
我们实现一个简单的类型函数例子，这里用到一个简单的内置函数sizeof，给定类型返回类型大小。代码如下：
```
template<typename T>
struct TypeSize
{
	static std::size_t const value = sizeof(T);
};
```
虽然没什么用，但是我们能明白类型函数定义的基本方式。
## 一：元素类型(Element Type)
假设我们有很多容器模板，比如std::vector<>和std::list<>，也可以包含内置数组。我们希望得到这样一个类型函数，当给的一个容器时，它可以返回相应的元素类型。我可以通过偏特化实现，代码如下：
```

namespace _19_3_1_
{
	// 元素类型

	template<typename T>
	struct ElementT;

	template<typename T>
	struct ElementT<std::vector<T>>
	{
		using Type = T;
	};

	template<typename T>
	struct ElementT<std::list<T>>
	{
		using Type = T;
	};

	template<typename T, std::size_t N>
	struct ElementT<T[N]>
	{
		using Type = T;
	};

	template<typename T>
	struct ElementT<T[]>
	{
		using Type = T;
	};

	template<typename T>
	void printElementType(T const& c)
	{
		std::cout << "Container of " << typeid(typename ElementT<T>::Type).name() << " elements.\n";
	}
}

```
偏特化的使用使得我们可以在容器不知道具体类型函数存在的情况下去实现类型函数。但是在某些情况下，类型函数是和其所使用的类型一起被设计的，此时相关实现可以被简化，比如某些容器类型定义了value_type成员类型，我们就可以有如下实现:
```
template<typename C>
struct ElementT
{
	using Type = typename C::value_type;
};
```
这个实现可以是默认实现，他不会排除那些针对没有定义成员类型value_type的容器的偏特化实现。
类型函数作用体现在哪里呢？它允许我们根据容器类型参数化一个模板，但是又不需要提供代表了元素类型和其他特性的参数。比如，相比于使用：
```
template<typename T, typename C>
T sumOfElements(C const& c);
```
这一需要显式指定元素类型的模板我们可以定义这样一个模板：
```
template<typename C>
typename ElementT<C>::Type sumOfElements(C const& c);
```
其元素类型是通过类型函数得到的。
在上述情况下，ElementT被称为萃取类，因为它被用来访问一个已有容器类型的萃取。为了方便，我们可以为这个类型函数创建一个别名模板。
```
template<typename T>
using ElementType = typename ElementT<T>::Type;
```
## 二： 转换萃取
除了可以被用来访问主参类型的某些特性，萃取还可以被用来做类型转换，比如为某个类型添加或者一处引用，const以及volatile限制符。
### 删除引用
代码如下：
```
namespace _19_3_2_1_
{
	//删除引用

	template<typename T>
	struct RemoveReferenceT
	{
		using Type = T;
	};

	template<typename T>
	struct RemoveReferenceT<T&>
	{
		using Type = T;
	};

	template<typename T>
	struct RemoveReferenceT<T&&>
	{
		using Type = T;
	};

	template<typename T>
	using RemoveReference = typename RemoveReferenceT<T>::Type;
}
```
### 添加引用
```
// 添加引用
	template<typename T>
	struct AddLValueReferenceT
	{
		using Type = T&;
	};

	template<>
	struct AddLValueReferenceT<void>
	{
		using Type = void;
	};

	template<>
	struct AddLValueReferenceT<void const>
	{
		using Type = void const;
	};

	template<>
	struct AddLValueReferenceT<void volatile>
	{
		using Type = void volatile;
	};

	template<>
	struct AddLValueReferenceT<void const volatile>
	{
		using Type = void const volatile;
	};

	template<typename T>
	using AddLValueReference = typename AddLValueReferenceT<T>::Type;


	template<typename T>
	struct AddRValueReferenceT
	{
		using Type = T&&;
	};

	template<typename T>
	using AddRValueReference = typename AddRValueReferenceT<T>::Type;
```

引用折叠的规则在这依然适用。如果我们只实现，而又不对它们进行偏特化的话我们可以简化成下面：
```
	template<typename T>
	using AddLValueReferenceT1 = T&;

	template<typename T>
	using AddRValueReferenceT1 = T&&;
```
### 移除限制符
```
template<typename T>
	struct RemoveConstT
	{
		using Type = T;
	};

	template<typename T>
	struct RemoveConstT<T const>
	{
		using Type = T;
	};

	template<typename T>
	using RemoveConst = typename RemoveConstT<T>::Type;

	template<typename T>
	struct RemoveVolatileT
	{
		using Type = T;
	};

	template<typename T>
	struct RemoveVolatileT<T volatile>
	{
		using Type = T;
	};

	template<typename T>
	using RemoveVolatile = typename RemoveVolatileT<T>::Type;

	template<typename T>
	struct RemoveCVT : RemoveConstT<typename RemoveVolatileT<T>::Type>
	{

	};
```

RemoveCVT 中有两个需要注意的地方。
1. 同时使用了RemoveConstT和RemoveVolatileT，首先移除类型中volatile，然后得到的类型传递给RemoveConstT。
2. 没有定义自己的和RemoveConstT中Type类似的成员，而是通过使用元函数转发，从RemoveConstT继承了Type成员。

### 退化
模仿c语言的退化机制，删除相应顶层const，volatile以及引用限制符。首先我们先实现一个按值传递的效果，他会打印经过编译器退化之后的参数类型：
```
template<typename T>
void f(T)
{
}
template<typename A>
void printParameterType(void (*)(A))
{
	std::cout << "Parameter type: " << typeid(A).name() << std::endl;
	std::cout << "- is int: " << std::is_same<A, int>::value << std::endl;
	std::cout << "- is const: " << std::is_const<A>::value << std::endl;
	std::cout << "- is pointer: " << std::is_pointer<A>::value << std::endl;
}

int main() 
{ 
	printParameterType(&f<int>); 
	printParameterType(&f<int const>); 
	printParameterType(&f<int[7]>); 
	printParameterType(&f<int(int)>); 
}
```
我们实现一个与之功能类似的萃取。为了和标准库std::decay保持匹配，我们称之为decayT，它的实现结合上文中的多种技术。首先我们对非数组、非函数的情 况进行定义，该情况只需要删除 const 和 volatile 限制符即可：
```
template<typename T>
	struct DecayT : _19_3_2_3_::RemoveCVT<T>
	{

	};

	template<typename T>
	struct DecayT<T[]>
	{
		using Type = T*;
	};

	template<typename T, std::size_t N>
	struct DecayT<T[N]>
	{
		using Type = T*;
	};

	template<typename R, typename... Args>
	struct DecayT<R(Args...)>
	{
		using Type = R(*)(Args...);
	};

	template<typename R, typename... Args>
	struct DecayT<R(Args..., ...)>
	{
		using Type = R(*)(Args..., ...);
	};

	template<typename T>
	using Decay = typename DecayT<T>::Type;


template<typename T>
	void printDecayedType()
	{
		using A = typename DecayT<T>::Type;
		using B = Decay<T>;

		std::cout << "Parameter type: " << typeid(A).name() << std::endl;
		std::cout << "- is int: " << std::is_same<A, int>::value << std::endl;
		std::cout << "- is const: " << std::is_const<A>::value << std::endl;
		std::cout << "- is pointer: " << std::is_pointer_v<A> << std::endl;
	}
```

## 三：预测型萃取
到目前为止，我们学习并开发了适用于单个类型的类型函数：给定一个类型，产生另一个相关的类型或者常量。但是通常而言，也可以设计基于多个参数的类型函数。这同样会引出另外一种特殊的类型萃取：类型预测（产生一个bool数值的类型函数）
### IsSameT:
IsSameT将判定两个类型是否相同：
```
namespace _19_3_3_1_
{
	template<typename T1, typename T2>
	struct isSameT
	{
		static constexpr bool value = false;
	};

	template<typename T>
	struct isSameT<T, T>
	{
		static constexpr bool value = true;
	};


	// 对于产生一个常量的萃取，我们没法为之定义一个别名模板，但是可以为之定义一个扮演相同角色的constexpr的变量模板
	template<typename T1, typename T2>
	constexpr bool isSame = isSameT<T1, T2>::value;
}
```
主模板传递进来两个参数是不同，value是false。使用偏特化，当遇到传递过来两个相同参数的情况，value是true。

### true_type和false_type
以上代码我们可以提取一下，我们把value提取出来单独做个公共接口：
```
namespace _19_3_3_2_
{
	template<bool val>
	struct BoolConstant
	{
		using Type = BoolConstant<val>;
		static constexpr bool value = val;
	};
	using TrueType = BoolConstant<true>;
	using FalseType = BoolConstant<false>;
}
```
就可以基于两个类型是否匹配，让相应的IsSameT分别继承自TrueType和FalseType：
```
namespace _19_3_3_2_
{
	template<typename T1, typename T2>
	struct IsSame : FalseType {};

	template<typename T>
	struct IsSame<T, T> : TrueType {};

	template<typename T1, typename T2>
	using IsSameT = typename IsSame<T1, T2>::Type;

	template<typename T1, typename T2>
	constexpr bool IsSameV = IsSame<T1, T2>::value;
}
```
## 四：返回结果类型萃取
先上一个简单的例子：
```
template<typename T>
Array<T> sum(Array<T> const& ,Array<T> const&);
```
这个有个问题比如char型数值求和，就会出现问题。我们需要处理下返回值类型。我们可以自定义PlusResultType<T1,T2>返回值类型模板代码如下：
```
	// 返回结果类型萃取
	template<typename T1, typename T2>
	struct PlusResultT
	{
		using Type = decltype(T1() + T2());
	};
```
我们可以通过decltype来计算表达式T1()+T2()的类型，将决定这一艰巨的工作交给编译器。但是有个问题，decltype保留了过多的信息。我们需要将这些多余类型信息去除代码如下：
```
template<typename T1, typename T2>
//去除多余类型信息;
	std::vector<std::remove_cv_t<std::remove_reference_t< PlusResult<T1, T2> > > >
		operator+(std::vector<T1> const& t1, std::vector<T2> const& t2)
	{
		std::vector<std::remove_cv_t<std::remove_reference_t< PlusResult<T1, T2> > > > arr;
		for (size_t i = 0; i < t1.size(); i++)
		{

			arr.push_back((t1[i] + t2[i]) / 2);
		}
		return arr;
	}
```
还是有个问题。假如我们代码删除默认构造函数，这个可能出现问题。我们将在下节解决这个问题；
### decval
我们可以在不需要构造函数的情况下计算+表达式的值，方法是使用一个可以为一个给定类型T生成数值的函数。代码如下：
```
namespace std 
{ 
	template<typename T> add_rvalue_reference_t<T> declval() noexcept; 
}
```

该函数模板被故意设计成未定义的状态，因为我们只希望它被用于 decltype，sizeof 或者其 它不需要相关定义的上下文中。它有两个很有意思的属性： 
* 对于可引用的类型，其返回类型总是相关类型的右值引用，这能够使 declval 适用于那 些不能够正常从函数返回的类型，比如抽象类的类型（包含纯虚函数的类型）或者数组 类型。因此当被用作表达式时，从类型 T 到 T&&的转换对 declval<T>()的行为是没有影 响的：其结果都是右值（如果 T 是对象类型的话），对于右值引用，其结果之所以不会 变是因为存在引用塌缩。 
*  在 noexcept 异常规则中提到，一个表达式不会因为使用了 declval 而被认成是会抛出异 常的。当 declval 被用在 noexcept 运算符上下文中时，这一特性会很有帮助   
  
修改之后的代码如下：
```
template<typename T1, typename T2>
	struct PlusResultT
	{
		using Type = decltype(std::declval<T1>() + std::declval<T2>());
	};
```
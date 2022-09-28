---
title: C++-模板-萃取的实现(五)
date: 2022-09-26 14:38:18
tags:
---
## if-Then-Else
我们可以再编译器实现一个if then else表达式,它接受一个bool型的模板参数，并根据该参数从另外两个类型参数中间做选择：
```
template<bool COND, typename TrueType, typename FalseType>
struct  ifThenElseT
{
	using Type = TrueType;
};

template<typename TrueType, typename FalseType>
struct  ifThenElseT<false, TrueType, FalseType>
{
	using Type = FalseType;
};

template<bool COND, typename TrueType, typename FalseType>
using IfThenElse = typename ifThenElseT<COND, TrueType,FalseType>::Type;
```
需要注意的是和常规c++的if-then-else语句不同，在最终做选择之前，then和else分支中的模板参数都会被计算，因此两个分支中的代码都不能有问题，否则整个程序就会有问题。考虑一下例子，是它要求传递进来的类型是有 符号的整形，而且不能是 bool 类型；否则它将使用未定义行为的结果：
```
template<typename T>
	struct UnsignedT
	{
		using Type = IfThenElse<(std::is_integral_v<T> && !std::is_same_v<T, bool>), typename std::make_unsigned<T>::type, T >;
	};
```
这样简单的实现是不行的，因为在实例化UnsingedT<bool>的时候，行为依然是未定义的，编译期依然会试图从下面的代码中生成返回类型：
```
typename std::make_unsigned<T>::type
```
为了解决这一问题，我们需要再引入一层额外的间接层，从而让ifThenElse的参数本身用类型函数去封装结果：
```
namespace _19_7_1_
{
	template<bool COND, typename TrueType, typename FalseType>
	struct  ifThenElseT
	{
		using Type = TrueType;
	};

	template<typename TrueType, typename FalseType>
	struct  ifThenElseT<false, TrueType, FalseType>
	{
		using Type = FalseType;
	};

	template<bool COND, typename TrueType, typename FalseType>
	using IfThenElse = typename ifThenElseT<COND, TrueType, FalseType>::Type;

	template<typename T>
	struct MakeUnsignedT
	{
		using Type = typename std::make_unsigned<T>::type;
	};
	template<typename T>
	struct IdentityT
	{
		using Type = T;
	};

	template<typename T>
	struct UnsignedT
	{
		using Type = IfThenElse<(std::is_integral_v<T> && !std::is_same_v<T, bool>), MakeUnsignedT<T>, IdentityT<T> >::Type;
	};
}
```
在这一版 UnsignedT 的定义中，IfThenElse 的类型参数本身也都是类型函数的实例。只不过 在最终 IfThenElse 做出选择之前，类型函数不会真正被计算。而是由 IfThenElse 选择合适的 类型实例（MakeUnsignedT 或者 IdentityT）。最后由::Type 对被选择的类型函数实例进行计 算，并生成结果 Type。
此处强调的是，之所以这么做，是因为IfThenElse中未被选择的封装类型永远不会被完全实例化。下面的代码也不能正常工作：
```
namespace _19_7_1_
{
	template<bool COND, typename TrueType, typename FalseType>
	struct  ifThenElseT
	{
		using Type = TrueType;
	};

	template<typename TrueType, typename FalseType>
	struct  ifThenElseT<false, TrueType, FalseType>
	{
		using Type = FalseType;
	};

	template<bool COND, typename TrueType, typename FalseType>
	using IfThenElse = typename ifThenElseT<COND, TrueType, FalseType>::Type;

	template<typename T>
	struct MakeUnsignedT
	{
		using Type = typename std::make_unsigned<T>::type;
	};
	template<typename T>
	struct IdentityT
	{
		using Type = T;
	};

	template<typename T>
	struct UnsignedT
	{
		using Type = IfThenElse<(std::is_integral_v<T> && !std::is_same_v<T, bool>), MakeUnsignedT<T>, T>::Type;
	};
}
```
我们必须要延后对 MakeUnsignedT<T>使用::Type，也就是意味着，我们同样需要为 else 分支 中的 T 引入 IdentyT 辅助模板，并同样延后对其使用::Type。

## 探测不抛出异常的操作
我们可能偶尔会需要判断某一个操作会不会抛出异常。比如，在可能的情况下，移动构造函数应当被标记成noexcept的，意思是它不会抛出异常。但是，某一特定class的move constructor是否抛出异常，通常决定于其成员或者基类的移动构造函数会不会抛出异常。比如对于下面这个简单类模板(pair)的移动构造函数：
```
template<typename T1, typename T2> 
class Pair 
{ 
    T1 first; 
    T2 second; 
    public: Pair(Pair&& other) : 
    first(std::forward<T1>(other.first)), 
    second(std::forward<T2>(other.second)) 
    { 

    } 
};
```
当T1或者T2的移动操作会抛出异常时,Pair的移动构造函数也会抛出异常。如果有一个叫IsNothrowMoveConstructibleT的萃取,就可以在Pair的移动构造函数中通过使用noexcept将这一异常的依赖关系表达出来：
```
template<typename T1, typename T2>
	class Pair
	{
	public:
		Pair(Pair&& other)
			noexcept(IsNothrowMoveConstructibleT<T1>::value&& IsNothrowMoveConstructibleT<T2>::value)
			: first(std::forward<T1>(other.first)),
			second(std::forward<T2>(other.second)) { }

		Pair(const T1& _Val1, const T2& _Val2) : first(_Val1), second(_Val2) {}
	private:
		T1 first;
		T2 second;
	};
```
现在剩下的事情就是实现IsNothrowMoveConstructibleT萃取了。我们可以直接用noexcept运算符实现这一萃取，这样就可以判断一个表达式是否进行nothrow修饰了：
```
template<typename T, typename = std::void_t<>>
	struct IsNothrowMoveConstructibleT : std::false_type
	{ };
	template<typename T>
	struct IsNothrowMoveConstructibleT<T, std::void_t<decltype(T(std::declval<T>()))>>
		: std::bool_constant<noexcept(T(std::declval<T>()))>
	{

	};
```
这里使用了运算符版本的noexept，它会判断一个表达式是否会抛出异常。由于其结果是bool型的，我们可以直接将它用于std::bool_constant<>基类的定义。

## 别名模板和萃取
别名模板为降低代码繁琐性提供了一种方法。相比于将类型萃取表达成一个包含了Type类型成员的类模板，我们可以直接使用别名模板。
但是别名模板用于类型萃取也有一些缺点：
1. 别名模板不能够被进行特化（在第 16.3 节有过提及），但是由于很多编写萃取的技术 都依赖于特化，别名模板最终可能还是需要被重新导向到类模板。 
2. 有些萃取是需要由用户进行特化的，比如描述了一个求和运算符是否是可交换的萃取， 此时在很多使用都用到了别名模板的情况下，对类模板进行特换会很让人困惑。 
3. 对别名模板的使用最会让该类型被实例化（比如，底层类模板的特化），这样对于给定 类型我们就很难避免对其进行无意义的实例化（正如在第 19.7.1 节讨论的那样）(对最后一点的另外一种表述方式是，别名模板不可以和元函数转发一起使用).

## 变量模板和萃取
对于之前代码IsSameT模板可以简化为：
```
template<typename T1, typename T2>
constexpr bool IsSame = IsSameT<T1,T2>::value;
```
## 判断基础类型

我们先定义一个可以判断某个类型是不是基础类型的模板。默认情况下，我们认为类型不是基础类型，而对于基础类型，我们分别进行了特化：
```
namespace _19_8_1_
{
	template<typename T>
	struct IsFundaT : std::false_type { };

#define MK_FUNDA_TYPE(T) \
    template<> struct IsFundaT<T> : std::true_type{ };

	MK_FUNDA_TYPE(void);
	MK_FUNDA_TYPE(bool);
	MK_FUNDA_TYPE(char);
	MK_FUNDA_TYPE(signed char);
	MK_FUNDA_TYPE(unsigned char);
	MK_FUNDA_TYPE(wchar_t);
	MK_FUNDA_TYPE(char16_t);
	MK_FUNDA_TYPE(char32_t);
	MK_FUNDA_TYPE(signed short);
	MK_FUNDA_TYPE(unsigned short);
	MK_FUNDA_TYPE(signed int);
	MK_FUNDA_TYPE(unsigned int);
	MK_FUNDA_TYPE(signed long);
	MK_FUNDA_TYPE(unsigned long);
	MK_FUNDA_TYPE(signed long long);
	MK_FUNDA_TYPE(unsigned long long);
	MK_FUNDA_TYPE(float);
	MK_FUNDA_TYPE(double);
	MK_FUNDA_TYPE(long double);
	MK_FUNDA_TYPE(std::nullptr_t);
#undef MK_FUNDA_TYPE


	template<typename T>
	void Test(T const& it)
	{
		if (IsFundaT<T>::value)
		{
			std::cout << "T is a fundamental type" << std::endl;
		}
		else
		{
			std::cout << "T is not a fundamental type" << std::endl;
		}
	}
}
```
对于每一种基础类型，我们都进行了特化，因此IsFundaT<T>::value的结果也都会返回true。
## 判断复合类型
复合类型是由其他类型构建出来的类型。简单的复合类型包含指针类型，左值以及右值引用类型，指向成员的指针类型，和数组类型。它们是由一种或者两种底层类型组成的。在这一分类方法中，枚举类型同样被认为是复杂的符合类型，虽然它们 不是由多种底层类型构成的。简单的复合类型可以通过偏特化来区分。

### 指针
代码如下：
```
template<typename T>
	struct IsPointerT : std::false_type
	{

	};

	template<typename T>
	struct IsPointerT<T*> : std::true_type
	{
		using BaseT = T;
	};
```
### 引用
```
template<typename T>
	struct IsLValueReferenceT : std::false_type
	{

	};
	template<typename T>
	struct IsLValueReferenceT<T&> : std::true_type
	{
		using BaseT = T;
	};

	template<typename T>
	struct IsRValueReferenceT : std::false_type
	{

	};

	template<typename T>
	struct IsRValueReferenceT<T&&> : std::true_type
	{
		using BaseT = T;
	};
```

### 数组
```
template<typename T>
	struct IsArrayT : std::false_type
	{

	};

	template<typename T, std::size_t N>
	struct IsArrayT<T[N]> : std::true_type
	{
		using BaseT = T;
		static constexpr std::size_t size = N;
	};

	template<typename T>
	struct IsArrayT <T[]> : std::true_type
	{
		using BaseT = T;
		static constexpr std::size_t size = 0;
	};
```

### 指向成员的指针
```
template<typename T>
	struct IsPointerToMemberT : std::false_type
	{

	};

	template<typename T, typename C>
	struct IsPointerToMemberT<T C::*> : std::true_type
	{
		using MemberT = T;
		using ClassT = C;
	};
```

### 识别函数类型
```


namespace _19_8_3_
{

	template<typename... Elements>
	class Typelist
	{
	};

	template<typename T>
	struct IsFunctionT : std::false_type
	{

	};

	template<typename R, typename... Params>
	struct IsFunctionT <R(Params...)> : std::true_type
	{
		using RType = R;
		using ParamsT = Typelist<Params...>;
		static constexpr bool variadic = false;
	};

	template<typename R, typename ... Params>
	struct IsFunctionT<R(Params..., ...)> : std::true_type
	{
		using RType = R;
		using ParamsT = Typelist<Params...>;
		static constexpr bool variadic = true;
	};

	template<typename R, typename ... Params>
	struct IsFunctionT<R(Params...) const> : std::true_type
	{
		using RType = R;
		using ParamsT = Typelist<Params...>;
		static constexpr bool variadic = false;
	};

	template<typename R, typename ... Params>
	struct IsFunctionT<R(Params...) volatile> : std::true_type
	{
		using RType = R;
		using ParamsT = Typelist<Params...>;
		static constexpr bool variadic = true;
	};

	template<typename R, typename ... Params>
	struct IsFunctionT<R(Params..., ...) volatile> : std::true_type
	{
		using RType = R;
		using ParamsT = Typelist<Params...>;
		static constexpr bool variadic = true;
	};


	template<typename R, typename ... Params>
	struct IsFunctionT<R(Params..., ...) const volatile> : std::true_type
	{
		using RType = R;
		using ParamsT = Typelist<Params...>;
		static constexpr bool variadic = true;
	};

	template<typename R, typename ... Params>
	struct IsFunctionT<R(Params..., ...)&> : std::true_type
	{
		using RType = R;
		using ParamsT = Typelist<Params...>;
		static constexpr bool variadic = true;
	};

	template<typename R, typename ... Params>
	struct IsFunctionT<R(Params..., ...) const&> : std::true_type
	{
		using RType = R;
		using ParamsT = Typelist<Params...>;
		static constexpr bool variadic = true;
	};

	int Test(int i)
	{
		return 1;
	}
}
```

### 判断class类型
```
namespace _19_8_4_
{
	template<typename T,typename = std::void_t<>>
	struct IsClassT : std::false_type
	{

	};

	template<typename T>
	struct IsClassT < T,std::void_t<int T::*>> : std::true_type
	{

	};

	template<typename T>
	constexpr bool IsClass_v = IsClassT<T>::value;
}
```
### 识别枚举类型
```
template<typename T> 
struct IsEnumT
{
	static constexpr bool value = !IsFundaT<T>::value && !IsPointerT<T>::value && !IsReferenceT<T>::value && !IsArrayT<T>::value && !IsPointerToMemberT<T>::value && !IsFunctionT<T>::value && !IsClassT<T>::value;
};
```
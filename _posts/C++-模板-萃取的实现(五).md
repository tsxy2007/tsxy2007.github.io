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

## 萃取的便捷性
一个关于萃取的普遍不满是它们相对而言有些繁琐，因为对类型萃取的使用通需要提供一个::type尾缀，而且在依赖上下文中
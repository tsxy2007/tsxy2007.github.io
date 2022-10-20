---
title: C++模板-类型列表
date: 2022-10-20 08:33:51
tags:
---

类型列表是指一种代表了一组类型，并且可以被模板元编程操作的类型。我们为类型列表提供一下基本操作：
* 遍历
* 添加
* 删除
* 索引
* 查询
* 反转
* 类型转换
* 累加
* 插入排序

## 1.定义类型列表
```
template<typename... Elements>
class TypeList
{
};

template<>
class TypeList<>
{
};
```
## 2.取首类型
```
template<typename List>
class FrontT;

template<typename Head,typename... Tail>
class FrontT<TypeList<Head,Tail...>>
{
public:
	using Type = Head;
};

template<typename List>
using Front = typename FrontT<List>::Type;
```
## 3.删除首类型
```
template<typename List>
class FrontPopT;

template<typename Head,typename... Tail>
class FrontPopT<TypeList<Head, Tail...>>
{
public:
	using Type = TypeList<Tail...>;
};

template<typename List>
using FrontPop = typename FrontPopT<List>::Type;
```
## 4. 在列表头插入新类型
```
template<typename List,typename NewElement>
class FrontPushT;

template<typename... Elements,typename NewElement>
class FrontPushT<TypeList<Elements...>, NewElement>
{
public:
	using Type = TypeList<NewElement, Elements...>;
};

template<typename List, typename NewElement>
using FrontPush = typename FrontPushT<List,NewElement>::Type;
```
## 5.根据索引取类型
我們使用递归实例化这一技术。然后使用模板特化这一技术停止递归。
```
	template<typename List,unsigned N>
	class NthElementT : public NthElementT<FrontPop<List>, N - 1>
	{
	public:
	};

	template<typename List>
	class NthElementT<List, 0> : public FrontT<List>
	{

	};

	template <typename List, unsigned N>
	using NthElement = typename NthElementT<List, N>::Type;
```
咱们手动展开下这个程序的继承关系，假如咱们有TypeList如下：
```
TypeList<short, int, long, signed char, long long>
```
我们像查找第三个元素是什么类型.下面我们展开继承关系：
```
NthElementT<TypeList<short, int, long, signed char, long long>,2>
NthElementT<TypeList< int, long, signed char, long long>,1>
NthElementT<TypeList<long, signed char, long long>,0>
```
当N == 0 的时候我们的特化版本继承了FrontT<List>所以我们能拿到索引2的类型为long
## 6.寻找最佳类型匹配
有些类型列表算法会取查找类型列表中的数据。例如可能想要找出类型列表中最大的类型。这同样可以利用递归元程序实现:
```
	template<typename List>
	class LargestTypeT_old;

	template<typename List>
	class LargestTypeT_old
	{
	private:
		using First = Front<List>;
		using Rest = typename LargestTypeT_old<FrontPop<List>>::Type;
	public:
		using Type = _19_7_1_::IfThenElse<(sizeof(First) >= sizeof(Rest)), First, Rest>;
	};

	template<>
	class LargestTypeT_old<TypeList<>>
	{
	public:
		using Type = char;
	};

	template<typename List>
	using LargestType_old = typename LargestTypeT_old<List>::Type;
```
程序会在类型列表为空的时候停止递归。默认情况下我们将char用作哨兵类型来初始化该算法，因为任何类型都不会比char小。但是以上代码我们显式使用了空的类型列表TypeList<>。这样不太好，因为它可能妨碍我们以后的代码。为了解决这个问题，我们需要定义判空的函数。
```
template<typename List>
	class IsEmpty
	{
	public:
		static constexpr bool value = false;
	};

	template<>
	class IsEmpty<TypeList<>>
	{
	public:
		static constexpr bool value = true;
	};
```
结合IsEmpty我设计我们新的最大匹配模板元函数
```
template<typename List,bool Empty = IsEmpty<List>::value>
	class LargestTypeT;

	template<typename List>
	class LargestTypeT<List, false>
	{
	private:
		using Contender = Front<List>;
		using Best = typename LargestTypeT<FrontPop<List>>::Type;
	public:
		using Type = _19_7_1_::ifThenElseT<(sizeof(Contender) >= sizeof(Best)), Contender, Best>;
	};

	template<typename List>
	class LargestTypeT<List, true>
	{
	public:
		using Type = char;
	};

	template<typename List>
	using LargestType = typename LargestTypeT<List>::Type;
```
## 7.在类型列表末尾插入新类型
有俩种方案
<b>1.类似FrontPush直接插入末尾</b>
```
template<typename List,typename NewElement>
class PushBackT_Old;

template<typename... Elements,typename NewElement>
class PushBackT_Old<TypeList<Elements...>, NewElement>
{
public:
	using Type = TypeList<Elements..., NewElement>;
};

template<typename List,typename NewElement>
using PushBack_Old = typename PushBackT_Old<List, NewElement>::Type;
```
不过和实现 LargestType 的算法一样，可以只用 Front，PushFront，PopFront 和 IsEmpty 等基
础操作实现一个更通用的 PushBack 算法：
```
template<typename List,typename NewElement,bool = IsEmpty<List>::value>
	class PushBackRecT;

	template<typename List, typename NewElement>
	class PushBackRecT<List, NewElement, false>
	{
		using Head = Front<List>;
		using Tail = FrontPop<List>;
		using NewTail = typename PushBackRecT<Tail, NewElement>::Type;
	public:
		using Type = FrontPush<NewTail,Head>;
	};

	template<typename List,typename NewElement>
	class PushBackRecT<List, NewElement, true>
	{
	public:
		using Type = FrontPush<List, NewElement>;
	};

	template<typename List,typename NewElement>
	class PushBackT : public PushBackRecT<List,NewElement>
	{};

	template<typename List,typename NewElement>
	using PushBack = typename PushBackT<List, NewElement>::Type;
```
## 8.类型列表的反转
我们可以利用FrontPop和PushBackT这两个基础操作实现我们自己对类型列表的反转
```
template<typename List,bool = IsEmpty<List>::value>
	class ReverseT;

	template<typename List>
	using Reverse = typename ReverseT<List>::Type;

	template<typename List>
	class ReverseT<List, false> : public  PushBackT<Reverse<FrontPop<List>>, Front<List>>
	{

	};

	template<typename List>
	class ReverseT<List, true>
	{
	public:
		using Type = List;
	};
```
## 9.类型列表的转换
之前的操作都是取，添加，反转TypeList，现在我们要给每个元素添加一些其他操作。比如为每个元素添加const修饰符。首先我们首先定义一个元函数：
```
template<typename T>
struct AddConstT
{
	using Type = T const;
};

template<typename T>
using AddConst = typename AddConstT<T>::Type;
```
要定义我们自己的转换元函数，我们需要三个模板参数：List; 具体的转换函数(MetaFun); 终止条件(IsEmpty<List>::value);
```
//(1)
template<typename List,template<typename T> typename MetaFun,bool = IsEmpty<List>::value>
class TransformT;
//(2)
template<typename List, template<typename T> typename MetaFun>
class TransformT<List, MetaFun, false> : public FrontPushT<typename TransformT<FrontPop<List>, MetaFun>::Type, typename MetaFun<Front<List>>::Type>
{

};
//(3)
template<typename List, template<typename T> typename MetaFun>
class TransformT<List, MetaFun, true>
{
public:
	using Type = List;
};

template<typename List, template<typename T> typename MetaFun = AddConstT>
using Transform = typename TransformT<List, MetaFun>::Type;
```
根据我们之前学习的折叠表达式我们把(2)设置成另外一种版本
```
template<typename... Elements, template<typename T> typename MetaFun>
class TransformT<TypeList<Elements...>,MetaFun,false>
{
public:
	using Type = TypeList<typename MetaFun<Elements>::Type...>;
};
```
## 10.类型列表的累加
```
// 类型列表的累加
template<typename List, template<typename X, typename Y> typename TypeFun, typename I, bool = IsEmpty<List>::value>
class AccumulateT;

template<typename List, template<typename X, typename Y> typename TypeFun, typename I>
class AccumulateT<List, TypeFun, I, false> : public AccumulateT<FrontPop<List>, TypeFun, typename TypeFun<I, Front<List>>::Type>
	{};

template<typename List, template<typename X, typename Y> typename TypeFun, typename I>
class AccumulateT<List, TypeFun, I, true>
{
public:
	using Type = I;
};

template<typename List, template<typename X, typename Y> typename TypeFun, typename I>
using Accumulate = typename AccumulateT<List, TypeFun, I>::Type;
```
这里初始类型 I 也被当作累加器使用，被用来捕捉当前的结果。因此当递归到类型列表末
尾的时候，递归循环的基本情况会返回这个结果。在递归情况下，算法将 TypeFun 作用于之前的结
果（I）以及当前类型列表的首元素，并将 TypeFun 的结果作为初始类型继续传递，用于下一级对
剩余列表的求和（Accumulating）。
我们可以修改之前的LargestType来验证我们的累加算法：
```
// 新的找寻最大值
	template<typename T, typename U>
	class LargestTypeT_New : public _19_7_1_::ifThenElseT<sizeof(T) >= sizeof(U), T, U>
	{

	};

	template<typename List, bool = IsEmpty<List>::value>
	class LargestTypeAccT;

	template<typename List>
	class LargestTypeAccT<List, false> : public AccumulateT<FrontPop<List>, LargestTypeT_New, Front<List>>
	{

	};

	template<typename List>
	class LargestTypeAccT<List, true>
	{ };

	template<typename List>
	using LargestTypeAcc = typename LargestTypeAccT<List>::Type;
```
## 11. 类型列表的插入排序
作为最后一个类型列表相关的算法，我们来介绍插入排序。和其它算法类似，其递归过程会
将类型列表分成第一个元素（Head）和剩余的元素（Tail）。然后对 Tail 进行递归排序，并
将 Head 插入到排序后的类型列表中的合适的位置。该算法的实现如下：
```
template<typename List, template<typename T, typename U> typename Compare, bool = IsEmpty<List>::value>
class InsertionSortT;

template<typename List, template<typename T, typename U> typename Compare>
using InsertionSort = typename InsertionSortT<List, Compare>::Type;

template<typename List, typename Element, template<typename T, typename U>typename Compare, bool = IsEmpty<List>::value>
class InsertSortedT;

template<typename List, typename Element, template<typename T, typename U>typename Compare>
using InsertSorted = typename InsertSortedT<List, Element, Compare>::Type;

template<typename List, template<typename T, typename U> typename Compare>
class InsertionSortT<List, Compare, false>
	: public InsertSortedT<InsertionSort<FrontPop<List>, Compare>, Front<List>, Compare>
{

};

template<typename List, template<typename T, typename U> typename Compare>
class InsertionSortT<List, Compare, true>
{
public:
	using Type = List;
};

template<typename List, typename Element, template<typename T, typename U>typename Compare>
class InsertSortedT<List, Element, Compare, false>
{
	using NewTail = typename _19_7_1_::IfThenElse<Compare<Element, Front<List>>::value,
		_19_7_1_::IdentityT<List>, InsertSortedT<FrontPop<List>, Element, Compare>>::Type;

	using NewHead = _19_7_1_::IfThenElse<Compare<Element, Front<List>>::value, Element, Front<List>>;
public:
	using Type = FrontPush<NewTail, NewHead>;
};

template<typename List, typename Element, template<typename T, typename U>typename Compare>
class InsertSortedT<List, Element, Compare, true> : public FrontPushT<List, Element>
{};
```
## 12 非类型类型列表
通过类型列表，有非常多的算法和操作可以用来描述并操作一串类型。某些情况下，还会希
望能够操作一串编译期数值，比如多维数组的边界，或者指向另一个类型列表中的索引。
有很多种方法可以用来生成一个包含编译期数值的类型列表。一个简单的办法是定义一个类
模板 CTValue（compile time value），然后用它表示类型列表中某种类型的值:
```
template<typename T, T Value>
struct CTValue
{
	static constexpr T value = Value;
};

```
用它就可以生成一个包含了最前面几个素数的类型列表：
```
using Primes = _24_::TypeList<CTValue<int, 2>, CTValue<int, 3>,
		CTValue<int, 5>, CTValue<int, 7>,
		CTValue<int, 11>>;
```
我们可以对此进行累加操作：
```
template<typename T,typename U>
struct MultiplyT;
	
template<typename T, T Value1, T Value2>
struct MultiplyT<CTValue<T,Value1>, CTValue<T, Value2>>
{
public:
	using Type = CTValue<T, Value1* Value2>;
};

template<typename T, typename U>
using Multiply = typename MultiplyT<T, U>::Type;
```
然后结合 MultiplyT，下面的表达式就会返回所有 Primes 中素数的乘积:
```
Accumulate<Primes, MultiplyT, CTValue<int, 1>>::value
```
不过这过于复杂，尤其是当所有值相同的时候。我们可以引入可以通过引入 CTTypelist 模板别名来进行优化：
```
template<typename T, T… Values>
using CTTypelist = Typelist<CTValue<T, Values>…>;
```
这样就可以使用 CTTypelist 来定义一版更为简单的 Primes（素数）：
```
using Primes = CTTypelist<int, 2, 3, 5, 7, 11>;
```
这一方式的唯一缺点是，别名终归只是别名，当遇到错误的时候，错误信息可能会一直打印
到 CTValueTypes 中的底层 Typelist，导致错误信息过于冗长。为了解决这一问题，可以定义
一个能够直接存储数值的、全新的类型列表类 Valuelist：
```
template<typename T, T...Values>
struct Valuelist
{

};

template<typename T, T... Values>
struct IsEmpty< Valuelist <T, Values...> >
{
	static constexpr bool value = sizeof...(Values) == 0;
};

template<typename T, T Head, T... Tail>
struct FrontT<Valuelist<T, Head, Tail...>>
{
	using Type = CTValue<T, Head>;
	static constexpr T value = Head;
};

template<typename T,T Head,T... Tail>
struct FrontPopT<Valuelist<T,Head,Tail...>>
{
	using Type = Valuelist<T, Tail...>;
};

template<typename T,T... Values,T New>
struct FrontPushT<Valuelist<T,Values...>,CTValue<T,New>>
{
	using Type = Valuelist<T, New, Values...>;
};

template<typename T,T... Values,T New>
struct PushBackT<Valuelist<T,Values...>,CTValue<T,New>>
{
	using Type = Valuelist<T, Values..., New>;
};
```

## 13.从类型列表中选取几个类型组成新类型列表
功能要求就是输入类型列表以及不定参数的索引并返回新的列表。
```
// 选取几个元素组成新的。
template<typename Types, typename Indices>
class SelectT;

template<typename Types, unsigned... Indices>
class SelectT<Types, Valuelist<unsigned, Indices...>>
{
public:
	using Type = TypeList<NthElement<Types, Indices>...>;
};

template<typename Types, unsigned... Indices>
using Select = typename SelectT<Types, Valuelist<unsigned,Indices...>>::Type;
```
## 14.Cons-style Typelists
在引入变参模板之前，类型列表通常参照 LISP 的 cons 单元的实现方式，用递归数据结构实
现。每一个 cons 单元包含一个值（列表的 head）和一个嵌套列表，这个嵌套列表可以是另
一个 cons 单元或者一个空的列表 nil。这一思路可以直接在 C++中按照如下方式实现：
```
// cons-style TypeList
class Nil
{
};

emplate<typename HeadT,typename TailT = Nil>
class Cons
{
public:
 using Head = HeadT;
 using Tail = TailT;
};

using SignedIntegralTypes = Cons<signed char, Cons<short, Cons<int,
 Cons<long, Cons<long long, Nil>>>>>;

template<typename List>
class FrontT
{
public:
 using Type = typename List::Head;
};

template<typename List,typename Element>
class FrontPushT
{
public:
 using Type = Cons<Element, List>;
};

template<typename List>
class FrontPopT
{
public:
 using Type = typename List::Tail;
};

template<>
struct IsEmpty<Nil> 
{
 static constexpr bool value = true;
};
```
## 测试代码：
```
int main()
{
	using TestType = _24_::TypeList<short, int, long, signed char, long long>;
		using TestType1 = _24_::FrontPush<_24_::FrontPop<TestType>,bool>;
		using TestType2 = _24_::NthElement<TestType, 2>;
		using LargestType = _24_::LargestType<TestType>;
		using PushBack_OldType = _24_::PushBack_Old<TestType, bool>;
		using PushBackType = _24_::PushBack<TestType, bool>;
		using NullType = _24_::TypeList<>;
		using PushBack_OldType2 = _24_::PushBack_Old<NullType, bool>;
		using ReverseType = _24_::Reverse<TestType>;
		using TranformType = _24_::Transform<TestType>;
		using ResultType = _24_::Accumulate<TestType, _24_::FrontPushT, NullType>;
		using LargestType_New = _24_::LargestTypeAcc<TestType>;
		using ST = _24_::InsertionSort<TestType, _24_::SmallerThanT>;
		int ResultPrimes = _24_::Accumulate<_24_3_1_::Primes, _24_3_1_::MultiplyT, _24_3_1_::CTValue<int, 1>>::value;
		int ResultCTPrimes = _24_::Accumulate<_24_3_1_::CTPrimes, _24_3_1_::MultiplyT, _24_3_1_::CTValue<int, 1>>::value;
		using Integers = _24_::Valuelist<int, 6, 2, 4, 9, 5, 2, 1, 7>;
		using SortedIntegers = _24_::InsertionSort<Integers, _24_::GreaterThanT>;
		int ResultCTPrimess = _24_::Accumulate<_24_::Primes, _24_::MultiplyT, _24_::CTValue<int, 1>>::value;
		using SelectType = _24_::Select<TestType, 1, 3, 4>;
		using SortedTypes = _24_::InsertionSort<_24_::SignedIntegralTypes, _24_::BiggerThanT>;
		std::cout << "原始：" << typeid(TestType).name() << std::endl;
		std::cout << "添加：" << typeid(TestType1).name() << std::endl;
		std::cout << "取第2个元素的类型：" << typeid(TestType2).name() << std::endl;
		std::cout <<"最大类型的大小："<< sizeof(LargestType) << std::endl;
		std::cout << typeid(PushBack_OldType).name() << std::endl;
		std::cout << typeid(PushBackType).name() << std::endl;
		std::cout << typeid(PushBack_OldType2).name() << std::endl;
		std::cout << typeid(ReverseType).name() << std::endl;
		std::cout <<" TranformType = " << typeid(TranformType).name() << std::endl;
		std::cout <<" ResultType = " << typeid(ResultType).name() << std::endl;
		std::cout << "最大类型的大小_New：" << sizeof(LargestType_New) << std::endl;
		std::cout << "ST：" << typeid(ST).name() << std::endl;
		std::cout << "ResultPrimes = " << ResultPrimes << std::endl;
		std::cout << "ResultCTPrimes = " << ResultCTPrimes << std::endl;
		std::cout << "SortedIntegers：" << typeid(SortedIntegers).name() << std::endl; 
		std::cout << "ResultCTPrimess = " << ResultCTPrimess << std::endl;
		std::cout << "SelectType：" << typeid(SelectType).name() << std::endl;
		std::cout << "SortedTypes：" << typeid(SortedTypes).name() << std::endl;
}
```
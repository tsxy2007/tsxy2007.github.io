---
title: C++模板-元组
date: 2022-10-25 17:37:51
tags:
---

C++的tuple(元组)是一个可以存储不同类型的组件,类似于c++的struct的方式来组织数据;只不过tuple是通过位置信息索引的，而不是通过名字。   
## 一: 存储
tuple包含了对模板参数列表中每一个类型的存储。这部分存储可以通过函数模板get进行访问，对于元组t，其用法为get<I>(t).
tuple实现的一种思路是：包含一个头元素以及剩余参数列表的tuple。代码如下：
```
template<typename... Types>
	class Tuple;

	template<typename Head, typename... Tail>
	class Tuple<Head, Tail...>
	{
	private:
		Head head;
		Tuple<Tail...> tail;
	public:
		Tuple()
		{

		}

		Tuple(Head const& inHead, Tuple<Tail...> const& inTail) :
			head(inHead), tail(inTail)
		{

		}

		Head& getHead() { return head; }
		Head const& getHead()const { return head; }

		Tuple<Tail...>& getTail() { return tail; }
		Tuple<Tail...> const& getTail() const { return tail; }
	};

	template<>
	class Tuple<>
	{

	};
```
至于怎么获取tuple里某个索引的元素，我们定义一个工具类tupleGet，其中我们定义一个非类型模板参数N这是我们定义类型索引当N=0的时候我们返回head代码如下：
```
template<unsigned N>
	struct TupleGet
	{
		template<typename Head,typename... Tail>
		static auto apply(Tuple<Head, Tail...> const& t)
		{
			return TupleGet<N - 1>::apply(t.getTail());
		}
	};

	template<>
	struct TupleGet<0>
	{
		template<typename Head,typename... Tail>
		static Head const& apply(Tuple<Head, Tail...> const& t)
		{
			return t.getHead();
		}
	};

	template<unsigned N,typename...Types>
	auto get(Tuple<Types...> const& t)
	{
		return TupleGet<N>::apply(t);
	}
```
我们来分析下TupleGet运行原理，假如我们有一个Tuple类型如下：
```
Tuple<int, double, std::string,int> t(17, 3.14, "Hello, World!",3);
```
我们想要获取get<2>(t)的值即N=1的时候
```
TupleGet<2>::apply(t);->t.getTail(); 当前的head是3.14
TupleGet<1>::apply(t);->t.getTail(); 当前的head是"Hello,World"
TupleGet<0>::apply(t);->t.getHead(); 当前的head是"Hello,World"
```
## 二:构造
我们之前可以通过默认构造和传递首值，另外一个Tuple创建新Tuple，但是只有这两种方式远远不够，我们需要通过不定参数，移动语义以及通过另外一个tuple创建。
```

template<typename VHead,typename... VTail,typename = std::enable_if_t<sizeof...(VTail)==sizeof...(Tail)>>
Tuple(VHead&& vHead,VTail&&... vtail) : head(std::forward<VHead>(vHead)), tail(std::forward<VTail>(vtail)...)
{

}
/* enable_if_t是因为Tuple<long int, long double, std::string> t2(t); 会更匹配用一组数值去初始化一个元组的构造函数模板，而不是用
一个元组去初始化另一个元组的构造函数模板*/
template<typename VHead,typename...VTail,typename = std::enable_if_t<sizeof...(VTail)==sizeof..(Tail)>>
Tuple(Tuple<VHead, VTail...> const& other)
	:head(other.getHead(), tail(other.getTail()))
{

}
```
接下来我们定义一个函数方便我们构造tuple
```
template<typename... Types>
auto makeTuple(Types&&... elems)
{
	return Tuple<std::decay_t<Types>...>(std::forward<Types>(elems)...);
}
```
## 三: 比较
元组是包含了其他数值的结构化类型。。为了比较两个元组，就需要比较它们的元素。因此可
以像下面这样，定义一种能够逐个比较两个元组中元素的 operator==：
```
	// 比较
	bool operator==(Tuple<> const&, Tuple<>const&)
	{
		return true;
	}

	template<typename Head1,typename...Tail1,typename Head2,typename...Tail2,
	typename = std::enable_if_t<sizeof...(Tail1)== sizeof...(Tail2)>>
		bool operator==(Tuple<Head1, Tail1...>const& lhs, Tuple<Head2, Tail2...>const& rhs)
	{
		return lhs.getHead() == rhs.getHead() && lhs.getTail() == rhs.getTail();
	}
```
## 四: 输出
我们可以通过定义一个friend operator<<函数来输出tuple内容：
```
	void printTuple(std::ostream& strm, Tuple<> const&, bool isFirst = true)
	{
		strm << (isFirst ? "( )" : ")");
	}

	template<typename Head,typename... Tail>
	void printTuple(std::ostream& strm, Tuple<Head, Tail...> const& t, bool isFirst = true)
	{
		strm << (isFirst ? "(" : ",");
		strm << t.getHead();
		printTuple(strm, t.getTail(), false);
	}

	template<typename... Types>
	std::ostream& operator<<(std::ostream& strm, Tuple<Types...>const& t)
	{
		printTuple(strm, t);
		return strm;
	}
```

## 五: 将元组用作类型列表
<b>tuple既有运行期运算(存储元素值)也有编译期间运算</b>可以通过部分特例化，可以将Tuple变成一个功能完整的[Typelist][!www.baidu.com]
```
	// 将元组用作类型列表
	template<>
	struct IsEmpty<Tuple<>>
	{
		static constexpr bool value = true;
	};

	template<typename Head,typename... Tail>
	class FrontT<Tuple<Head, Tail...>>
	{
	public:
		using Type = Head;
	};

	template<typename Head,typename... Tail>
	class FrontPopT<Tuple<Head, Tail...>>
	{
	public:
		using Type = Tuple<Tail...>;
	};

	template<typename... Types,typename Element>
	class FrontPushT<Tuple<Types...>, Element>
	{
	public:
		using Type = Tuple<Element, Types...>;
	};

	template<typename... Types,typename Element>
	class PushBackT<Tuple<Types...>, Element>
	{
	public:
		using Type = Tuple<Types..., Element>;
	};

```

## 六: 添加以及删除元素
对于添加元素，我们需要利用typelist的FrontPushT获取返回的tuple类型，然后我们用新的类型定义一个新的对象返回代码如下：
```
	// 添加以及删除元素

	template<typename... Types,typename V>
	FrontPush<Tuple<Types...>,V> pushFront(Tuple<Types...> const& tuple, V const& value)
	{
		//typelist的FrontPushT获取返回的tuple类型
		using T = FrontPush<Tuple<Types...>, V>;
		return T(value, tuple);
	}

```
同理添加元素到末尾代码如下：
```
	template<typename Head,typename... Tail,typename V>
	Tuple<Head, Tail...,V> pushBack(Tuple<Head, Tail...>const& tuple, V const& value)
	{
		return Tuple<Head, Tail..., V>(tuple.getHead(), pushBack(tuple.getTail(), value));
	}
```
删除头元素代码如下：
```
	template<typename... Types>
	FrontPop<Tuple<Types...>> frontPop(Tuple<Types...> const& tuple)
	{
		return tuple.getTail();
	}
```

## 七: 元组的反转
元组的反转可以采用另一种递归的、类型列表的反转方式实现：
```
	// 元组的反转
	Tuple<> reverse(Tuple<> const& t)
	{
		return t;
	}

	template<typename Head,typename... Tail>
	Reverse<Tuple<Head, Tail...>> reverse(Tuple<Head,Tail...> const& t)
	{
		return pushBack(reverse(t.getTail()), t.getHead());
	}
```

## 八: 删除最后一个元素
```
	// 删除最后一个元素
	template<typename... Types>
	PopBack<Tuple<Types...>> popBack(Tuple<Types...> const& t)
	{
		return reverse(frontPop(reverse(t)));
	}
```

## 九: 测试反转效率
上文的反转tuple的递归方法虽然正确，但是运行期间的效率却非常低，为了展现这个问题，我们引入以下可以计算被copy次数的类：
```
// 计算copy 次数
	template<int N>
	struct CopyCounter
	{
		inline static unsigned numCopies = 0;
		CopyCounter()
		{

		}

		CopyCounter(CopyCounter const&)
		{
			++numCopies;
		}
	};
 //测试代码：
 void copycountertest()
{
	_25_2_::Tuple<CopyCounter<0>, CopyCounter<1>, CopyCounter<2>, CopyCounter<3>,
		CopyCounter<4>> copies;
	auto reversed = reverse(copies);
	std::cout << "0: " << CopyCounter<0>::numCopies << " copies\n";
	std::cout << "1: " << CopyCounter<1>::numCopies << " copies\n";
	std::cout << "2: " << CopyCounter<2>::numCopies << " copies\n";
	std::cout << "3: " << CopyCounter<3>::numCopies << " copies\n";
	std::cout << "4: " << CopyCounter<4>::numCopies << " copies\n";
}
```
以上代码输出如下（我都编译器vs2022）   
0: 5 copies   
1: 8 copies   
2: 9 copies   
3: 8 copies   
4: 5 copies   
copy了很多次，为了不必要的copy，我们先用一个笨办法，已知我们知道tuple的大小。我们可以使用以下简单的算法:
```
auto copycountertest2 = []()
{
	_25_2_::Tuple<CopyCounter<0>, CopyCounter<1>, CopyCounter<2>, CopyCounter<3>,CopyCounter<4>> copies;
	auto reversed = _25_2_::makeTuple(_25_2_::get<4>(copies), _25_2_::get<3>(copies), _25_2_::get<2>(copies), _25_2_::get<1>(copies), _25_2_::get<0>(copies));
	std::cout << "0: " << CopyCounter<0>::numCopies << " copies\n";
	std::cout << "1: " << CopyCounter<1>::numCopies << " copies\n";
	std::cout << "2: " << CopyCounter<2>::numCopies << " copies\n";
	std::cout << "3: " << CopyCounter<3>::numCopies << " copies\n";
	std::cout << "4: " << CopyCounter<4>::numCopies << " copies\n";
};															
```
## 十：通过索引列表进行反转
如果我们在编译期就知道反转的索引，那么我们在运行期就可以直接通过反转的索引队列构造tuple这样我们就可以减少copy次数。
我们先通过以下简单的模板MakeIndexList，逐步生成索引列表：
```
	// 构造索引列表从0-N-1
	template<unsigned N, typename Result = Valuelist<unsigned>>
	struct MakeIndexListT : MakeIndexListT<N - 1, FrontPush<Result, CTValue<unsigned, N - 1>>>
	{

	};

	template<typename Result>
	struct MakeIndexListT<0, Result>
	{
		using Type = Result;
	};

	template<unsigned N>
	using MakeIndexList = typename MakeIndexListT<N>::Type;
```
现在可以结合MakeIndexList和之前的Reverse算法，生成所需的索引列表：
```
using MyIndexList = Reverse<MakeIndexList<5>>;
```
为了真正实现反转，需要将索引列表中的索引捕获进一个非类型参数包。这可以通过将
reverse()分成两部分来实现：
```
	//
	template<typename... Elements, unsigned... Indices>
	auto reverseImpl(Tuple<Elements...> const& t, Valuelist<unsigned, Indices...>)
	{
		return makeTuple(get<Indices>(t)...);
	}

	template<typename... Elements>
	auto reverseN(Tuple<Elements...> const& t)
	{
		return reverseImpl(t, Reverse<MakeIndexList<sizeof...(Elements)>>());
	}
```
## 十一：洗牌和选择
我们通过reverseImpl这个方法可以从已有的元组选出特定的值，并用它们生成一个新的元组。我现在实现一个简单的例子就是splat，从tuple选出某一个元素，并将之重复若干次之后组成一个新的元组。比如：
```
auto tuple = _25_2_::makeTuple(1, 2.5f, "hello world");
auto SelectList = _25_2_::splat<1, 4>(ReverseTuple2);
std::cout << "SelectList: " << SelectList << std::endl;
```
他会生成一个Tuple<double,double,double,double>类型的元组，其中每个值都是2.5f。   
为了达到这个效果我们先定义一个ReplicatedIndexListT构造一个重复索引的valuelist代码如下：
```
	template<unsigned I, unsigned N, typename IndexList = Valuelist<unsigned>>
	class ReplicatedIndexListT;

	template<unsigned I, unsigned N, unsigned... Indices>
	class ReplicatedIndexListT<I, N, Valuelist<unsigned, Indices...>>
		:public ReplicatedIndexListT<I, N - 1, Valuelist<unsigned, Indices..., I>>
	{};

	template<unsigned I, unsigned... Indices>
	class ReplicatedIndexListT<I, 0, Valuelist<unsigned, Indices...> >
	{
	public:
		using Type = Valuelist<unsigned, Indices...>;
	};

	template<unsigned I, unsigned N>
	using ReplicatedIndexList = typename ReplicatedIndexListT<I, N>::Type;

	template<unsigned I, unsigned N, typename... Elements>
	auto splat(Tuple<Elements...> const& t)
	{
		return select(t, ReplicatedIndexList<I, N>());
	}
```
我们也可以用这种方法实现Tuple的类型排序(按照类型大小),之前我们定义的比较类型大小的方法SmallerThanT
```
	//NthElement 具体用法在TypeList章节有。
	template<typename List, template<typename T, typename U> typename Func>
	class MetafunOfNthElementT
	{
	public:
		template<typename T, typename U>
		class Apply;

		template<unsigned N, unsigned M>
		class Apply<CTValue<unsigned, M>, CTValue<unsigned, N>> :
			public Func<NthElement<List, M>, NthElement<List, N>>
		{
		};
	};

	template<template<typename T, typename U> typename Compare, typename... Elements>
	auto sort(Tuple<Elements...> const& t)
	{
		return select(t,
			InsertionSort<MakeIndexList<sizeof...(Elements)>,
			MetafunOfNthElementT<Tuple<Elements...>,
			Compare>::template Apply>());
	}
```

## 十二: 元组的展开
```
// 元组的展开
	template<typename F, typename ... Elements, unsigned...Indices>
	auto applyImpl(F f, Tuple<Elements...> const& t,
		Valuelist<unsigned, Indices...>)->decltype(f(get<Indices>(t)...))
	{
		return f(get<Indices>(t)...);
	}

	template<typename F, typename... Elements, unsigned N = sizeof...(Elements)>
	auto apply(F f, Tuple<Elements...> const& t)->decltype(applyImpl(f, t, MakeIndexList<N>()))
	{
		return applyImpl(f, t, MakeIndexList<N>());
	}
```
##  十三：元组的优化
我们实现的元组，其存储方式所需要的存储空间，要比其严格意义上所需要的存储空间多。
其中一个问题是，tail 成员最终会是一个空的数值（因为所有非空的元组都会以一个空的元
组作为结束），而任意数据成员又总会至少占用一个字节的内存   
为了提高元组的存储效率，我们可以使用之前讲过的EBCO，让tuple继承自一个tail tuple（尾元组），而不是一个成员代码如下：
```
template<typename Head, typename... Tail>
class Tuple<Head, Tail...> :  private Tuple<Tail...>
{
private:
	Head head;
}
```
但是这样初始化会造成一个问题。就是尾元组会比Head成员先初始化。解决方案是我们把head 也放入父类代码如下：
```
// Tuple
 /* 虽然这一方式解决了元素初始化顺序的问题，但是却引入了一个更糟糕的问题：如果一个元
	组包含两个类型相同的元素（比如 Tuple<int, int>），我们将不再能够从中提取元素，因为
	此时从 Tuple<int, int>向 TupleElt<int>的转换（自派生类向基类的转换）不是唯一的（有歧义）。
	为了打破歧义，需要保证在给定的 Tuple 中每一个 TupleElt 基类都是唯一的。一个方式是将
	这个值的“高度”信息（也就是 tail 元组的长度信息）编码进元组中。元组最后一个元素的
	高度会被存储生 0，倒数第一个元素的长度会被存储成 1，以此类推：
*/
	template<unsigned Height, typename T, bool = std::is_class_v<T> && !std::is_final_v<T>>
	class TupleElt;

	template<unsigned Height, typename T>
	class TupleElt<Height, T, false>
	{
		T value;

	public:

		TupleElt() = default;

		template<typename U>
		TupleElt(U&& other) : value(std::forward<U>(other))
		{}

		T& get() { return value; }

		T const& get() const { return value; }
	};

	template<unsigned Height, typename T>
	class TupleElt<Height, T, true> : private T
	{
	public:

		TupleElt() = default;

		template<typename U>
		TupleElt(U&& other) : T(std::forward<U>(other))
		{}

		T& get() { return *this; }

		T const& get() const { return *this; }
	};

	template<typename... Types>
	class Tuple;
	template<typename Head, typename... Tail>
	class Tuple<Head, Tail...> : private TupleElt<sizeof...(Tail), Head >, private Tuple<Tail...>
	{
		using HeadElt = TupleElt<sizeof...(Tail), Head>;
		using _MyBase = Tuple<Tail...>;
	public:
		Tuple()
		{

		}

		Tuple(Head const& inHead, Tuple<Tail...> const& inTail) 
			: HeadElt(inHead),
			_MyBase(inTail)
		{

		}
		
		template<typename VHead, typename... VTail, typename = std::enable_if_t<sizeof...(VTail) == sizeof...(Tail)>>
		Tuple(VHead&& vHead, VTail&&... vtail) : HeadElt(std::forward<VHead>(vHead)), _MyBase(std::forward<VTail>(vtail)...)
		{

		}

		template<typename VHead, typename...VTail, typename = std::enable_if_t<sizeof...(VTail) == sizeof...(Tail)>>
		Tuple(Tuple<VHead, VTail...> const& other) : HeadElt(other.getHead()), _MyBase(other.getTail())
		{

		}

		Head& getHead() { return static_cast<HeadElt*>(this)->get(); }
		Head const& getHead()const { return static_cast<HeadElt const*>(this)->get(); }

		Tuple<Tail...>& getTail() { return *this; }
		Tuple<Tail...> const& getTail() const { return *this; }

		template<unsigned I, typename... Elements>
		friend auto getN(Tuple<Elements...>& t)
			-> decltype(getHeight<sizeof...(Elements) - I - 1>(t));
	};

	template<>
	class Tuple<>
	{

	};

```
当提供给 TupleElt 的模板参数是一个可以被继承的类的时候，它会从该模板参数做 private
继承从而也可以将 EBCO 用于被存储的值。
有了高度信息我们就可以直接用过获取高度，直接定位到我们想要的类型代码如下：
```
	template<unsigned H, typename T>
	T& getHeight(TupleElt<H, T>& te)
	{
		return te.get();
	}

	template<unsigned I,typename... Elements>
	auto getN(Tuple<Elements...>& t)->decltype(getHeight<sizeof...(Elements) - I - 1>(t))
	{
		return getHeight<sizeof...(Elements) - 1 - I>(t);
	}
```
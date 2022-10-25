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
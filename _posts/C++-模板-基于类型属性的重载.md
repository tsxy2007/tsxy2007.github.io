---
title: C++-模板-基于类型属性的重载
date: 2022-10-01 22:10:18
tags:
---
## 函数模板
### 一：算法特化(算法重载)
函数模板重载的一个动机就是基于算法适用的类型信息，为算法提供更为特化的版本。考虑一个交换两个数值的swap()操作：
```
template<typename T>
	void swap(T& x, T& y)
	{
		T tmp(x);
		x = y;
		y = tmp;
	}
```
这一实现用到了三次拷贝操作。但是对于某些类型，可以有更为高效的swap()实现，比如std::vector:
```
template <typename T>
	void swap(std::vector<T>& x, std::vector<T>& y)
	{
		x.swap(y);
	}
```
两种都可以正确交换两个vector对象的内容.但是后者实现方式的效率要高很多，因为它利用了vector额外的特性，因此后一种实现比第一种实现更为特化。为一个泛型算法中引入更为特化的变体，这一设计和优化方式被称为算法特化。   
在适用的情况下更为特化的算法变体会自动的被选择，这一点对算法特化的实现至关重要，调用者甚至不知道具体的变体的存在。   
并不是所有的概念上更为特化的算法变体，都可以直接转换成提供了正确的部分排序行为的函数模板。比如：
```
template<typename Iterator,typename Distance>
	void advanceIter(Iterator& x, Distance n)
	{
		while(n>0)
        {
            ++x;
            --n;
        }
	}
```
对于特定类型的迭代器（随机访问操作的迭代器），我们可以为该操作提供一个更为高效的实现方式
```
template<typename RandomAccessIterator,typename Distance>
	void advanceIter(RandomAccessIterator& x, Distance n)
	{
		x+=n;
	}
```
同时定义以上两种函数模板会导致编译错误， 这是因为只有模板参数名字不同的函数模板是不可以被重载的。本章剩余的内容会讨论能够 允许我们实现类似上述函数模板重载的一些技术。
### 二：标记派发(Tag Dispatching)
算法特化的一种方式是，用一个唯一的，可以区分特定变体的类型来标记不同算法变体的实现。代码如下：
```
namespace _20_2_
{
	template<typename Iterator,typename Distance>
	void advanceIterImpl(Iterator& x, Distance n, std::input_iterator_tag)
	{
		std::cout << "std::input_iterator_tag" << std::endl;
		while (n > 0)
		{
			++x;
			--n;
		}
	}

	template<typename Iterator, typename Distance>
	void advanceIterImpl(Iterator& x, Distance n, std::random_access_iterator_tag)
	{
		std::cout << "std::random_access_iterator_tag" << std::endl;
		x += n;
	}

	template<typename Iterator,typename Distance>
	void advanceIter(Iterator& x, Distance n)
	{
		advanceIterImpl(x, n, std::iterator_traits<Iterator>::iterator_category());
	}
}
```
根据stl标准库可以通过
```
std::iterator_traits<Iterator>::iterator_category()
```
获取当前迭代器的tag，然后给每个算法函数打上tag实现不同调用不同算法函数；
tag种类如下：
```
struct input_iterator_tag {};

struct output_iterator_tag {};

struct forward_iterator_tag : input_iterator_tag {};

struct bidirectional_iterator_tag : forward_iterator_tag {};

struct random_access_iterator_tag : bidirectional_iterator_tag {};
```
### 三：Enable/Disable 函数模板
算法特化需要提供可以基于模板参数的属性进行选择的、不同的函数模板。不幸的是，无论 是函数模板的部分排序规则还是重载解析，都不能满足更 为高阶的算法特化的要求。
C++标准库为之提供了一个辅助工具是std::enable_if，我们现在实现自己的版本并称之为EnableIf,代码如下:
```
namespace _20_3_
{
	template<bool,typename T = void>
	struct EnableIfT
	{

	};

	template<typename T>
	struct EnableIfT<true,T>
	{
		using Type = T;
	};

	template<bool Cond,typename T = void>
	using EnableIf = typename EnableIfT<Cond, T>::Type;
}
```
如果匹配true Type为相关的T否则什么都不发生；
我们先定义一个函数,功能是判定是否是随机访问操作的迭代器是的话可以获取该类型，不是什么都不操作：
```
	template<typename Iterator>
	constexpr bool IsRandomAccessIterator = std::is_convertible_v<typename std::iterator_traits<Iterator>::iterator_category,
		std::random_access_iterator_tag>;
```
具体用法如下：
```
template<typename Iterator,typename Distance>
	EnableIf<IsRandomAccessIterator<Iterator>> advanceIter(Iterator& x, Distance n)
	{
		std::cout << "std::random_access_iterator_tag" << std::endl;
		x += n;
	}

	template<typename Iterator,typename Distance>
	EnableIf<!IsRandomAccessIterator<Iterator>> advanceIter(Iterator& x, Distance n)
	{
		std::cout << "std::input_iterator_tag" << std::endl;
		while (n>0)
		{
			++x;
			--n;
		}
	}
```
#### 提供多种特化版本
我们还可以继续添加该算法的特化版本，比如允许指定一个负的距离参数， 以让迭代器向“后”移动。很显然这对一个“输入迭代器（input itertor）”是不适用的，对 一个随机访问迭代器却是适用的。我们的算法现在面临三种情况如下：
1. 随机访问迭代器：适用于随机访问的情况（常数时间复杂度，可以向前或向后移动） 
2. 双向迭代器但又不是随机访问迭代器：适用于双向情况（线性时间复杂度，可以向前或 向后移动） 
3. 输入迭代器但又不是双向迭代器：适用于一般情况（线性时间复杂度，只能向前移动）
具体完整代码如下：

```
namespace _20_3_1_ // 提供多种特化版本
{
	template<bool, typename T = void>
	struct EnableIfT
	{

	};

	template<typename T>
	struct EnableIfT<true, T>
	{
		using Type = T;
	};

	template<bool Cond, typename T = void>
	using EnableIf = typename EnableIfT<Cond, T>::Type;

	template<typename Iterator>
	constexpr bool IsRandomAccessIterator = std::is_convertible_v<typename std::iterator_traits<Iterator>::iterator_category,
		std::random_access_iterator_tag>;

	template<typename Iterator>
	constexpr bool IsBidirectionalIterator = std::is_convertible_v<typename std::iterator_traits<Iterator>::iterator_category,
		std::bidirectional_iterator_tag>;

	template<typename Iterator, typename Distance>
	EnableIf<IsRandomAccessIterator<Iterator>> advanceIter(Iterator& x, Distance n)
	{
		std::cout << "std::random_access_iterator_tag" << std::endl;
		x += n;
	}

	template<typename Iterator, typename Distance>
	EnableIf<IsBidirectionalIterator<Iterator>
		&&!IsRandomAccessIterator<Iterator>> advanceIter(Iterator& x, Distance n)
	{
		std::cout << "std::bidirectional_iterator_tag" << std::endl;
		if (n > 0) 
		{
			for (; n > 0; ++x, --n) 
			{ //linear time 
			} 
		} 
		else 
		{ 
			for ( ; n < 0; --x, ++n) 
			{ //linear time 
			} 
		}
	}

	template<typename Iterator, typename Distance>
	EnableIf<!IsBidirectionalIterator<Iterator>> advanceIter(Iterator& x, Distance n)
	{
		if (n < 0) 
		{ 
			throw "advanceIter(): invalid iterator category for negative n"; 
		}
		std::cout << "std::random_access_iterator_tag" << std::endl;
		while (n > 0)
		{
			++x;
			--n;
		}
	}
}
```
### 四：编译期if
c++17提供了编译期if的功能让我们可以把代码写在同一函数里：
```
namespace _20_3_3_ 
{
	//编译期 if

	template<typename Iterator>
	constexpr bool IsRandomAccessIterator = std::is_convertible_v<typename std::iterator_traits<Iterator>::iterator_category,
		std::random_access_iterator_tag>;

	template<typename Iterator>
	constexpr bool IsBidirectionalIterator = std::is_convertible_v<typename std::iterator_traits<Iterator>::iterator_category,
		std::bidirectional_iterator_tag>;

	template<typename Iterator>
	constexpr bool IsInputIterator = std::is_constructible_v<typename std::iterator_traits<Iterator>::Iterator_category,
		std::input_iterator_tag>;

	template<typename Iterator,typename Distance>
	void advanceIter(Iterator& x, Distance n)
	{
		if constexpr(IsRandomAccessIterator<Iterator>)
		{
			std::cout << "std::IsRandomAccessIterator" << std::endl;
			x += n;
		}
		else if constexpr(IsBidirectionalIterator<Iterator>)
		{
			std::cout << "std::bidirectional_iterator_tag" << std::endl;
			if (n > 0)
			{
				for (; n > 0; ++x, --n)
				{ //linear time 
				}
			}
			else
			{
				for (; n < 0; --x, ++n)
				{ //linear time 
				}
			}
		}
		else
		{
			std::cout << "std::input_iterator_tag" << std::endl;
			if (n<0)
			{
				throw "advanceIter(): invalid iterator category for negative n";
			}
			while (n > 0)
			{
				++x;
				--n;
			}
		}
	}
}
```
但是，该方法也有其缺点。只有在泛型代码组件可以被在一个函数模板中完整的表述时，这 一使用 constexpr if 的方法才是可能的。在下面这些情况下，我们依然需要 EnableIf： 
1. 需要满足不同的“接口”需求 
2. 需要不同的 class 定义 
3. 对于某些模板参数列表，不应该存在有效的实例化。
### 五：Concepts
上述技术到目前为止都还不错，但是有时候却稍显笨拙，它们可能会占用很多的编译器资源， 以及在某些情况下，可能会产生难以理解的错误信息。因此某些泛型库的作者一直都在盼望 着一种能够更简单、直接地实现相同效果的语言特性。为了满足这一需求，一个被称为 conceptes 的特性很可能会被加入到 C++语言中代码如下：
```
namespace _20_3_4_
{
	//Concepts

	template<typename Iterator>
	concept IsRandomAccessIterator = std::is_convertible_v<typename std::iterator_traits<Iterator>::iterator_category,
		std::random_access_iterator_tag>;

	template<typename Iterator>
	concept IsBidirectionalIterator = std::is_convertible_v<typename std::iterator_traits<Iterator>::iterator_category,
		std::bidirectional_iterator_tag>;

	template<typename Iterator>
	concept IsInputIterator = std::is_constructible_v<typename std::iterator_traits<Iterator>::Iterator_category,
		std::input_iterator_tag>;


	template<typename Iterator, typename Distance>
	requires(!IsBidirectionalIterator<Iterator>)
	void advanceIter(Iterator& x, Distance n)
	{
		std::cout << "std::IsInputIterator" << std::endl;
		while (n > 0)
		{
			++x;
			--n;
		}
	}


	template<typename Iterator, typename Distance>
	requires(IsRandomAccessIterator<Iterator>)
	void advanceIter(Iterator& x, Distance n)
	{
		std::cout << "std::IsRandomAccessIterator" << std::endl;
		x += n;
	}

	template<typename Iterator, typename Distance>
	requires(IsBidirectionalIterator<Iterator>
	&& !IsRandomAccessIterator<Iterator>)
	void advanceIter(Iterator& x, Distance n)
	{
		std::cout << "std::bidirectional_iterator_tag" << std::endl;
		if (n > 0)
		{
			for (; n > 0; ++x, --n)
			{ //linear time 
			}
		}
		else
		{
			for (; n < 0; --x, ++n)
			{ //linear time 
			}
		}
	}
	
}
```
## 类的特化
### 一：启用/禁用类模板
类模板的偏特化可以被用来提供一个可选的，为特定模板参数进行了特化的实现，这一点和函数模板的重载很像。启用/禁用类模板的不同实现方式的方法是使用类模板偏特化。为了将EnableIf用于类模板的偏特化，需要引入一个未命名的,默认的模板参数：
```
namespace _20_4_1_
{
	template<bool, typename T = void>
	struct EnableIfT
	{

	};

	template<typename T>
	struct EnableIfT<true, T>
	{
		using Type = T;
	};

	template<bool Cond, typename T = void>
	using EnableIf = typename EnableIfT<Cond, T>::Type;

	template<typename, typename, typename = std::void_t<>>
	struct HasLessT : std::false_type {};

	template<typename T1, typename T2>
	struct  HasLessT < T1, T2, std::void_t< decltype(std::declval<T1>() < std::declval<T2>())> > : std::true_type{};

	template<typename T1,typename T2>
	constexpr bool Hasless = HasLessT<T1, T2>::value;                                     

	template<typename Key,typename Value,typename = void>
	class Dictionnary
	{
	public:
		Dictionnary()
		{
			std::cout << "1111111111" << std::endl;
		}
	private:
		std::vector<Value> data;
	};

	template<typename Key, typename Value>
	class Dictionnary<Key, Value, EnableIf<Hasless<Key, Key>>>
	{
	public:
		Dictionnary()
		{
			std::cout << "2222222222222" << std::endl;
		}

	private:
		std::map<Key, Value> data;
	};

}
```

### 二：类模板的标记派发
同样地，标记派发也可以被用于在不同的模板特化版本之间做选择。为了展示这一技术，我 们定义一个类似于之前章节中介绍的 advanceIter()算法的函数对象类型 Advance<Iterator>， 它同样会以一定的步数移动迭代器。会同时提供基本实现（用于 input iterators）和适用于双 向迭代器和随机访问迭代器的特化版本，并基于辅助萃取 BestMatchInSet（下面会讲到）为 相应的迭代器种类选择最合适的实现版本：
```
namespace _20_4_2_
{
	// 类模板的标记派发
	template<typename... Types>
	struct MatchOverloads;

	template<>
	struct MatchOverloads<>
	{
		static void match(...);
	};
	template<typename T1,typename... Rest>
	struct MatchOverloads<T1,Rest...> : public MatchOverloads<Rest...>
	{
		static T1 match(T1);
		using MatchOverloads<Rest...>::match;
	};

	template<typename T,typename ... Types>
	struct BestMatchInSetT
	{
		using Type = decltype(MatchOverloads<Types...>::match(std::declval<T>()));
	};
	
	template<typename T, typename ... Types>
	using BestMatchInSet = typename BestMatchInSetT<T, Types...>::Type;

	template<typename Iterator,
		typename Tag = BestMatchInSet<typename std::iterator_traits<Iterator>::iterator_category,
		std::input_iterator_tag,
		std::bidirectional_iterator_tag,
		std::random_access_iterator_tag>>
	class Advance;

	template<typename Iterator>
	class Advance<Iterator,std::input_iterator_tag>
	{
	public:
		using DefferenceType = typename std::iterator_traits<Iterator>::difference_type;

		void operator()(Iterator& x, DefferenceType n) const
		{
			std::cout << "std::input_iterator_tag" << std::endl;
			while (n > 0)
			{
				++x;
				--n;
			}
		}

	private:

	};

	template<typename Iterator>
	class Advance<Iterator, std::bidirectional_iterator_tag>
	{
	public:
		using DefferenceType = typename std::iterator_traits<Iterator>::difference_type;

		void operator()(Iterator& x, DefferenceType n) const
		{
			std::cout << "std::bidirectional_iterator_tag" << std::endl;
			if (n > 0)
			{
				for (; n > 0; ++x, --n)
				{ //linear time 
				}
			}
			else
			{
				for (; n < 0; --x, ++n)
				{ //linear time 
				}
			}
		}

	private:

	};


	template<typename Iterator>
	class Advance<Iterator, std::random_access_iterator_tag>
	{
	public:
		using DefferenceType = typename std::iterator_traits<Iterator>::difference_type;

		void operator()(Iterator& x, DefferenceType n) const
		{
			std::cout << "std::random_access_iterator_tag" << std::endl;
			x += n;
		}

	private:

	};

	template<typename Iterator,typename Distance>
	void advanceIter(Iterator& x, Distance n)
	{
		Advance<Iterator> a;
		a(x, n);
	}
}
```
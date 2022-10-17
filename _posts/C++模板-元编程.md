---
title: C++模板-元编程
date: 2022-10-17 08:49:51
tags:
---

元编程的意思是“编写一个程序”。也就是说，我们构建了可以被编程系统用来产生新代码的代码，而且新产生的代码实现了我们想要的功能。   
为什么需要元编程呢？很明显作为一个“懒惰”的程序员我们想要尽可能少的“付出”，换取尽可能多的功能，其中“付出”可以用代码长度、维护成本之类的事情来衡量。元编程的特性之一是在
编译期间（at translation time，翻译是否准确？）就可以进行一部分用户定义的计算。
目前元编程分为：1.值元编程;2.类型元编程;3.混合元编程;三种。
## 值元编程
得益于C++14标准的constexpr关键字，我们可以像普通代码一样实现编译期计算，示例如下：
```
namespace _23_1_1_
{
	template <typename T>
	constexpr T sqrt(T x)
	{
		if (x <= 1)
		{
			return x;
		}
		T lo = 0, hi = x;
		for (;;)
		{
			auto mid = (hi + lo) / 2, midSquared = mid * mid;
			if (lo + 1 >= hi || midSquared == x)
			{
				return mid;
			}
			if (midSquared < x)
			{
				lo = mid;
			}
			else
			{
				hi = mid;
			}
		}
	}
}

int main()
{
    std::cout << _23_1_1_::sqrt(17) << std::endl;
}
```
虽然这个算法在运行期间并不是最高效的，但是由于是编译期计算，绝对的效率并没有移植性重要。而且并没有用什么“模板魔法”。
## 类型元编程
之前我们在学习类型萃取的时候，已经遇到过一种类型元编程，它接受一个类型作为输入并输出一个新的类型。比如RemoveReferenceT类模板，但是这只是很初级的用法，接下来我们实现一个通过<b>递归模板实例化</b>--这也是主要的基于模板的元编程手段。例子如下：
```
namespace _23_1_2_
{
	template<typename T>
	struct RemoveAllExtentsT
	{
		using Type = T;
	};

	template<typename T,std::size_t SZ>
	struct RemoveAllExtentsT<T[SZ]>
	{
		using Type = typename RemoveAllExtentsT<T>::Type;
	};

	template<typename T>
	struct RemoveAllExtentsT<T[]>
	{
		using Type = typename RemoveAllExtentsT<T>::Type;
	};

	template<typename T>
	using RemoveAllExtents = typename RemoveAllExtentsT<T>::Type;
}

int main()
{
    //元函数通过偏特化来匹配高层次的数组，递归地调用自己并最终完成任务。
    using namespace _23_1_2_;
    std::cout << typeid(RemoveAllExtents<int[]>).name() << std::endl;
    std::cout << typeid(RemoveAllExtents<int[5][10]>).name() << std::endl; //递归
    std::cout << typeid(RemoveAllExtents<int[][10]>).name() << std::endl; // 递归
    std::cout << typeid(RemoveAllExtents<int(*)[]>).name() << std::endl; //还是int(*)[]
}
```
## 混合元编程
我们可以在编译期间，以编程的方式组合一些有运行期效果的代码。我们称之为混合元编程。举一个简单的例子，计算俩个std::array的点乘结果。如果用以上的知识我们一般设计如下代码：
```
namespace _23_1_3_1_
{
	template<typename T, std::size_t N>
	auto dotProduct(std::array<T, N> const& x, std::array<T, N> const& y)
	{
		T result{};
		for (size_t i = 0; i < N; i++)
		{
			result += x[i] * y[i];
		}
		return result;
	}
}

int main()
{
    std::array<int,4> ia{ 1,2,3,4 };
	std::array<int, 4> ib{ 10,10,10,10 };
	std::cout << "_23_1_3_1_ ia dot ib = " << _23_1_3_1_::dotProduct(ia, ib) << std::endl;
}

```
下面我们实现一个不使用for循环的版本
```

namespace _23_1_3_2_
{
	template<typename T, std::size_t N>
	struct DotProductT
	{
		static inline T result(auto&& a, auto&& b)
		{
			return (*a) * (*b) + DotProductT<T, N - 1>::result(a + 1, b + 1);
		}
	};

	template<typename T>
	struct DotProductT<T,0>
	{
		static inline T result(auto&& a, auto&& b)
		{
			return T{};
		}
	};

	template<typename T, std::size_t N>
	auto dotProduct(std::array<T, N> const& x, std::array<T, N> const& y)
	{
		return DotProductT<T, N>::result(x.begin(), y.begin());
	}
}
int main()
{
    std::array<int,4> ia{ 1,2,3,4 };
	std::array<int, 4> ib{ 10,10,10,10 };
	std::cout << "_23_1_3_2_ ia dot ib = " << _23_1_3_2_::dotProduct(ia, ib) << std::endl;
}
```
新的实现将计算放在了类模板DotProductT中。这样做的目的是为了使用类模板的递归实例化计算结果，并能够通过部分特例化终止递归。这段代码的主要特点是它融合了编译期计算（这里通过递归的模板实例化实现，这决定了代
码的整体结构）和运行时计算（通过调用 result()，决定了具体的运行期间的效果）。

## 反射元编程的维度
值元编程和类型元编程采用了明显不同的方式来驱动计算。值元编程也可以通过模板的递归实例化来实现。来看下实现方式：
```

namespace _23_2_
{
	template<long long N, long long LO = 1, long long HI = N>
	struct Sqrt
	{
		static constexpr auto mid = (LO + HI + 1) / 2;
		static constexpr auto value = (N < mid* mid) ? Sqrt<N, LO, mid - 1>::value : Sqrt<N, mid, HI>::value;
	};

	template<long long N, long long M>
	struct Sqrt<N,M,M>
	{
		static constexpr auto value = M;
	};
}

int main()
{
    std::cout << _23_2_::Sqrt<100>::value << std::endl;
}
```
我们已经看到元编程的计算引擎可以有多种潜在的选择。但是计算不是唯一的一
个我们应该在其中考虑相关选项的维度。一个综合的元编程解决方案应该在如下 3 个维度中
间做选择：
* 计算维度
* 反射维度
* 生成维度

反射维度指的是以编程的方式检测程序特性的能力。生成维度指的是为程序生成额外代码的
能力。   
但是实例化模板的成本并不低廉。以上代码如果计算数值特别大，会产生很多模板实例。这会导致我们编译时间加长。

<b>一个模板元程序可能会包含以下内容：</b>
* 状态变量：模板参数
* 循环结构：通过递归实现
* 执行路径选择：通过条件表达式或者偏特例化实现
* 整数运算
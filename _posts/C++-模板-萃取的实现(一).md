---
title: C++-模板-萃取的实现(一)
date: 2022-08-31 15:52:18
tags:
---
萃取是c++模板编程中很重要的概念，可以帮我们实现很多神奇的功能，我们将来的几章会从stl标准库的萃取取几个重要的例子，来分析萃取的用法。

## 对一个序列求和
### 固定的萃取
先让我们看一个例子如下：
```
namespace _19_1_1_
{
	template<typename T>
	T accum(T const* beg, T const* end)
	{
		T total{};
		while (beg != end)
		{
			total += *beg;
			++beg;
		}
		return total;
	}
}
```
该例子唯一有些微妙的地方是，如何创建一个类型正确的零值（zero value）来作为求和的起始值。此处我们使用值初始化（{}符号）。局部的total要么被其默认值初始化，要么被零初始化。如下代码我们使用的例子：
```
int main()
{
    int num[] = { 1,2,3,4,5 };
	std::cout << "the average value of the integer values is " << _19_1_1_::accum(num, num + 5) / 5 << std::endl;
	char name[] = "template";
	int lenght = sizeof(name) - 1;
	std::cout << "the average value of the char in " << name << " is " << _19_1_1_::accum(name, name + lenght) / lenght << std::endl;
    return 0;
}
```
int 类型的数组可以正常工作，但是char类型出现错误。因为char是范围较小类型。相加超出char最大值。我们可以使用如下代码解决这个问题：
```
accum<int>(name,name+5);
```
但是这回强制要求开发者记住烦人的基础知识，我们只想调用函数实现，我可以通过萃取模板建立accum()的T和与之对应的应该存储返回值的类型之间某种联系。代码如下：
```
namespace _19_1_2_
{
	template<typename T>
	struct AccumulationTraits;

	template<>
	struct AccumulationTraits<char>
	{
		using Acct = int;
	};

	template<>
	struct AccumulationTraits<short>
	{
		using Acct = int;
	};

	template<>
	struct AccumulationTraits<int>
	{
		using Acct = long;
	};

	template<typename T>
	auto accum(T  const* beg, T const* end)
	{
		using AccT = typename AccumulationTraits<T>::Acct;
		AccT total{};
		while (beg != end)
		{
			total += *beg;
			++beg;
		}
		return total;
	}
}

```
这样我们就完成一个基础的萃取模板，但是还是会遇到问题。我们娓娓道来。

### 值萃取
目前为止我们主要是萃取类型信息，除了这个之外，我们还可以将常量以及其他数值类和一个类型关联起来。
之前代码我们初始化total使用值初始化。很显然，这并不能保证会生成一个合适的值，因为AccT可能根本没有默认构造函数。   
萃取这时候又可以来救场了。对于这个例子我们为AccumulationTraits添加一个新值萃取。代码如下：
```
namespace _19_1_2_
{
	template<typename T>
	struct AccumulationTraits;

	template<>
	struct AccumulationTraits<char>
	{
		using Acct = int;
		static Acct const zero = 0;
	};

	template<>
	struct AccumulationTraits<short>
	{
		using Acct = int;
		static Acct const zero = 0;
	};

	template<>
	struct AccumulationTraits<int>
	{
		using Acct = long;
		static Acct const zero = 0;
	};

	template<typename T>
	auto accum(T  const* beg, T const* end)
	{
		using AccT = typename AccumulationTraits<T>::Acct;
		AccT total = AccumulationTraits<T>::zero;
		while (beg != end)
		{
			total += *beg;
			++beg;
		}
		return total;
	}
}
```
在这个例子中，新的萃取提供了一个可以在编译期间计算的zero成员。但是这并不完整，因为c++只允许我们在类中对一个整形或者枚举类型的static const数据成员进行初始化。如果添加constexpr的static 数据成员会好一些，允许我们对float类型以及其他字面类型进行初始化，但是无论const还是constexpr都禁止对非字面类型（自定义类）进行这一类初始化，例如一下代码就编译不过：
```
struct BigInt
{
	int value = 0;
	BigInt operator+(const BigInt& otherr) const
	{
		return { value + otherr.value };
	}
	BigInt operator+=(const BigInt& otherr)
	{
		value += otherr.value;
		return *this;
	}
	BigInt operator/(const int& num) const
	{
		return { value / num };
	}
};
template<>
	struct AccumulationTraits<BigInt>
	{
		using Acct = BigInt;
		static Acct const zero = BigInt{}; // 不允许这么使用
	};
```
解决办法有两种
1. 一种c++17之后我们可以使用inline解决代码如下：
```
namespace _19_1_2_17_
{
	struct BigInt
	{
		int value = 0;

		BigInt(int v) : value(v) {}

		BigInt operator+(const BigInt& otherr) const
		{
			return { value + otherr.value };
		}

		BigInt operator+=(const BigInt& otherr)
		{
			value += otherr.value;
			return *this;
		}

		BigInt operator/(const int& num) const
		{
			return { value / num };
		}
	};

	template<typename T>
	struct AccumulationTraits;

	template<>
	struct AccumulationTraits<char>
	{
		using Acct = int;
		inline static Acct const zero = { 0 };
	};

	template<>
	struct AccumulationTraits<short>
	{
		using Acct = int;
		inline static Acct const zero = { 0 };
	};

	template<>
	struct AccumulationTraits<int>
	{
		using Acct = long;
		inline static Acct const zero = { 0 };
	};

	template<>
	struct AccumulationTraits<float>
	{
		using Acct = double;
		inline static Acct const zero = { 0.f };
	};

	template<>
	struct AccumulationTraits<BigInt>
	{
		using Acct = BigInt;
		inline static Acct const zero = Acct{ 0 };
	};

	template<typename T>
	auto accum(T  const* beg, T const* end)
	{
		using AccT = typename AccumulationTraits<T>::Acct;
		AccT total = AccumulationTraits<T>::zero;
		while (beg != end)
		{
			total += *beg;
			++beg;
		}
		return total;
	}
}
```
2. c++17之前我们静态constexpr函数代码如下：
```
namespace _19_1_2_1411_
{
	struct BigInt
	{
		int value = 0;

		BigInt operator+(const BigInt& otherr) const
		{
			return { value + otherr.value };
		}

		BigInt operator+=(const BigInt& otherr)
		{
			value += otherr.value;
			return *this;
		}

		BigInt operator/(const int& num) const
		{
			return { value / num };
		}
	};

	template<typename T>
	struct AccumulationTraits;

	template<>
	struct AccumulationTraits<char>
	{
		using Acct = int;
		static constexpr Acct zero()
		{
			return 0;
		}
	};

	template<>
	struct AccumulationTraits<short>
	{
		using Acct = int;
		static constexpr Acct zero()
		{
			return 0;
		}
	};

	template<>
	struct AccumulationTraits<int>
	{
		using Acct = long;
		static constexpr Acct zero()
		{
			return 0;
		}
	};

	template<>
	struct AccumulationTraits<unsigned int>
	{
		using Acct = unsigned long;
		static constexpr Acct zero()
		{
			return 0;
		}
	};

	template<>
	struct AccumulationTraits<float>
	{
		using Acct = double;
		static constexpr Acct zero()
		{
			return 0;
		}
	};

	template<>
	struct AccumulationTraits<BigInt>
	{
		using Acct = BigInt;
		static constexpr Acct zero()
		{
			return BigInt{ 0 };
		}
	};

	template<typename T>
	auto accum(T  const* beg, T const* end)
	{
		using AccT = typename AccumulationTraits<T>::Acct;
		AccT total = AccumulationTraits<T>::zero();
		while (beg != end)
		{
			total += *beg;
			++beg;
		}
		return total;
	}
}
```
## 参数化萃取

以上用法我们是固定在accum()里，一旦定义了解耦合萃取，我们是不可以替换的，为了解决这个问题，我可以引入一个新模板参数AT，其默认值由萃取模板决定。代码如下：
```
template<typename T, typename AT = AccumulationTraits<T>>
	auto accum(T  const* beg, T const* end)
	{
		using AccT = typename AT::Acct;
		AccT total = AT::zero;
		while (beg != end)
		{
			total += *beg;
			++beg;
		}
		return total;
	}
```
采用这种方式，一部分用户可以忽略掉额外模板参数，而对于那些有着特殊需求的用户，他 们可以指定一个新的类型来取代默认类型。但是可以推断，大部分的模板用户永远都不需要 显式的提供第二个模板参数，因为我们可以为第一个模板参数的每一种（通过推断得到的） 类型都配置一个合适的默认值

## 策略(Policies)
到目前为止我们的accum()依赖类型的operator+=()函数，如果某个类型不支持total+=*beg 这一类型运算我们就会出现问题。解决这一问题办法我们通过引入策略方法。代码如下：
```
class SumPolicy
	{
	public:
		template<typename T1, typename T2>
		static void accumulate(T1& total, T2 const& value)
		{
			total += value;
		}
	};

	struct BigInt
	{
		int value = 0;

		BigInt(int v) : value(v) {}

		BigInt operator+(const BigInt& otherr) const
		{
			return { value + otherr.value };
		}

		BigInt operator+=(const BigInt& otherr)
		{
			value += otherr.value;
			return *this;
		}

		BigInt operator/(const int& num) const
		{
			return { value / num };
		}
	};

	template<typename T>
	struct AccumulationTraits;

	template<>
	struct AccumulationTraits<char>
	{
		using Acct = int;
		inline static Acct const zero = { 0 };
	};

	template<>
	struct AccumulationTraits<short>
	{
		using Acct = int;
		inline static Acct const zero = { 0 };
	};

	template<>
	struct AccumulationTraits<int>
	{
		using Acct = long;
		inline static Acct const zero = { 0 };
	};

	template<>
	struct AccumulationTraits<float>
	{
		using Acct = double;
		inline static Acct const zero = { 0.f };
	};

	template<>
	struct AccumulationTraits<BigInt>
	{
		using Acct = BigInt;
		inline static Acct const zero = Acct{ 0 };
	};

	template<typename T, typename Policy = SumPolicy, typename AT = AccumulationTraits<T>>
	auto accum(T  const* beg, T const* end)
	{
		using AccT = typename AT::Acct;
		AccT total = AT::zero;
		while (beg != end)
		{
			Policy::accumulate(total, *beg);
			++beg;
		}
		return total;
	}
```
在这种情况下，我们可能会认识到，累积循环的初始值应该是累计策略的一部分。这个策略 可以使用也可以不使用其 zero()萃取。其它一些方法也应该被记住：不是所有的事情都要用 萃取和策略才能够解决的。比如，C++标准库中的 std::accumulate()就将其初始值当作了第三 个参数。
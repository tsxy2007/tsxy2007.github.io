---
title: C++-模板-桥接static和dynamic多态
date: 2022-10-12 21:20:18
tags:
---
## 函数对象，指针以及std::function<>
### 1.函数对象
在给模板定制化行为的时候，函数对象会比较有用。比如，下面的函数模板列举了从0到某个值之间所有的整数，并将每个值都提供给了一个已有的函数对象f：
```
namespace _22_1_1_
{
	template<typename F>
	void forUpTo(int n, F f)
	{
		for (int i = 0; i < n; i++)
		{
			f(i);
		}
	}

	void printInt(int i)
	{
		std::cout << i << ' ';
	}
}

int main()
{
	using namespace _22_1_1_;
	std::vector<int> values;
	forUpTo(5, [&values](int i) {
		values.push_back(i);
		});
	forUpTo(5, printInt);
	std::cout << std::endl;
	return 0;
}
```
forUpTo函数模板适用于所有函数对象，包括lambda,函数指针，以及任意实现了核屎的operator()运算符或者可以转换为一个函数指针或引用的类,但是有个缺点就是每一次对forUpTo()的使用都很有可能产生一个不同的函数模板实例。
### 2.函数指针
缓解代码量增加的一个办法是将函数模板转变为非模板形式，这样就不用实例化。
```
namespace _22_1_3_
{
	void forUpTo(int n, void(*f)(int))
	{
		for (int i = 0; i < n; i++)
		{
			f(i);
		}
	}

	void printInt(int i)
	{
		std::cout << i << ' ';
	}
}
int main()
{
	FPrint Print("第22章 指针");
	using namespace _22_1_3_;
	std::vector<int> values{ 1,2,3,4,5 };

	/*auto func = [&values](int i) {
		values.push_back(i);
	}
	forUpTo(5, &func); error*
	forUpTo(5, printInt);
	std::cout << std::endl;
}
```
但是，虽然在给其传递 printInt()的时候该方式可以正常工作，给其传递 lambda 却会导致错
误
### 3.std::functional<>
标准库中的类模板 std::functional<>则可以用来实现另一种类型的 forUpTo()
```

namespace _22_1_2_
{
	void forUpTo(int n, std::function<void(int)> f)
	{
		for (int i = 0; i < n; i++)
		{
			f(i);
		}
	}

	void printInt(int i)
	{
		std::cout << i << ' ';
	}
}

int main()
{
	using namespace _22_1_2_;
	std::vector<int> values;
	forUpTo(5, [&values](int i) {
		values.push_back(i);
		});
	forUpTo(5, printInt);
	std::cout << std::endl;
	return 0;
}
```
Std::functional<>的模板参数是一个函数类型，该类型体现了函数对象所接受的参数类型以及
其所需要产生的返回类型，非常类似于表征了参数和返回类型的函数指针
这一形式的 forUpTo()提供了 static 多态的一部分特性：适用于一组任意数量的类型（包含函
数指针，lambda，以及任意实现了适当 operator()运算符的类），同时又是一个只有一种实
现的非模板函数。为了实现上述功能，它使用了一种称之为类型消除（type erasure）的技
术，该技术将 static 和 dynamic 多态桥接了起来。   
Std::functional<>类型是一种高效的、广义形式的 C++函数指针，提供了与函数指针相同的基
本操作：
* 在调用者对函数本身一无所知的情况下，可以被用来调用该函数。
* 可以被拷贝，move 以及赋值。
* 可以被另一个（函数签名一致的）函数初始化或者赋值。
* 如果没有函数与之绑定，其状态是“null”。

与 C++函数指针不同的是，std::functional<>还可以被用来存储 lambda，以及其它任意
实现了合适的 operator()的函数对象，所有这些情况对应的类型都可能不同。
## 自定义FunctionPtr
### 1.基本接口
FunctionPtr的接口非常直观的提供了构造,拷贝,move,析构,初始化,以及从任意函数对象赋值，还有就是要能够调用其底层的函数对象。接口中最有意思的一部分是如何在
一个类模板的偏特化中对其进行完整的描述，该偏特化将模板参数（函数类型）分解为其组
成部分（返回类型以及参数类型）：
```
template <typename Signature>
	class FunctionPtr;

	template<typename R,typename... Args>
	class FunctionPtr<R(Args...)>
	{
	private:
		FunctionBridge<R, Args...>* bridge;
	public:
		FunctionPtr() : bridge(nullptr)
		{

		}

		FunctionPtr(FunctionPtr const& other) :bridge(nullptr);

		FunctionPtr(FunctionPtr&& other) : bridge(other.bridge)
		{
			other.bridge = nullptr;
		}

		// 利用多态类型擦除
		template<typename F>
		FunctionPtr(F&& f);

		FunctionPtr& operator=(FunctionPtr const& other)
		{
			FunctionPtr tmp(other);
			swap(*this, tmp);
			return *this;
		}

		FunctionPtr& operator=(FunctionPtr&& other)
		{
			delete bridge;
			bridge = other.bridge;
			other.bridge = nullptr;
			return *this;
		}

		template<typename F>
		FunctionPtr& operator=(F&& f)
		{
			FunctionPtr tmp(std::forward<F>(f));
			swap(*this, tmp);
			return *this;
		}

		~FunctionPtr()
		{
			delete bridge;
		}

		friend void swap(FunctionPtr& fp1, FunctionPtr& fp2)
		{
			std::swap(fp1.bridge, fp2.bridge);
		}

		explicit operator bool() const
		{
			return bridge == nullptr;
		}

		R operator()(Args... args) const
		{
			return bridge->invoke(std::forward<Args>(args)...);
		}

	};
```
### 2.桥接接口
FunctionBridge类模板负责持有以及维护底层的函数对象,它被实现为一个抽象基类,为functionptr的动多态打下基础:
```
template<typename R, typename... Args>
	class FunctionBridge
	{
	public:
		virtual ~FunctionBridge() {}
		virtual FunctionBridge* clone() const = 0;
		virtual R invoke(Args... args) const = 0;
		virtual bool equals(FunctionBridge const* fb) const = 0;
	private:

	};
```
我们在functionptr把一下两个函数补全
```
FunctionPtr(FunctionPtr const& other) :bridge(nullptr)
		{
			if (other.bridge)
			{
				bridge = other.bridge->clone();
			}
		}

		
		R operator()(Args... args) const
		{
			return bridge->invoke(std::forward<Args>(args)...);
		}
```
### 3.类型擦除(Type Erasure)
FunctorBridge 的每一个实例都是一个抽象类，因此其虚函数功能的具体实现是由派生类负责
的。为了支持所有可能的函数对象（一个无界集合），我们可能会需要无限多个派生类。幸
运的是，我们可以通过用其所存储的函数对象的类型对派生类进行参数化：
```
//类型擦除
	template<typename Functor, typename R, typename... Args>
	class SpecificFunctorBridge : public FunctionBridge<R, Args...>
	{
		Functor functor;
	public:
		template<typename FunctorFwd>
		SpecificFunctorBridge(FunctorFwd&& infunctor)
			:functor(std::forward<FunctorFwd>(infunctor))
		{

		}

		virtual SpecificFunctorBridge* clone() const override
		{
			return new SpecificFunctorBridge(functor);
		}

		virtual R invoke(Args ... args) const override
		{
			return functor(std::forward< Args >(args)...);
		}

		virtual bool equals(FunctionBridge<R, Args...> const* fb) const override
		{
			return false;
		}
	};
```
每一个 SpecificFunctorBridge 的实例都存储了函数对象的一份拷贝（类型为 Functor），它可
以被调用，拷贝（通过 clone()），以及销毁（通过隐式调用析构函数）。SpecificFunctorBridge
实例会在 FunctionPtr 被实例化的时候顺带产生，FunctionPtr 的剩余实现如下：
```
// 利用多态类型擦除
		template<typename F>
		FunctionPtr(F&& f) : bridge(nullptr)
		{
			using Functor = std::decay_t<F>;
			using Bridge = SpecificFunctorBridge< Functor, R, Args... >;
			bridge = new Bridge(std::forward<F>(f)); // 生成brige
		}
```

注意，此处由于 FunctionPtr 的构造函数本身也被函数对象类型模板化了，该类型只为
SpecificFunctorBridge 的特定偏特化版本（以 Bridge 类型别名表述）所知。一旦新开辟的 Bridge
实例被赋值给数据成员 bridge，由于从派生类到基类的转换（Bridge* --> FunctorBridge<R, Args...>*），特定类型 F 的额外信息将会丢失。类型信息的丢失，解释了为什么名称“类型
擦除”经常被用于描述用来桥接 static 和 dynamic 多态的技术
接下来我们测试下我们的代码：
```
void forUpTo(int n, FunctionPtr<void(int)> f)
	{
		for (int i = 0; i < n; i++)
		{
			f(i);
		}
	}

	void printInt(int i)
	{
		std::cout << i << ' ';
	}
int main()
{
	FPrint Print("第22章 自定义Function");
			using namespace _22_2_;
			std::vector<int> values;

			auto func = [&values](int i) {
				values.push_back(i);
			};

			forUpTo(5, func);

			forUpTo(5, printInt);
			std::cout << std::endl;
}
```
### 4.完善
上述代码基本完成但是有个小小的缺点，就是我们并没有提供判等的操作代码如下：
```
virtual bool equals(FunctionBridge<R, Args...> const* fb) const override
		{
			if (auto specFb = dynamic_cast<SpecificFunctorBridge const*>(fb))
			{
				return functor == specFb->functor;
			}
			return false;
		}
```
然后我们在FunctionPtr里实现operator==和operator!=操作：
```
friend bool operator==(FunctionPtr const& f1, FunctionPtr const& f2)
		{
			if (!(&f1) || !(&f2))
			{
				return !(&f1) && !( & f2);
			}
			return f1.bridge->equals(f2.bridge);
		}

		friend bool operator!=(FunctionPtr const& f1, FunctionPtr const& f2)
		{
			return !(f1 == f2);
		}
```
以上代码基本完成判等的任务，但是有个小问题，就是如果 FunctionPtr 被一个没有实现合适
的 operator==的函数对象（比如 lambdas）赋值，或者是被这一类对象初始化，那么这个程
序会遇到编译错误。这可能会很让人意外，因为 FunctionPtrs 的 operator==可能根本就没有
被使用，却遇到了编译错误。而诸如 std::vector 之类的模板，只要它们的 operator==没有被
使用，它们就可以被没有相应 operator==的类型实例化。   
我们可以使用SFNINE技术实现判等
```
template<typename T>
	class IsEqualityComparable
	{
	private:
		static void* conv(bool);

		template<typename U>
		static std::true_type test(decltype(conv(std::declval<U const&>() == std::declval<U const&>())),
			decltype(conv(!(std::declval<U const&>() == std::declval<U const&>()))));

		template<typename U>
		static std::false_type test(...);
	public:
		static constexpr bool value = decltype(test<T>(nullptr, nullptr))::value;
	};

	template<typename T,bool EqComparable = IsEqualityComparable<T>::value>
	struct TryEquals
	{
		static bool equals(T const& x1, T const& x2)
		{
			return x1 == x2;
		}
	};

	class NotEqualityComparable : public std::exception 
	{
	}; 
	
	template<typename T> 
	struct TryEquals<T, false>
	{
		static bool equals(T const& x1, T const& x2)
		{
			return false;
			//throw NotEqualityComparable();
		}
	};
```
然后我们修改SpecificFunctorBridge的equals函数:
```
virtual bool equals(FunctionBridge<R, Args...> const* fb) const override
		{
			if (auto specFb = dynamic_cast<SpecificFunctorBridge const*>(fb))
			{
				//return functor == specFb->functor;
				return TryEquals<Functor>::equals(functor, specFb->functor);
			}
			return false;
		}
```
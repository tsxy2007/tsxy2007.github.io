---
title: C++-模板-萃取的实现(三)
date: 2022-09-05 09:52:18
tags:
---

# <center>基于SFINAE的萃取
SFINAE会将在模板参数推断过程中，构造无效类型和表达式的潜在错误转换成简单的推断错误，这样就允许重载解析继续在其他待选项中间做选择。虽然SFINAE最开始是被用来避免与函数重载相关的伪错误，我们也可以用它在编译期间判断特定类型和表达式的有效性。基于SFINAE的两个主要技术是：
1. 用SFINAE排除某些重载函数
2. 用SFINAE排除某些偏特化

## 一：用SFINAE排除某些重载函数
### 方法一：
我们用一个例子来说明基于SFINAE用于排除某些重载函数，判断一个类型是否是默认可构造的，对于可以默认构造的类型，就可以不同过值初始化来创建对象。
```
namespace _19_4_1_
{
	template<typename T>
	struct IsDefaultConstructibleT
	{
	private:
		template<typename U, typename = decltype(U())>
		static char test(void*);

		template<typename>
		static long test(...);
	public:
		static constexpr bool value = _19_3_3_2_::IsSameV<decltype(test<T>(nullptr)), char>;
	};
}
```
通过函数重载实现一个基于SFINAE的萃取的常规方式声明两个返回值类型不同的同名（test()）重载函数模板：
```
	static char test(void*);
	static long test(...);
```
第一个重载函数只有在所需的检查成功时才会被匹配到，第二个重载函数是用来应急的：他会匹配任意调用，但是由于它通过"..."进行匹配的，因此其它任何匹配优先级都比它高。
返回值value的具体值取决于最终选择了哪个test函数：
```
static constexpr bool value = _19_3_3_2_::IsSameV<decltype(test<T>(nullptr)), char>;
```
现在，到了该处理我们所需要检测的属性的时候了。目标是只有当我们所关心的测试条件被 满足的时候，才可以使第一个 test()有效。在这个例子中，我们想要测试的条件是被传递进 来的类型 T 是否是可以被默认构造的。为了实现这一目的，我们将 T 传递给 U，并给第一个 test()声明增加一个无名的（dummy）模板参数，该模板参数被一个只有在这一转换有效的 情况下才有效的构造函数进行初始化。在这个例子中，我们使用的是只有当存在隐式或者显 式的默认构造函数 U()时才有效的表达式。我们对 U()的结果施加了 deltype 操作，这样就可 以用其结果初始化一个类型参数了。   
第二个模板参数不可以被推断，因为我们不会为之传递任何参数。而且我们也不会为之提供 显式的模板参数。因此，它只会被替换，如果替换失败，基于 SFINAE，相应的 test()声明会 被丢弃掉，因此也就只有应急方案可以匹配相应的调用。  
需要注意的是：
```
namespace _19_4_1_
{
	template<typename T>
	struct IsDefaultConstructibleT
	{
	private:
		template<typename T, typename = decltype(T())>
		static char test(void*);

		template<typename>
		static long test(...);
	public:
		static constexpr bool value = _19_3_3_2_::IsSameV<decltype(test<T>(nullptr)), char>;
	};
}
```
这种做法并不可取，因为对于任意的T，所有模板参数为T的成员函数都会被执行模板参数替换，因此对于一个不可以被默认构造的类型，这些代码会遇到编译错误，而不是忽略掉第一个test()。通过将类模板的模板参数 T 传递给函数模板的参数 U，我们就只为第二个 test()的 重载创建了特定的 SFINAE 上下文。
### 方法二：
另一种方法是基于返回值类型不同的重载函数模板，基于返回值类型的大小判定使用了哪一个重载函数代码如下：
```
template<typename T>
	struct IsDefaultConstructibleT1
	{
	private:
		using Size1T = char(&)[1]; 
		using Size2T = char(&)[2];
		template<typename U, typename = decltype(U())>
		static Size1T test(void*);

		template<typename>
		static Size2T test(...);
	public:
		static constexpr bool value = (sizeof(test<T>(nullptr)) == sizeof(Size1T));
	};
```
## 二: 用 SFINAE 排除偏特化
我们用基于用 SFINAE 排除偏特化实现以上例子：
```
namespace _19_4_2_
{
	template<typename, typename = std::void_t<>>
	struct IsDefaultConstructibleT : std::false_type
	{

	};

	template<typename T>
	struct IsDefaultConstructibleT<T, std::void_t<decltype(T())>> : std::true_type
	{

	};
}
```
> 此处比较有意思的地方是，第二个模板参数设定为一个辅助别名模板std::void_t。这使得我们能够定义各种使用任意数量编译期类型构造的偏特化。

## 三: 将泛型 Lambdas 用于 SFINAE
无论使用哪种技术，在定义萃取的时候总是需要用到一些样板代码：重载并调用两个test()成员函数，或者实现多个偏特化。接下来我们会展示在c++17中，如何通过指定一个泛型lambda来做条件测试，将样板代码的数量最小化。
首先，先介绍一个两个嵌套泛型lambda表达式构造的工具:
```
namespace _19_4_3_
{
	struct Test
	{
		//Test() = delete;
		int First;
	};
	// 将泛型lambdas用于SFINAE
	template<typename F, typename... Args,
		typename = decltype(std::declval<F>()(std::declval<Args&&>()...))>
		std::true_type isValidImpl(void*);

	template<typename F, typename... Args>
	std::false_type isValidImpl(...);

	inline constexpr auto isValid = [](auto f)
	{
		return [](auto&&... args)
		{
			return decltype(isValidImpl<decltype(f), decltype(args)&&...>(nullptr)){};
		};
	};

	template<typename T>
	struct TypeT
	{
		using Type = T;
	};

	template<typename T>
	constexpr auto type = TypeT<T>();

	template<typename T>
	T valueT(TypeT<T>);
}
```
> 先从 isValid 的定义开始：它是一个类型为 lambda 闭包的 constexpr 变量。声明中必须要使 用一个占位类型（placeholder type，代码中的 auto），因为 C++没有办法直接表达一个闭包 类型。在 C++17 之前，lambda 表达式不能出现在 const 表达式中，因此上述代码只有在 C++17 中才有效。因为 isValid 是闭包类型的，因此它可以被调用，但是它被调用之后返回的依然 是一个闭包类型，返回结果由内部的 lambda 表达式生成。  
再深入讨论之前，先看一个isValid的典型用法：
```
constexpr auto isDefaultConstructible = isValid([](auto x) -> decltype((void)decltype(valueT(x))()) {});
```
> 我们已经知道 isDefaultConstructible 的类型是闭包类型，而且正如其名字所暗示的 那样，它是一个可以被用来测试某个类型是不是可以被默认构造的函数对象。也就是说， isValid 是一个萃取工厂（traits factory）：它会为其参数生成萃取，并用生成的萃取对对象进 行测试。  
> 辅助变量模板 type 允许我们用一个值代表一个类型。对于通过这种方式获得的数值 x，我们 可以通过使用 decltype(valueT(x))得到其原始类型，这也正是上面被传递给 isValid 的 lambda 所做的事情。如果提取的类型不可以被默认构造，我们要么会得到一个编译错误，要么相关 联的声明就会被 SFINAE 掉（得益于 isValid 的具体定义，我们代码中所对应的情况是后者）。  
> 可以像下面这样使用 isDefaultConstructible：
```
			std::cout << _19_4_3_::isDefaultConstructible(_19_4_3_::type<int>) << std::endl;
			std::cout << _19_4_3_::isDefaultConstructible(_19_4_3_::type<int&>) << std::endl;

```
> 为 了 理 解 各 个 部 分 是 如 何 工 作 的 ， 先 来 看 看 当 isValid 的 参 数 f 被 绑 定 到 isDefaultConstructible 的泛型 lambda 参数时，isValid 内部的 lambda 表达式会变成什 么样子。通过对 isValid 的定义进行替换，我们得到如下等价代码：
```
constexpr auto isDefaultConstructible = [](auto&&… args) 
{ 
	return decltype(isValidImpl<decltype([](auto x) -> decltype((void)decltype(valueT(x))())), 
	decltype(args)&&… > (nullptr)
	){}; 
};
```
如果我们回头看看第一个 isValidImpl()的定义，会发现它还有一个如下形式的默认模板参数：
```
decltype(std::declval<F>()(std::declval<Args&&>()…))>
```
它试图对第一个模板参数的值进行调用，而这第一个参数正是 isDefaultConstructible 定 义 中 的 lambda 的 闭 包 类 型 ， 调 用 参 数 为 传 递 给 isDefaultConstructible 的 (decltype(args)&&...)类型的值。由于 lambda 中只有一个参数 x，因此 args 就需要扩展成一个参数；在我们上面的 static_assert 例子中，参数类型伟 TypeT<int>或者 TypeT<int&>。对于 TypeT<int&>的情况，decltype(valueT(x))的结果是 int&，此时 decltype(valueT(x))()是无效的， 因此在第一个 isValidImpl()的声明中默认模板参数的替换会失败，从而该 isValidImpl()声明会 被 SFINAE 掉。这样就只有第二个声明可用，且其返回值类型为 std::false_type  
整体而言，在传递 type<int&>的时候，isDefaultConstructible 会返回 false_type。而 如果传递的是 type<int>的话，替换不会失败，因此第一个 isValidImpl()的声明会被选择，返 回结果也就是 true_type 类型的值。  

回忆一下，为了使 SFINAE 能够工作，替换必须发生在被替换模板的立即上下文（immediate context）中。在我们这个例子中，被替换的模板是 isValidImpl 的第一个声 明，而且泛型 lambda 的调用运算符被传递给了 isValid。因此，被测试的内容必须出现在 lambda 的返回类型中，而不是函数体中。   

我们的 isDefaultConstructible 萃取和之前的萃取在实现上有一些不同，主要体现在 它需要执行函数形式的调用，而不是指定模板参数。这可能是一种更为刻度的方式，但是也 可以按照之前的方式实现：  
```
template<typename T>
using IsDefaultConstructibleT = decltype(isDefaultConstructible(std::declval<T>()));
```

到目前为止，这一技术看上去好像并没有那么有竞争力，因为无论是实现中涉及的表达式还 是其使用方式都要比之前的技术复杂得多。但是，一旦有了 isValid，并且对其进行了很好的 理解，有很多萃取都可以只用一个声明实现。比如，对是否能够访问名为 first 的成员进行 测试，就非常简洁： 
```
constexpr auto hasFirst = isValid([](auto x) -> decltype((void)valueT(x).first) {}); v
```
## 四: SFINAE 友好的萃取
类型萃取应该可以在不使程序出现问题的情况下回答特定的问题。基于SFINAE的萃取解决这一难题的方式是“小心的将潜在的问题捕获进一个SFINAE上下文中”，将可能出现的错误转变成相反的结果。
比如我们之前的PlusResultT这个例子，在应对错误的时候表现的并不这么友好。回忆一下之前PlusResultT的定义：
```
template<typename T1, typename T2>
	struct PlusResultT
	{
		using Type = decltype(std::declval<T1>() + std::declval<T2>());
	};
```
在这一定义中，用到+这个运算符并没有被SFINAE保护，因此，如果程序对不支持+运算符的类型执行PlusResultT的话，那么PlusResultT计算本身就会使遇到错误，例如一下例子
```
class A
{

};
class B 
{

};
```
很显然，如果没有为数组元素定义合适的+运算符的话，使用 PlusResultT<>就会遇到错误。这里的问题并不是错误会发生在代码明显有问题的地方，而是错误会发生在对operator+进行模板参数推断的时候，在很深层次的PlusResultT<A,B>的实例化中。   
为了解决这个问题，我可以通过定义一个HasPlusT萃取，来判断给定的类型是否有个一个可用的+运算符：
```
template<typename, typename, typename = std::void_t<>>
 struct HasPlusT : std::false_type 
 {}; 
 // partial specialization (may be SFINAE’d away): 
 template<typename T1, typename T2> 
 struct HasPlusT<T1, T2, std::void_t<decltype(std::declval<T1>() + std::declval<T2> ())>> : std::true_type 
 {};
```
如果其返回结果为 true，PlusResultT 就可以使用现有的实现。否则，PlusResultT 就需要一个 安全的默认实现。对于一个萃取，如果对某一组模板参数它不能生成有意义的结果，那么最 好的默认行为就是不为其提供 Type 成员。这样，如果萃取被用于 SFINAE 上下文中（比如之 前代码中 array 类型的 operator+的返回值类型），缺少 Type 成员会导致模板参数推断出错， 这也正是我们所期望的、array 类型的 operator+模板的行为。   
下一版PlusResultT的实现就提供了上述行为：
```
namespace _19_4_4_
{
	template<typename,typename,typename = std::void_t<>>
	struct HasT : std::false_type{};

	template<typename T1, typename T2>
	struct HasT <T1,T2, std::void_t< decltype(std::declval<T1>() + std::declval<T2>())>>: std::true_type{};

	template<typename T1, typename T2,bool = HasT<T1, T2>::value>
	struct PlusResultT
	{
		using Type = decltype(std::declval<T1>() + std::declval<T2>());
	};

	template<typename T1, typename T2>
	struct PlusResultT<T1,T2,false>
	{
	};

	template<typename T1, typename T2>
	using PlusResult = typename PlusResultT<T1, T2>::Type;
}
```
作为一般的设计原则，在给定了合理的模板参数的情况下，萃取模板永远不应该在实例化阶 段出错。其实先方式通常是执行两次相关的检查： 
1. 一次是检查相关操作是否有效 
2. 一次是计算其结果
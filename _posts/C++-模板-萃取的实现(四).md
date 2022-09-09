---
title: C++-模板-萃取的实现(四)
date: 2022-09-09 08:32:18
tags:
---

## IsConvertibleT
细节很重要，因此基于SFINAE萃取的常规方法在实际中会变得更加复杂。为了展示这一复杂性，我们将定义一个能够判断一种类型是否可以转化成另外一种类型的萃取，比如当我们期望某个基类或者某一子类作为参数的时候。IsConvertibleT就可以判断其第一个类型参数是否可以转换成第二个参数：
```
namespace _19_5_1_
{
	template<typename FROM, typename TO>
	struct IsConvertibleHelper
	{
	private:
		static void aux(TO);

		template<typename F, typename = decltype(aux(std::declval<F>()))>
		static std::true_type test(void*);

		template<typename>
		static std::false_type test(...);

	public:
		using Type = decltype(test<FROM>(nullptr));
	};

	template<typename FROM, typename TO>
	struct IsConveribleT : IsConvertibleHelper<FROM, TO>::Type {};

	template <typename FROM, typename TO>
	using IsConvertible = typename IsConveribleT<FROM, TO>::Type;

	template<typename FROM, typename TO>
	constexpr bool isConvertible = IsConveribleT<FROM, TO>::value;
}
```
这里我们使用了函数重载的方法，因为我们使用一个中间函数aux(TO),这一步很重要，我需要在编译期判断FROM类型是否匹配aux函数，如果匹配返回std::true_type否者失败，编译器自动自动匹配第二个可变参数，返回false_type。
再次声明一下代码不对！！！！！
```
template<typename FROM, typename TO>
	struct IsConvertibleHelper
	{
	private:
		static void aux(TO);

		template<typename = decltype(aux(std::declval<FROM>()))>
		static std::true_type test(void*);

		template<typename>
		static std::false_type test(...);

	public:
		using Type = decltype(test<FROM>(nullptr));
	};
```
这样当成员函数模板被解析的时候，FROM和TO都已经完全确定了，因此对一组不适合做相应转换的类型，在调用test()之前就会立即触发错误。因此我需要引入成员模板函数，把FROM当作参数传递进去。如下：
```
		template<typename F,typename = decltype(aux(std::declval<F>()))>
		static std::true_type test(void*);
```
### 处理特殊情况
下面3中情况还不能被上面的IsConveribleT正确处理：
1. 向数组类型的转换要始终返回 false，但是在上面的代码中，aux()声明中的类型为 TO 的 参数会退化成指针类型，因此对于某些 FROM 类型，它会返回 true。 
2. 向指针类型的转换也应该始终返回 false，但是和 1 中的情况一样，上述实现只会将它 们当作退化后的类型。 
3. 向（被 const/volatile 修饰）的 void 类型的转换需要返回 true。但是不幸的是，在 TO 是void 的时候，上述实现甚至不能被正确实例化，因为参数类型不能包含 void 类型（而 且 aux()的定义也用到了这一参数）。

因此对于这几种情况，我们需要对它们进行额外的偏特化。但是，为所有可能的与const以及volatile的组合情况都分别进行偏特化是很不明智的。相反，我们为辅助类模板引入一个额外的模板参数：
```
namespace _19_5_2_
{
	template<typename FROM, typename TO, bool = std::is_void_v<TO> || std::is_array_v<TO> || std::is_function_v<TO>>
	struct IsConvertibleHelper
	{
		using Type = std::integral_constant<bool, std::is_void_v<TO>&& std::is_void_v<FROM>>;
	};

	template<typename FROM, typename TO>
	struct IsConvertibleHelper<FROM, TO, false>
	{
	private:
		static void aux(TO);

		template<typename F, typename = decltype(aux(std::declval<F>()))>
		static std::true_type test(void*);

		template<typename>
		static std::false_type test(...);
	public:
		using Type = decltype(test<FROM>(nullptr));
	};

	template<typename FROM, typename TO>
	struct IsConveribleT : IsConvertibleHelper<FROM, TO>::Type {};

	template <typename FROM, typename TO>
	using IsConvertible = typename IsConveribleT<FROM, TO>::Type;

	template<typename FROM, typename TO>
	constexpr bool isConvertible = IsConveribleT<FROM, TO>::value;
}

```

## 探测成员
另外一种基于SFINAE的萃取的应用是，创建一个可以判断一个给定类型T是否有名为X的成员(类型或者非类型成员)的萃取 。
### 探测类型成员(Detecting Member Types)
我们首先定义一个例子：判定给定类型T是否含有类型成员size_type的萃取 ：
```
namespace _19_6_1_
{
	template<typename, typename = std::void_t<>>
	struct HasSizeTypeT : std::false_type {};

	template<typename T>
	struct HasSizeTypeT<T, std::void_t<typename std::remove_reference_t<T>::size_type > > : std::true_type {};
}
```
这种情况下我们只需要一个条件
```
typename T::size_type
```
1. 该条件只有在T含有size_type的时候才有效，这也是我们想要做的。如果对于某个类型T,该条件无效，那么SFINAE会使偏特化实现被丢弃，我们就退回到主模板的情况。否则偏特化有效并且会被有限选取。 特别注意，typename T::size_type是无效的。<b>也就是说，该萃取所做的事情是测试我们是否能够访问类型成员 size_type。</b>
2. 里层的std::remove_reference_t是因为当我们处理引用类型时，需要我们去除引用。因为引用类型确实没有成员。
3. 注入类名字（Injected Class Names）代码如下：
```
struct size_type { };
struct Sizeable : size_type { };
static_assert(HasSizeTypeT<Sizeable>::value, "Compiler bug: Injected class name missing");
```
后面的 static_assert 会成功，因为 size_type 会将其自身的名字当作类型成员，而且这一成员 会被继承。如果 static_assert 不会成功的话，那么我就发现了一个编译器的问题。
### 探测任意类型成员
在定义HasSizeTypeT的萃取之后，我们自然回想如何将该萃取参数化，方便我们对任意名称的类型成员做探测。目前我们只能通过宏来实现这个目的，代码如下：
```
namespace _19_6_2_
{
#define DEFINE_HAS_TYPE(MemType) \
    template<typename,typename = std::void_t<>> \
    struct HasTypeT_##MemType : std::false_type{}; \
    template<typename T> \
    struct HasTypeT_##MemType<T, std::void_t<typename std::remove_reference_t<T>::MemType > > : std::true_type {};

	DEFINE_HAS_TYPE(value_type);
	DEFINE_HAS_TYPE(char_type);
}
```
### 探测非类型成员
通过简单的修改，我们测试成员和成员函数：
```
#define DEFINE_HAS_MEMBER(Member) \
        template<typename,typename = std::void_t<>>\
        struct HasMemberT_##Member : std::false_type {}; \
        template<typename T> \
        struct HasMemberT_##Member<T,std::void_t<decltype(&T::Member)>> : std::true_type{};
```
1. Member 必须能够被用来没有歧义的识别出 T 的一个成员（比如，它不能是重载成员你 函数的名字，也不能是多重继承中名字相同的成员的名字）。 
2. 成员必须可以被访问。
3. 成员必须是非类型成员以及非枚举成员（否则前面的&会无效）。
4. 如果 T::Member 是 static 的数据成员，那么与其对应的类型必须没有提供使得 &T::Member 无效的 operator&（比如，将 operator&设成不可访问的）。
#### 探测成员函数
注意，HasMember 萃取只可以被用来测试是否存在“唯一”一个与特定名称对应的成员。 如果存在两个同名的成员的话，该测试也会失败，比如当我们测试某些重载成员函数是否存 在的时候：
```
DEFINE_HAS_MEMBER(begin); 
std::cout << HasMemberT_begin<std::vector<int>>::value; // false
```
SFINAE 会确保我们不会在函数模板声明中创建非法的 类型和表达式，从而我们可以使用重载技术进一步测试某个表达式是否是病态的。 也就是说，可以很简单地测试我们能否按照某种形式调用我们所感兴趣的函数，即使该函数 被重载了，相关调用可以成功。正如 IsConvertibleT 一样，此处的关键是 能否构造一个表达式，以测试我们能否在 decltype 中调用 begin()，并将该表达式用作额外 的模板参数的默认值：
```
#define DEFINE_HAS_FUNCTION(FUNC) \
    template<typename, typename = std::void_t<>>\
    struct HasMemberT_##FUNC : std::false_type {};\
    template<typename T>\
    struct HasMemberT_##FUNC<T, std::void_t<decltype(std::declval<T>().FUNC())>> : std::true_type {};

	DEFINE_HAS_FUNCTION(begin);
```
#### 探测其它的表达式
相同的技术还可以用于其他的表达式，甚至是多个表达式的组合。比如<运算符可用：
```
template<typename, typename, typename = std::void_t<>>
	struct HasLessT : std::false_type {};

	template<typename T1, typename T2>
	struct  HasLessT < T1, T2, std::void_t< decltype(std::declval<T1>() < std::declval<T2>())> > : std::true_type{};

```
采用这种方式探测表达式有效性的萃取是很稳健的：如果表达式没有问题，它会返回 true， 而如果<运算符有歧义，被删除，或者不可访问的话，它也可以准确的返回 false。
### 用泛型Lambda探测成员
我们之前介绍了isValid lambda,提供了一种定义可以被用来测试成员的更为紧凑技术，其主要的好处是在处理名称任意的成员时，不需要使用宏。
代码如下：
```
constexpr auto hasFirst = isValid([](auto x) -> decltype((void)valueT(x).first) {});
```
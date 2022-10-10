---
title: C++-模板-模板和继承
date: 2022-10-10 10:22:18
tags:
---
## 一:空基类优化(EBCO)
C++中有的类是"空"，也就是说它们的内部表征在运行期间不占用内存。典型的情况 是那写只包含类型成员，非虚成员函数，以及静态数据成员的类。而非静态数据成员，虚函 数，以及虚基类，在运行期间则是需要占用内存的。然而即使是空的类，其所占的内存大小也不是0。
```
class EmptyClass
{

};

int main()
{
	std::cout<<"sizeof(EmptyClass) : "<< sizeof(EmptyClass)<< std::endl;
}
```
在某些平台上会打印1。
### 1.布局原则
虽然c++中没有内存占用为0的类型，但是c++标准却指出，在空class被用作基类的时候，如果不给它分配内存，并不会导致其被存储到与其他类型对象或者子对象相同的地址上，那么就可以不给它分配内存。如下程序：
```
namespace _21_1_1_1_
{
	class Empty
	{
		using Int = int;
	};

	class EmptyToo : public Empty
	{

	};

	class EmptyThree : public EmptyToo
	{

	};
}
```
如图所示三个类大小是相同
![EBCO](https://github.com/tsxy2007/tsxy2007.github.io/blob/Hexo/_posts/texture/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20221010110446.png)

## 二:CRTP
该模式最简单的C++实现方式如下：
```
namespace _21_2_1_
{
	template<typename Devrived>
	class CuriousBase
	{

	};

	class Curious : public CuriousBase<Curious>
	{

	};
}
```
上面的 CRTP 的例子使用了非依赖性基类：Curious 不是一个模板类，因此它对在依赖性基类中遇到的名称可见性问题是免疫的。但是这并不是 CRTP 的固有特征。事实上，我们同样可以使用下面的这一实现方式：
```
namespace _21_2_2_
{
	template<typename Devrived>
	class CuriousBase
	{

	};

	template<typename T>
	class CuriousTemplate : public CuriousBase<CuriousTemplate<T>>
	{

	};
}
```
一个简单的CRTP的简单应用是将其用于追踪从一个class类型实例化出了多少对象。代码如下：
```
namespace _21_2_2_1_
{
	template<typename CountedType>
	class ObjectCounter
	{
	private:
		inline static std::size_t count = 0;
	protected:
		ObjectCounter()
		{
			++count;
		}

		ObjectCounter(ObjectCounter<CountedType> const&)
		{
			++count;
		}

		ObjectCounter(ObjectCounter<CountedType>&&)
		{
			++count;
		}

		~ObjectCounter()
		{
			--count;
		}

	public:
		static std::size_t live()
		{
			return count;
		}
	};

	template<typename T>
	class FTest : public ObjectCounter<FTest<T>>
	{

	};
}

int main()
{
	using namespace _21_2_2_1_;

	FTest<int> t;
	{
		FTest<int> t1 = t;
		std::cout << "t1 live(): " << t1.live() << std::endl;
	}
	std::cout <<"t1 live(): " << t.live() << std::endl;
}
```
### <b>restricted template expansion(限制模板扩张)</b>
为了说明这一技术，假设我们有一个需要为之定义 operator ==的类模板 EqualityComparable。一个可能的 方案是将该运算符定义为类模板的成员，但是由于其第一个参数（绑定到 this 指针上的参数） 和第二个参数的类型转换规则不同（为什么？一个是指针？一个是 Arry 类型？）。由于我 们希望 operator ==对其参数是对称的，因此更倾向与将其定义为某一个 namespace 中的函 数。一种很直观的实现方式可能会像下面这样:
```
namespace _21_2_2_2_
{
	template<typename Devried>
	class EqualityComparable
	{
	public :
		friend bool operator!=(Devried const& x1, Devried const& x2)
		{
			return !(x1 == x2);
		}
	};

	class X : public EqualityComparable<X>
	{
	public:
		friend bool operator==(X const& x1, X const& x2)
		{
			return false;
		}
	};
}
```
## 三: Facades(门面模式)
为了展示Facades模式,我们为迭代器实现了一个facade,这样可以大大简化一个符合标准库要求的迭代器编写。一个迭代器类型所需要支持的接 口是非常多的。下面的一个基础版的 IteratorFacade 模板展示了对迭代器接口的要求：
```
template<typename Derived,typename Value,typename Category,
	typename Reference = Value&,typename Distance = std::ptrdiff_t>
	class IteratorFacade
	{
	public:
		using value_type = typename std::remove_const_t<Value>;
		using reference = Reference;
		using pointer = Value*;
		using difference_type = Distance;
		using iterator_category = Category;


		Derived& operator--()
		{

		}

		Derived operator--(int)
		{

		}

		reference operator[](difference_type n)
		{

		}

		Derived& operator+=(difference_type n)
		{

		}

		friend bool operator== (IteratorFacade const& lhs, IteratorFacade const& rhs)
		{
			return lhs.asDerived().equals(rhs.asDerived());
		}

		friend difference_type operator-(IteratorFacade const& lhs, IteratorFacade const& rhs)
		{
			return lhs.asDerived().measureDistance(rhs.asDerived());
		}

		friend bool operator<(IteratorFacade const& lhs, IteratorFacade const& rhs)
		{

		}

		Derived& asDerived()
		{
			return *static_cast<Derived*>(this);
		}

		Derived const& asDerived() const
		{
			return *static_cast<Derived const*>(this);
		}

		reference operator*()const
		{
			return asDerived().dereference();
		}

		Derived& operator++()
		{
			asDerived().increment();
			return asDerived();
		}

		Derived operator++(int) 
		{ 
			Derived result(asDerived()); 
			asDerived().increment(); 
			return result; 
		}
		
	};
```
以上代码为了简洁，省略了一部分声明，但是即使只是给每一个新的迭代器实现上述 代码中列出的接口，也是一件很繁杂的事情。幸运的是，可以从这些接口中提炼出一些核心 的运算符：
1. 对于所有的迭代器，都有如下运算符： 
* 解引用（dereference）：访问由迭代器指向的值（通常是通过 operator *和->）。 
* 递增（increment）：移动迭代器以让其指向序列中的下一个元素。 
* 相等（equals）：判断两个迭代器指向的是不是序列中的同一个元素。 
2. 对于双向迭代器，还有： 
* 递减（decrement）：移动迭代器以让其指向列表中的前一个元素。 
3. 对于随机访问迭代器，还有： 
* 前进（advance）：将迭代器向前或者向后移动 n 步。 
* 测距（measureDistance）：测量一个序列中两个迭代器之间的距离。
### 迭代器的适配器
我们先定义个Person类
```
struct Person
	{
		std::string firstName;
		std::string lastName;

		friend std::ostream& operator<<(std::ostream& strm, Person const& p)
		{
			return strm << p.lastName << ", " << p.firstName;
		}
	};
```
如果我们只想要Person中firstname，我们可以开发一款迭代器，通过它我们可以将底层迭代器投射到一些指向数据成员的指针：
```
template<typename Iterator,typename T>
	class ProjectionIterator : public IteratorFacade<ProjectionIterator<Iterator,T>,T,
	typename std::iterator_traits<Iterator>::iterator_category,T&,typename
	std::iterator_traits<Iterator>::difference_type>
	{
		using Base = typename std::iterator_traits<Iterator>::value_type;
		using Distance = typename std::iterator_traits<Iterator>::difference_type;
		Iterator iter;
		T Base::* member;
		//friend class IteratorFacadeAccess;
	public:
		ProjectionIterator(Iterator inIter, T Base::* inMember)
			: iter(inIter), member(inMember)
		{

		}

		T& dereference() const
		{
			return (*iter).*member;
		}

		void increment()
		{
			++iter;
		}

		bool equals(ProjectionIterator const& other) const
		{
			return iter == other.iter;
		}

		void decrement()
		{
			--iter;
		}

		Distance measureDistance( ProjectionIterator const& rhs) const
		{
			Distance d = iter - rhs.iter;
			return d;
		}
	private:

	};

	template<typename Iterator,typename Base,typename T>
	auto project(Iterator iter, T Base::* member)
	{
		return ProjectionIterator<Iterator, T>(iter, member);
	}

	int main()
	{
		using namespace _21_2_3_;
			std::vector<Person> authors = { 
				{"David", "Vandevoorde"}, 
				{"Nicolai", "Josuttis"}, 
				{"Douglas", "Gregor"}
			}; 
			std::copy(project(authors.begin(), &Person::lastName),
				project(authors.end(), &Person::lastName),
				std::ostream_iterator<std::string>(std::cout, "\n"));
	}
```
## 三: Mixins(混合?)
Mixins 是另一种可以客制化一个类型的行为但是不需要从其进行继承的方法。事实上，Mixins 反转了常规的继承方向，因为新的类型被作为类模板的基类“混合进”了继承层级中，而不 是被创建为一个新的派生类。这一方式允许在引入新的数据成员以及某些操作的时候，不需要去复制相关接口。
### 普通版本
```
namespace _21_3_1_3_
{
	template<typename... Mixins>
	class Point : public Mixins...
	{
	public:
		double x, y;
		Point() : Mixins()..., x(0.f), y(0.f){}
		Point(double ix, double iy) :Mixins()..., x(ix), y(iy){}
	};

	class Label
	{
	public:
		std::string label;
		Label():label(""){}
	};

	class Color
	{
	public:
		unsigned char red = 0, green = 0, blue = 0;
	};

	using LabelPoint = Point<Label>;
	using MyPoint = Point<Label, Color>;

	template<typename ... Mixins>
	class Polygon
	{
	public:

	private:
		std::vector<Point<Mixins...>> points;
	};
}
```
### CRTP版本
```

namespace _21_3_1_4_
{
	template<template<typename> typename... Mixins>
	class Point : public Mixins<Point<Mixins...>>...
	{
	public:
		double x, y;
		Point() :Mixins<Point>()..., x(0.f), y(0.f){}
		Point(double ix, double iy) :Mixins<Point>()..., x(ix), y(iy){}
	private:

	};

	template<typename T>
	class Label
	{
	public:
		std::string label;
		Label() :label("") {}
	};

	template<typename T>
	class Color
	{
	public:
		unsigned char red = 0, green = 0, blue = 0;
	};

	template<typename ... Mixins>
	class Polygon
	{
	public:

	private:
		std::vector<Point<Color>> cpoints;
		std::vector<Point<Label, Color>> clpoints;
	};
}
```
## 四: Named Template Arguments（命名的模板参数）
不少模板技术会有很多类型参数。我们通常会设置默认值。如下：
```
template<typename Policy1 = DefaultPolicy1, 
typename Policy2 = DefaultPolicy2, 
typename Policy3 = DefaultPolicy3, 
typename Policy4 = DefaultPolicy4> 
class BreadSlicer 
{

};
```
在使用的时候一般都是默认值，但是如果我们想指定某个模板参数为自定义就需要我们把这几个都重新设置下，而我们只想设置我们想要的参数。类似如下
```
BreadSlicer<Policy3 = custom>
```
首先我们定义一个PolicySelector模板，功能是将不同模板参数融合进一个单独的类型，而且这个类型需要用那个没有指定默认值类型去重载默认的类型别名成员。代码如下：
```
	template <typename Base, int D>
	class Discriminator : public Base {
	};

	template <typename Setter1, typename Setter2,
		typename Setter3, typename Setter4>
	class PolicySelector : public Discriminator<Setter1, 1>,
		public Discriminator<Setter2, 2>,
		public Discriminator<Setter3, 3>,
		public Discriminator<Setter4, 4> {
	};

```
注意此处对中间的 Discriminator 模板的使用。其要求不同的 Setter 类型是类似的（不能使用 多个类型相同的直接基类。而非直接基类，则可以使用和其它基类类似的类型）。正如之前提到的，我们将全部的默认值收集到基类中：
```

	class DefaultPolicy1 {};
	class DefaultPolicy2 {};
	class DefaultPolicy3 {
	public:
		static void doPrint() {
			std::cout << "DefaultPolicy3::doPrint()\n";
		}
	};
	class DefaultPolicy4 {};


	// define default policies as P1, P2, P3, P4
	class DefaultPolicies {
	public:
		typedef DefaultPolicy1 P1;
		typedef DefaultPolicy2 P2;
		typedef DefaultPolicy3 P3;
		typedef DefaultPolicy4 P4;
	};


	// class to define a usage of the default policy values
	// - avoids ambiguities if we derive from DefaultPolicies more than once
	class DefaultPolicyArgs : virtual public DefaultPolicies {
	};
```
我们也需要一些模板来重载掉那些默认的策略值：
```
template <typename Policy>
	class Policy1_is : virtual public DefaultPolicies {
	public:
		typedef Policy P1;  // overriding typedef
	};

	template <typename Policy>
	class Policy2_is : virtual public DefaultPolicies {
	public:
		typedef Policy P2;  // overriding typedef
	};

	template <typename Policy>
	class Policy3_is : virtual public DefaultPolicies {
	public:
		typedef Policy P3;  // overriding typedef
	};

	template <typename Policy>
	class Policy4_is : virtual public DefaultPolicies {
	public:
		typedef Policy P4;  // overriding typedef
	};

	// define a custom policy
	class CustomPolicy {
	public:
		static void doPrint() {
			std::cout << "CustomPolicy::doPrint()\n";
		}
	};
```
最后，我们如何使用以上代码：
```
	template <typename PolicySetter1 = DefaultPolicyArgs,
		typename PolicySetter2 = DefaultPolicyArgs,
		typename PolicySetter3 = DefaultPolicyArgs,
		typename PolicySetter4 = DefaultPolicyArgs>
	class BreadSlicer {
		using Policies = PolicySelector<PolicySetter1, PolicySetter2,
			PolicySetter3, PolicySetter4>;
		// use Policies::P1, Policies::P2, //... to refer to the various policies.
	public:
		void print() {
			Policies::P3::doPrint();
		}
		//...
	};
```
我们在main里验证以上想法：
```
int main()
{
	using namespace _21_4_;
	BreadSlicer<> bc1;
	bc1.print();

	BreadSlicer<Policy3_is<CustomPolicy> > bc2;
	bc2.print();
}
```
---
layout: post
title:  有趣的c++模板元编程11个例子
date: 2022-08-26
categories:
- c/c++
tags: [c++]
---

#### 1. 实现加一

```c++
#include <iostream>

template<int x>
struct M
{
	constexpr static int val = x + 1;
};

int main()
{
	//目标：从类模板导入元编程,实现加一
	std::cout << M<4>::val << std::endl;//输出为5，实现了加一
}
```

#### 2. 用编译期函数实现加一

```c++
#include<iostream>

constexpr int add_fun(int x) {
	return x + 1;
}

constexpr int val = add_fun(5);

int main(){
    std::cout << val << std::endl;//已经输出为6，实现了加一
}
```

#### 3. 去掉变量的引用，并加上const修饰

```c++
//need c++ 17
#include <iostream>
#include <type_traits>

template<typename T>
struct Fun
{
	using RemDef = typename std::remove_reference<T>::type;
	using type = typename std::add_const<RemDef>::type;
};

int main()
{
	//目标：去掉变量的引用，并加上const
	Fun<int&>::type x = 3;
	std::cout << std::is_const_v<Fun<int&>::type> << std::endl;//返回1,说明存在const
	std::cout << std::is_reference_v<Fun<int&>::type> << std::endl;//返回0,说明不存在引用
}
```

#### 4. 计算给定类型是否是指定长度字节

```c++
#include <iostream>
#include <type_traits>

template<typename T,int S>
struct Fun2
{
	constexpr static bool value = (sizeof(T) == S);
};
int main()
{
	//目标：计算给定类型是否是四个字节
	constexpr bool res = Fun2<int, 4>::value;
	std::cout << res << std::endl;//返回1，说明确实是四个字节
}
```

#### 5. factorial求和

```c++
#include<iostream>

template<unsigned int n>
struct factorial
{
	constexpr static  int  value = n * factorial<n - 1>::value;
};

template<>
struct factorial<0>
{
	constexpr static int value = 1;
};
int main()
{
	//目标：factorial求和
	std::cout << factorial<7>::value << std::endl;    // prints "5040"
}
```

#### 6. 将数组里的元素求和（可变模板参数求和）

```c++
#include<iostream>

template <int... inputs>
struct Accumulate {
	constexpr static int value = 0;
};

template <int CurInput,int... Inputs>
struct Accumulate <CurInput,Inputs...>{
	constexpr static int value = CurInput + Accumulate<Inputs...>::value;
};

int main()
{
	//目标：将数组里的元素求和
	constexpr static int res_accumulate = Accumulate<1,2,3,4,5,6>::value; 
	std::cout << res_accumulate << std::endl; //print"21"
}
```

#### 7. 打印类似Cont<1,2,3>的数组，打印可变模板参数

```c++
#include<iostream>

template<unsigned int... T>
struct Cont_arr {};

void print_list(Cont_arr<>) {};

template<unsigned int N, unsigned int... T>
void print_list(Cont_arr<N, T...>) {
	std::cout << N << std::endl;
	print_list(Cont_arr<T...>{});
}
int main()
{
	//目标：打印类似Cont<1,2,3>的数组
	print_list(Cont_arr<1, 2 ,3>{});//打印了 1 2 3
}

```
#### 8. 判断一个数是不是质数,以下的写法在c++98也能使用，书上抄的例子

```c++
#include <iostream>
template<unsigned p, unsigned d>
struct DoIsPrime {
	static const bool value = (p % d != 0) && DoIsPrime<p, d - 1>::value;
};

template<unsigned p>
struct DoIsPrime<p, 2>
{
	static const bool value = (p % 2 != 0);
};

template<unsigned p>
struct IsPrime
{
	static const bool value = DoIsPrime<p, p / 2>::value;
};
int main()
{
	//目标：判断一个数是不是质数
	auto value = IsPrime<5>::value;
	std::cout << value << std::endl;//返回1，是质数

	auto value2 = IsPrime<6>::value;
	std::cout << value2 << std::endl;//返回0，是偶数
}
```

#### 9. SFINAE（发音类似 sfee-nay）
SFINAE,用一个求长度的例子len来实现,也是书上的例子

```c++
#include <iostream>
#include <vector>

template<typename T,unsigned N>
std::size_t len(T(&)[N])
{
	return N;
};

template<typename T>
typename T::size_type len(T const& t) {//size_type是容器概念,没有容器不能使用
	return t.size();
};

std::size_t  len(...)
{
	return 0;
}
int main()
{
	//SFINAE	
	std::cout << len("123") << std::endl;//只有那个为裸数组定义的函数模板能够匹配
	std::cout << len(std::vector<int>(4) ) <<std::endl;//传递 std::vector<>作为参数,只有第二个模板能匹配
	int* p = nullptr;
	std::cout << len(p); //以上两个模板都不会被匹配上,只能用上第三个泛用模板
}

```

#### 10. 翻转数列

```c++
//c++ 17
#include<iostream>
template<unsigned... N> struct Cont {};

template<typename T> struct Reverse;

template<typename T, typename P, size_t S> struct do_reverse;

template <unsigned... L>
struct Reverse<Cont<L...>> {
	using type = typename do_reverse<Cont<>, Cont<L...>, sizeof...(L)>::value;
};

template <unsigned N, unsigned... L1, unsigned... L2>
struct do_reverse<Cont<L2...>, Cont<N, L1...>, 1> {
	using value = Cont<N, L2...>;
};

template <unsigned N, size_t S, unsigned... L1, unsigned... L2>
struct do_reverse<Cont<L2...>, Cont<N, L1...>, S> {
	using value = typename do_reverse<Cont<N, L2...>, Cont<L1...>, (S - 1)>::value;
};
template<unsigned ...x>
void print_list(Cont<x...>) {
	((std::cout << x << ","), ...);
}
int main()
{
	//目标：翻转数列
	print_list(Reverse<Cont<1, 2, 3, 4, 5>>::type{});
}
```

#### 11. 数列相加

```c++
#include <iostream>
template<unsigned int... T>
struct Cont {};

template<typename T, typename P> struct Add;

template<typename T, typename P, typename Q, unsigned S> struct do_add;

template<unsigned ...L1, unsigned... L2>
struct Add<Cont<L1...>, Cont<L2...>> {
	using type = typename do_add<Cont<>, Cont<L1...>, Cont<L2...>, 0>::value;
};

template<unsigned... L3, unsigned... L1, unsigned... L2, unsigned N1, unsigned N2, unsigned S>
struct do_add<Cont<L3...>, Cont<N1, L1...>, Cont<N2, L2...>, S> {
	using value = typename do_add < Cont< L3..., ((N1 + N2 + S) % 10) >, Cont<L1...>, Cont<L2...>, ((N1 + N2 + S) / 10) > ::value;
};

template<unsigned... L3, unsigned... L1, unsigned N1, unsigned S>
struct do_add<Cont<L3...>, Cont<N1, L1...>, Cont<>, S> {

	using value = typename do_add < Cont<L3..., ((N1 + S) % 10) >, Cont<L1...>, Cont<>, ((N1 + S) / 10) > ::value;

};

template<unsigned... L3, unsigned... L2, unsigned N2, unsigned S>
struct do_add<Cont<L3...>, Cont<>, Cont<N2, L2...>, S> {
	using value = typename do_add < Cont < L3..., ((N2 + S) % 10) >, Cont<>, Cont<L2...>, ((N2 + S) / 10) > ::value;
};

template<unsigned... L3>
struct do_add<Cont<L3...>, Cont<>, Cont<>, 1> {
	using value = Cont<L3..., 1>;
};

template<unsigned... L3>
struct do_add<Cont<L3...>, Cont<>, Cont<>, 0> {
	using value = Cont<L3...>;
};

//目标：打印类似Cont<1,2,3>的数组
void print_list(Cont<>) {};

template<unsigned int N, unsigned int... T>
void print_list(Cont<N, T...>) {
	std::cout << N << std::endl;
	print_list(Cont<T...>{});
}

int main()
{
	//目标：数列相加
	print_list(Add<Cont<1, 2, 3, 4, 5>, Cont<1, 2, 8, 9, 7, 3>>::type{});//print 2 4 1 4 3 4
}

```




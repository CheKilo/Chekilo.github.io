---
layout: post
title: 浅谈一下move和forward
header-img: img/in-post/head/10.jpg
header-style: text
catalog: true
tags:
  - C++
  - std标准库
  - 移动语义
  - 完美转发
  - 学习
---
本文真的是浅浅谈一下std::move和std::forward。

没有任何理论正确性保证，仅作为笔者学习过程中的思考。
{:.info}

有关移动语义和完美转发的概念我就不赘述了，本文着重来聊一聊move和forward容易让人误解的地方。


### std::move

首先就是 **std::move** 这个函数，单从名字上来看，它的作用仿佛是将某些的东西 move(移动搬走) //saki 移动！

在刚开始接触这个函数时，我以为 std::move 是将目标对象转化为右值，而大多数讲解也确实是这么说的。

但是我们从它的源码就能看出来：
```cpp
  template<typename _Tp>
    constexpr typename std::remove_reference<_Tp>::type&&
    move(_Tp&& __t) noexcept
    { return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); }
```

这返回的分明是右值引用，而我们都知道，右值引用本身是左值。

所以 std::move 本质上来说，就是对目标对象做了一个右值引用的强制类型转换(简单又暴力啊)。

如此一来，在赋值或者拷贝的时候，就可以通过右值引用类型来触发目标类的移动拷贝/移动赋值构造函数。

而对于没有实现移动语义的类，即便你使用 std::move 也依旧只能触发普通的拷贝/赋值构造函数，仍然无法避免资源的再拷贝。

下面我们来聊聊完美转发，这也是当初困惑了我许久的一个问题。

在聊完美转发之前，先简单补充一下有关万能引用和引用折叠的概念。

#### 万能引用和引用折叠

众所周知，引用分为左值引用和右值引用，而所谓的万能引用便是利用模版参数实现的引用 **T&&** 。

虽然看起来 **T&&** 长得很像右值引用，但实际上它可以接受任意引用。

而它的实现原理，就是利用**引用折叠**。

引用折叠是 C++ 中的一种规则，当出现嵌套引用时，编译器会根据一定的规则将其折叠为单一的引用类型。具体的折叠规则如下：

T& & 折叠为 T&

T& && 折叠为 T&

T&& & 折叠为 T&

T&& && 折叠为 T&&

简单来说就是除了&& &&会被编译器折叠成&&外，其他不论是什么搭配都会被折叠成&。

举个例子：
```cpp
print(T &&){};
...

int x = 10;
int &&y = 20;
print(x);
print(y);
...
```
这里的x类型是int&，代入参数T后会变成int& &&,根据规则，编译器会将其折叠成int&。

而同理，y的类型是int&&，代入参数T后会变成int&& &&，最终被折叠成int&&。

由此可以看出，T&&既可以接受左值引用，也可以接受右值引用。

### std::forward

言归正传， std::forward 的作用是将参数类型完美地转发给其他函数。

举个例子：
```cpp
#include <iostream>

void printType(int& value) {
    std::cout << "Lvalue reference" << std::endl;
}

void printType(int&& value) {
    std::cout << "Rvalue reference" << std::endl;
}

template<typename T>
void test(T&& arg) {
    printType(arg);
}

int main() {
    int x = 10;
    test(x);  // 传入左值
    test(20); // 传入右值
    return 0;
}
```

如果是初学cpp的选手，恐怕认为上面的 test(x) 和 test(20) 会分别输出Lvalue和Rvalue吧？

但遗憾的是，这两次函数调用最终都会匹配成printType的左值版本。

我们先来看看函数调用的流程：

首先 test(20) 在模版实例化后会变成 test(int&& arg) ，紧接着20会被赋值给右值引用arg，最后test会再调用printType(arg)。

有的读者会说，这没毛病啊？

arg的类型是int&&，不正好匹配上右值版本的函数了吗？

而事实是右值引用本身是引用，也就是说它是个左值。

在进行匹配时，身为左值的arg是无法被赋值给右值引用int&&的，于是它便匹配上了printType的左值版本。

而一旦我们将对应的左值版本删去，编译器就会报错：

**error: cannot bind rvalue reference of type 'int&&' to lvalue of type 'int'**

含义也很清楚：不能将右值引用绑定到左值上。

 std::forward 就是为了解决这种形参赋值后导致右值变成左值的问题。

我们先来看看它的源码：

 ```cpp
  template<typename _Tp>
    _GLIBCXX_NODISCARD
    constexpr _Tp&&
    forward(typename std::remove_reference<_Tp>::type& __t) noexcept
    { return static_cast<_Tp&&>(__t); }

  template<typename _Tp>
    _GLIBCXX_NODISCARD
    constexpr _Tp&&
    forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
    {
      static_assert(!std::is_lvalue_reference<_Tp>::value,
	  "std::forward must not be used to convert an rvalue to an lvalue");
      return static_cast<_Tp&&>(__t);
    }
 ```

其实很简单，就是将左值引用强转为左值引用，右值引用强转为右值引用。

而这里对于右值引用的处理，实际上和 std::move 很类似。

阅读到这里，可能读者们还会有疑惑，为什么将右值引用强转为右值引用就能解决右值变左值的问题呢？

把苹果强转为苹果，这不是多此一举吗？

实际上这里强转后返回的并不是普通的右值引用对象，而是匿名的右值引用对象。

匿名右值引用对象（即临时的右值引用）在函数匹配时会被当作右值处理，这就能够解决上面描述的问题了。

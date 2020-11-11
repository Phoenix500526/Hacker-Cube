---
title: 对 muduo 网络库单例模式的思考与实践(下)
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-09 14:17:27
password:
summary:
tags: [muduo网络库, SFINAE, 泛型编程]
categories: muduo源码剖析
---

#### 文前导读
本文是 muduo 网络库源码剖析系列文章的第八篇文章，是对前一篇文章的一个补充。另外需要注意，本文假定读者已经了解了 SFINAE 的基本含义，因此没有花费笔墨具体讨论什么是 SFINAE，如果读者不知道什么是 SFINAE ，建议先阅读知乎大佬：空明流转的文章 [《C++模板进阶指南：SFINAE》](https://zhuanlan.zhihu.com/p/21314708)，以便对 SFINAE 有基本的了解。
本文主要包含以下内容
> * SFINAE —— muduo Singleton 中 has_no_destroy 的启示
> * `has_no_destroy` 的进阶玩法 —— 如何判断一个类的基类子类型中是否声明了某个函数
>   * `has_no_destroy` 为何无法判断子类中基类方法
>   * 从一个简单的例子讲起 —— 从编译错误中得到的启示
>   * `has_no_destroy` 的进阶实现与测试

本文中所涉及的代码来源于我的个人项目：tmuduo。本着“纸上学来终觉浅，绝知此事要躬行”的想法，我将 muduo 网络库重新实现了一遍，并在上面验证了自己的不少猜想。项目的仓库地址为 git@github.com:Phoenix500526/Tmuduo.git, 欢迎各位fork + star，一起加入学习
<!-- more -->

#### SFINAE —— muduo Singleton 中 has_no_destroy 的启示
再讲完了 tmuduo 中单例模式的实现后，我们来看看 muduo 中 Singleton 使用到的另一个技术 —— SFINAE(Substitution failure is not an error)。我们先来看看 Singleton 中 `has_no_destroy` 的实现：
```C++
template<typename T>
struct has_no_destroy{
  template <typename C> static char test(decltype(&C::no_destroy));
  template <typename C> static int32_t test(...);
  const static bool value = sizeof(test<T>(0)) == 1;
};
```
如前言所述，`has_no_destroy` 的作用是用来判断一个类中是否声明了 `no_destroy` 函数。我们先来看看它的具体用法：
```C++
#include <iostream>
using namespace std;

class A{
public:
    void no_destroy();
};

class B{
public:
    int no_destroy(int);
};

class C : A{};

class D{};

int main(){
    cout << "Whether A has no_destroy or not? " << (has_no_destroy<A>::value ? "yes" : "no") << endl;  
    cout << "Whether B has no_destroy or not? " << (has_no_destroy<B>::value ? "yes" : "no") << endl;   
    cout << "Whether C has no_destroy or not? " << (has_no_destroy<C>::value ? "yes" : "no") << endl;  
    cout << "Whether D has no_destroy or not? " << (has_no_destroy<D>::value ? "yes" : "no") << endl;  
}
```
执行结果：
> Whether A has no_destroy or not? yes
> Whether B has no_destroy or not? yes
> Whether C has no_destroy or not? no
> Whether D has no_destroy or not? no

在讲述 `has_no_destroy` 的原理前，我们先看下面的代码：
```C++
#include <iostream>
using namespace std;

int32_t foo1(int);
char foo2(void*);

int main(){
    cout << "sizeof foo1(0) is " << sizeof(foo1(0)) << endl;   
    cout << "sizeof foo2(0) is " << sizeof(foo2(0)) << endl;   
}
```
执行结果:
> sizeof foo1(0) is 4
> sizeof foo2(0) is 1

在上述代码中，我们声明了两个函数: `foo1` 和 `foo2`。注意这两个函数都是只声明未定义。当我们执行 `sizeof(foo1(0))` 时，实际上是在检测 `foo1` 返回值类型的大小。这里简单提一下，编译器会在编译时使用 `sizeof(int32_t)` 代替 `sizeof(foo1(0))` 进行求值。虽然 `sizeof(foo1(0))` 看似产生了函数调用，实则不然。由于编译器没有编译出函数调用的命令，所以链接器并不会去查找 `foo1` 的定义，也就不会产生任何错误。

让我们重新看看 `has_no_destroy` 的实现：
```C++
template<typename T>
struct has_no_destroy{
  template <typename C> static char test(decltype(&C::no_destroy));
  template <typename C> static int32_t test(...);
  const static bool value = sizeof(test<T>(nullptr)) == 1;
};
```
在上述代码中，两个 `test` 函数都只是只声明未定义。当我们查看 `has_no_destroy<A>::value` 的值时，会先执行 `has_no_destroy<A>` 的实例化。由于 A 中声明了 `no_destroy` 函数，根据模板实例化的原则，函数 ``static char test(decltype(&C::no_destroy))``是最佳匹配，因此会实例化该函数，此时 `test<T>(nullptr)` 的返回值便是 char 类型，大小为 1 ，故 value 的值为 true。当执行 `has_no_destroy<D>::value`，同样会执行 `has_no_destroy<D>` 的实例化。由于 D 中没有声明 `no_destroy` 函数，因此只能实例化 `static int32_t test(...)` 函数，其返回值为 `int32_t`, 故 value 的值为 false(...代表接受任意数量任意类型的参数)。这样我们就实现了一种能够判断类中是否具有成员函数 `no_destroy` 的方法。

#### `has_no_destroy` 的进阶玩法 —— 如何判断一个类的基类子类型中是否声明了某个函数
###### `has_no_destroy` 为何无法判断子类中基类方法
从前面的实例当中我们可以看出，`has_no_destroy` 不能用于判断具有子类中是否具有基类的 `no_destroy` 方法，例如前面例子中的 C 类型。有没有什么办法能够增强 `has_no_destroy` 的功能，使其既能保持当前的用法不变，同时又能自动判断子类中的基类子类型是否包含 `no_destroy` 函数。

我们先看看为什么 `has_no_destroy` 无法干这个活。`has_no_destroy` 的基本工作原理，是利用了模板的匹配原则，如果 C 中恰好声明了函数 `no_destroy`,那么函数 `static char test(decltype(&C::no_destroy))` 将会得到实例化。但是当子类型中包含了基类函数时，以上述的 C 为例子，C 中虽然包含了从基类 A 中继承来的成员函数 `no_destroy`，但其函数签名应当是 `A::no_destroy` 而非　`no_destroy`。当我们对 C 实例化对象调用 `no_destroy` 函数时，编译器在 C 自身类型中找不到对应 `no_destroy` 函数，便会去基类子类型中查找 `no_destroy` 函数。**为了便于理解，你可以将这个过程看成是一次隐式类型转换：编译器将 `no_destroy` 转换成了 `A::no_destroy`。不过由于模板自身有类型推导规则，因此编译器不会为 `has_no_destroy` 执行这个转换工作。**我们需要自己来。
显然现在摆在我们面前的只有两条路，要么多增加一个模板类型参数，让用户把基类类型传进来，然后在 `has_no_destroy` 中做手动转换；要么反过来，让所有没有定义 `no_destroy` 的函数匹配成功，然后对 value 取反就可以。第一条路非常不好走，一方面是因为我之前说明了，我希望做一个 `has_no_destroy` 的增强版，因此我不希望增加多余的类型参数，另一方面，这种做法对于没有继承体系的类非常不友好，甚至来说是错误的存在。因此我选择了第二种做法，不仅能保持 `has_no_destroy` 的基本用法不变，而且对带继承或不带继承的类型等同视之，能够简化判断的策略。

###### 从一个简单例子讲起 —— 从编译错误中得到的启示
既然明确了要采用反证的思想来解决这个问题，我们希望所有定义了 `no_destroy` 函数的类型均无法正确匹配。在这之前先用一个例子来说明一下我的思路：
```C++
#include <iostream>
using namespace std;
class A{
public:
    void no_destroy(){
        cout << "A::no_destroy()" << endl;
    }
};

class B{
public:
    void no_destroy(){
        cout << "B::no_destroy()" << endl;
    }
};

class C: public A{};

class D: public A, public B{};

class F: public C, public B{};

int main(){
    C c;
    c.no_destroy();
    D d;
    //d.no_destroy(): // error: member 'no_destroy' found in multiple base classes of different types
    F f;
    //f.no_destroy(): // error: member 'no_destroy' found in multiple base classes of different types
    return 0;
}
```
在上述代码中，D 由于同时继承 A 和 B，因此当我们执行 `d.no_destroy()`函数时，会产生二义性错误(f 亦同理)。反观 C，由于 C 只继承了 A，因此当执行 `c.no_destroy()`， 编译器在 C 的类型中找不到 `no_destroy` 的定义时，会自动到 C 的基类子类型中寻找。我们可以模仿这种做法，**先在 `has_no_destroy` 中定义一个声明了 `no_destroy` 函数 `Base` 作为基类，然后让类型 T 也作为基类，并让 Derive 作为二者的联合派生类。接着在对 Derive 中的 `no_destroy` 进行模板类型推导。如果 T 中或 T 的基类子类型中包含了 `no_destroy` 类型，那么都会产生二义性错误从而导致相应的函数模板实例化失效。**

###### 进阶 `has_no_destroy` 的实现与测试
讲完了上述想法后，我们可以开始着手 `has_no_destroy` 进阶版的实现：
```C++
//SFINAE.h
template <typename T> 
class has_no_destroy
{ 
   using yes = char;
   using no = int32_t;
   struct Base 
   { 
     void no_destroy();
   }; 
   struct Derive : public T, public Base {}; 
   template <typename C, C>  class Helper{}; 
   template <typename U> 
   static no test(U*, Helper<decltype(&Base::no_destroy), &U::no_destroy>* = nullptr); 
   static yes test(...); 
public: 
   static const bool value = sizeof(yes) == sizeof(test(static_cast<Derive*>(nullptr))); 
}; 
```
在上述代码中，由于采用了反证的思想，逻辑比较绕，我采用了更有意义的类型别名 `yes` 和 `no` 来描述结果。其中 `Helper` 是用来辅助模板类型推断的辅助类模板。 不管是 U 还是 U 的基类子类型中，只要声明了 `no_destroy`，都会因产生二义性错误而导致 `Helper` 实例化失败，进而导致 `static no test` 实例化失败。 此时 `value` 中对 `test` 的调用将会匹配到 `static yes test`。如果 U 或者 U 的基类子类型中没有声明 `no_destroy` 函数，则 `static no test` 将会实例化成功，根据最佳匹配原则，`value` 中对 `test` 的调用将会匹配到 `static no test`,这样我们便实现了区分
以下是我对进阶版 `has_no_destroy` 的测试。
测试代码：
```C++
#include <iostream>
#include "SFINAE.h"
using namespace std;
class A {
    void no_destroy();
};

class B : A {};

class C {};

int main()
{
    cout << "Whether A has no_destroy or not? " << (has_no_destroy<A>::value ? "yes" : "no") << endl;  
    cout << "Whether B has no_destroy or not? " << (has_no_destroy<B>::value ? "yes" : "no") << endl;   
    cout << "Whether C has no_destroy or not? " << (has_no_destroy<C>::value ? "yes" : "no") << endl;  
}
```
测试结果：
> Whether A has no_destroy or not? yes
> Whether B has no_destroy or not? yes
> Whether C has no_destroy or not? no

关于进阶版的 `has_no_destroy` 的实现代码我放在了 tmuduo/base/Singleton.h 当中，不过由于这个功能仅是兴趣，没有涉及到 tmuduo 本身的使用，我没有将测试代码添加到 tmuduo/test 当中，测试本身不复杂，有需要的可以自己实现。
---
title: 关于 C++ 的 lvalue、xvalue 和 prvalue
top: false
cover: false
toc: true
mathjax: true
date: 2020-12-17 20:23:17
password:
summary:
tags: [C++, lvalue, xvalue, prvalue]
categories: C++沉思录
---

#### 文前导读

本文主要涉及了对 C++ 语言中 lvalue、xvalue 和 rvalue 的一些探讨，主要包括以下内容

> * 一段有趣的代码
> * C++ 中的 Expression Category Taxonomy
> * prvalue 的作用以及判定标准
> * xvalue 的定义及判定标准
> * lvalue 的定义及判断标准
> * 关于 glvalue 与 rvalue
> * Reference

本文按照先总后分，自底向上的逻辑讨论，先论述 C++ 中的 Expression Category Taxonomy，再到底层的 prvalue、xvalue 以及 lvalue，最后回到 glvalue 及 rvalue 上。

<!-- more -->

#### 起因：一段有趣的代码<sup>[0]</sup>

在前一段时间，我写出了如下的一段代码：

```C++
// value.cc
#include <iostream>
using namespace std;

class A{
public:
    void sendByRvalue(string&& str){
        cout << str << endl;
    }
};

class B{
private:
    A a;
    void send(string&& str){
        a.sendByRvalue(str);
    }
public:
    void run(const string& str){
        send("run " + str + "\n");
    } 
}; 

int main(void){
    string str("hello world");
    B b;
    b.run(str);
    return 0;
}
```

这段代码看似平平无奇，但当我对其进行编译时，我却得到了如下的错误信息：

```bash
$ clang++ value.cc -std=c++11
$ error: rvalue reference to type 'basic_string<[...]>' cannot bind to lvalue of type 'basic_string<[...]>'
        a.sendByRvalue(str);
```

从错误的信息来看，出错的原因应该是我试图将一个左值 str 传递给一个参数类型为 rvalue reference 的 `sendByRvalue` 函数。等等，好像哪里不对？str 本来就是一个 rvalue reference 啊。起初我怀疑是 str 在传递给 `send` 函数时发生了突变，导致在执行`send`函数体时变成了 lvalue，于是我对代码做了一点修改：

```C++
class B{
private:
    A a;
    void send(string&& str){
        cout << boolalpha << is_rvalue_reference<decltype(str)>::value << endl;
        a.sendByRvalue(std::move(str));
    }
public:
    void run(const string& str){
        send("run " + str + "\n");
    } 
}; 
```

现在，代码能够正常编译并运行，但是代码的运行结果却使我大跌眼镜：

```bash
$ true
$ run hello world
```

显然，str 在传递给 `a.sendByRvalue` 前还是 rvalue reference。然而，结合之前编译器的错误信息，我们竟然能够得出这样一个结论：一个右值引用变量居然不能传递给一个参数类型为右值引用的函数！？ 这是咋回事？？

#### C++ 中的 Expression Category Taxonomy

在 C++ 中，任何表达式都具备两个独立的属性：类型(type)和值类别(value category)。关于表达式的 type，指的是常量表达式，整型表达式等概念，而 **value category 虽然名字上带有 `value`，但实际上描述的是表达式而非值**，主要包括 glvalue、rvalue、lvalue、xvalue 以及 prvalue，它们之间的关系如下图：

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/7d1d6e2a-9dbe-4341-b09f-2bfe4c848544.png"/>
</div>
<div class="image-caption" align="center">Expression category taxonomy</div>

cpp reference 上为这种分类方法提供了解释依据：

> With the introduction of move semantics in C++11, value categories were redefined to characterize two independent properties of expressions:
> * has identity: it's possible to determine whether the expression refers to the same entity as another expression, such as by comparing addresses of the objects or the functions they identify (obtained directly or indirectly);
> * can be moved from: move constructor, move assignment operator, or another function overload that implements move semantics can bind to the expression. <sup>[1]</sup>

从上述解释中，我们可以知道 C++ 11 中对于表达式值分成 glvalue 和 rvalue 的主要原因。早期的 C++ 标准不支持移动语义，因此对于表达式的类别的判断依据仅为是否拥有 "identity", 这也就将表达式的类别一分为二，分为了 lvalue 和 rvalue。**在 C++ 11 标准中引入了移动语义后，标准将 `has identity` 的表达式定为 glvalue， 而将`can be moved from`的表达式定为 rvalue**，并根据这两种属性两两组合，得到了以下的 4 种可能：

> * have identity and cannot be moved from are called *lvalue* expressions;
> * have identity and can be moved from are called *xvalue* expressions;
> * do not have identity and can be moved from are called *prvalue* ("pure rvalue") expressions;
> * do not have identity and cannot be moved from are not used.

#### prvalue 的作用以及判定标准

我们先来看看 prvalue，所谓的 prvalue 是 `Pure rvalue` 的缩写。prvalue 主要包括以下两种：

* 用于计算与对象无关的值(computes a value that is not associated with an object)
* 创建出来的临时对象

这里简单地解释一下什么叫做`与对象无关的值(a value that is not associated with an object)`。在 C++ draft 当中是这样解释 object 的：
> The constructs in a C++ program create, destroy, refer to, access, and manipulate objects. An object is created by a definition, by a new-expression , by an operation that implicitly creates objects (see below), when implicitly changing the active member of a union, or when a temporary object is created . An object occupies a region of storage in its period of construction , throughout its lifetime, and in its period of destruction.  <sup>[2]</sup>

在 C++ 中，所谓的对象是指一个存储区域，这块存储区域需要通过对定义式或 new 表达式显式或隐式地调用来创建。而与对象无关的值，实际上指的就是那些没有相应存储区的值，例如除了字符串字面值以外的所有字面值。

在 C++ 中，属于 prvalue  的表达式有以下几种：
> * 除了字符串字面值以外的所有字面值，如 `12`, `true` 或者 `nullptr`
> * 返回值类型为 non-reference 类型的函数调用表达式，包括了一般形式的函数调用，如 `f()`，也包括了重载过的函数运算符，如 "str1 + str2"
> * 返回临时值的 built-in 运算符表达式，例如 `a++`, `a + b`, `&a`, `a < b` 等
> * 将对象转换为 non-reference 类型的 cast 表达式，如 `static_cast<double>(x)`
> * **任何枚举值都是 prvalue**，也包括定义在类中的枚举值，如 `obj.m` 或 `obj_ptr->m`，其中 m 是枚举值
> * this 指针
> * lambda 表达式

为了避免和前面非对象类的 prvalue 判断相混淆，我特地将类对象的 prvalue 的部分单独拎出来，放在了下面：
> 针对成员变量：
> * 若对象 obj 是 rvalue，则形如 `obj.m` 与 `obj.*mp` 为 prvalue，其中 m 为 non-static 的 non-reference 类型的成员变量， mp 为指向数据成员的指针
>
> 针对成员函数(注意，此处函数指的是函数本身`obj.func`，而不是函数调用`obj.func()`)：
> * 当 func 为 obj 的 non-static 成员函数时， `obj.func` 和 `obj_ptr->func` 也为 prvalue。
> * 当 func_ptr 为指向对象成员函数的函数指针时，则`obj.*func_ptr` 和 `obj->*func_ptr` 均为 prvalue

#### xvalue 的定义及判定标准

接下来看看 xvalue，所谓 xvalue 是 `eXpiring value` 的缩写，代表了资源在回收前可以被重复利用的 glvalue。在 C++ 当中，类型为 xvalue 的表达式有以下四种：
>  * 对类型为 rvalue 类型的数组进行 built-in 的下标运算，如 arr[n]
>  * 对于形如 obj.m 或 obj.\*mp 的表达式，如果 obj 是 xvalue，或者 m 为 non-static、non-reference 的数据成员，或者 mp 为指向数据成员的指针，则 obj.m 与 obj.\*mp 为 xvalue
>  * 当 obj 为 rvalue 时，且 m 为 non-static non-reference 类型的数据成员，mp 为指向数据成员的指针，则 `obj.m` 或 `obj.*mp` 为 xvalue 
>  * 返回类型为 rvalue reference 的函数调用表达式，包括了一般的函数调用以及重载运算符，如 std::move(x)
>  * 将一个对象强制转换为 rvalue reference 的表达式，如 static_cast<int&&>(obj)

#### lvalue 的定义及判断标准

最后来看看 lvalue，所谓的 lvalue 最初是指那些可以被放在 built-in 赋值运算符左侧的表达式，然而随着 C++ 语言的演化，这一说法也变得不再准确。在新版本中，lvalue 被定义为那些属于 glvalue 但是不属于 xvalue 的表达式类型。lvalue expression 都符合以下条件：

> * **任何有名字的变量，函数，数据成员，无论其类型是什么，一律都是 lvalue**。即便是 rvalue reference 类型的具名变量，其 value category 也是 lvalue
> * 任何返回类型为 lvalue reference 的函数调用表达式，其中函数调用也包括了以重载运算符形式进行的调用
> * 任何返回类型为**对函数的 rvalue reference** 的函数调用表达式，其中函数调用也包括了以重载运算符形式进行的调用
> * 任何返回类型为 lvalue reference 的 built-in 运算符表达式，例如 `++a`，`a += b` 等。
> * 对 lvalue 类型数组 arr 执行 built-in 的取下标运算符, 如 `arr[n]`
> * 对指针变量 p 的解引用操作，如 `*p` 或者 `p[n]`
> * 字符串字面值，如 `HelloWorld`
> * 将对象强制转换为 lvalue reference 的 cast 表达式，如 `static_cast<int&>(x)`
> * 将对象强制转换为**对函数的 rvalue reference** 的 cast 表达式，如 `static_cast<void (&&)(int)>(x)`

和前面相同，这里单独将与对象相关的 lvalue 拎出来讨论：

> 针对成员变量：
>
> * 对于 `obj.m` 的表达式，如果 obj 是 glvalue 或者 m 是静态数据成员，则 `obj.m` 是 lvalue
> * 对于 `obj.*mp` 的表达式，如果 obj 是 lvalue 且 mp 是指向数据成员的指针，则 obj.*mp 是 lvalue
> * 对于 `obj->*mp` 的表达式是 lvalue，其中 mp 是指向其数据成员的指针
>
> 针对成员函数(注意，此处函数指的是函数本身`obj.func`，而不是函数调用`obj.func()`)：
>
> * 若 func 为静态成员函数，则 `obj.func` 和 `obj->*func`是 lvalue

这里需要提一下，**只有字符串字面值是 lvalue，而其他的字面值都是 rvalue**。因为在 C++ 中，任何字符串字面值，例如 "Hello world"，**其类型都是数组类型，而不是指针类型**。这主要是因为在 C 语言中，数组类型必须作为 lvalue，否则无法兼容 C 语言中的一些用法，比如使用 `sizeof(arr)` 来求出 arr 所占内存空间的大小。C++ 为了兼容 C 语言，沿袭了这一习惯<sup>[3]</sup>。

#### 关于 glvalue 与  rvalue 

先来看看 glvalue。在 C++ 11 中，glvalue 代表了 `Generalized lvalue`, 一个 glvalue 要么是 lvalue，要么是 xvalue，具备以下特征：

> * 一个 glvalue 通过隐式转换，如 lvalue-to-rvalue、array-to-pointer 或者 function-to-pointer，转换成为 prvalue
> * 一个 glvalue 可以是多态的，其对象的动态类型并不一定需要与表达式的静态类型相一致
> * 在表达式允许的范围内，glvalue 可以拥有不完全类型

lvalue 可以视为是对 glvalue 特征的一个补充，它除了具备 glvalue 所有的特征以外，还额外具备以下特征：

> * 可以使用 built-in 的取地址运算符 & 为 lvalue expression 进行取地址运算，例如 `&++i` 是合法的
> * 一个可变的 lvalue expression 可作为 built-in 的赋值运算符和复合赋值运算符的左操作数
> * 一个 lvalue expression 可以用于被绑定到一个 lvalue reference 上

再来看看 rvalue。在 C++ 11 中， 一个 rvalue 要么是 prvalue，要么是 xvalue，具备了以下的特征：

> * 无法使用 built-in 的 & 运算符对其取地址，例如 `&int()`, `&i++` 等都是非法的
> * rvalue 无法被放在 built-in 的赋值运算符的左侧
> * rvalue 可以绑定到 const lvalue reference 或 rvalue reference 类型的变量上，一旦完成绑定，rvalue 的生命周期将会延长到引用结束

对于 prvalue 而言，除了具备 rvalue 的所有特征外，还额外具备以下特征：

> * prvalue 不能是除了 `void` 以外的不完全类型
> * non-class、non-array 的 prvalue expression 不能被 cv 限定符修饰
> * 一个 prvalue expression 不具备多态性，其动态类型始终和它的静态类型相一致、

而 xvalue 作为 lvalue 到 prvalue 之间的过渡，它同时具备 rvalue 以及 glvalue 的特征。一个 xvalue 既可以像 rvalue 那样被绑定到 rvalue reference 上，又可以像 glvalue 那样体现出多态的特性。

最后，回到之前的那段代码, 尽管 `void B::send(string&& str)` 中的参数 `str` 是一个 rvalue reference，但它依然是一个 lvalue，无法用来初始化一个 rvalue reference，自然也就无法传递给函数 `void A::sendByRvalue(string&& str)`。

#### Reference

[0]. [Why the rvalue reference parameter cannot be passed between functions directly?](https://stackoverflow.com/questions/64798137/why-the-rvalue-reference-parameter-cannot-be-passed-between-functions-directly)
[1]. [Cppreference: Value categories](https://en.cppreference.com/w/cpp/language/value_category)
[2]. [P0593R5. Implicit creation of objects for low-level object manipulation](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0593r5.html#662-object-model-introobject)
[3]. [Is a literal, say “Hello world”，a rvalue when I passing it to a template which its paramater is an universal reference? - Stackoverflow](https://stackoverflow.com/questions/63899666/is-a-literal-say-hello-world-a-rvalue-when-i-passing-it-to-a-template-which-i)
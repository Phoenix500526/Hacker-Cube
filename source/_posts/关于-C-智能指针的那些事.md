---
title: 关于 C++ 智能指针的那些事
top: false
cover: false
toc: true
mathjax: true
date: 2020-12-20 09:29:47
password:
summary:
tags: [C++, smart pointer, shared_ptr, weak_ptr, unique_ptr, PImpl Idiom, CRTP Idiom]
categories: C++沉思录
---

#### 文前导读

本文并不是一篇关于Smart Pointer的基础文章，它假定阅读读者至少使用过 C++11 下的 Smart Pointer，因此文章中并不涉及 Smart Pointer 的基础用法，而是希望讨论一些更深层次的东西，主要包含了以下内容

> * 引用型 Smart Pointer 的内存布局以及相关推论
> * make_shared 的优劣
> * enable_shared_from_this 与 shared_from_this 的实现
> * weak_ptr 的内存布局及相关应用
> * unique_ptr 与 PImpl Idiom 的相关问题
> * Reference

<!-- more -->

#### 引用型 Smart Pointer 的内存布局

在 C++11 中，`shared_ptr`的大小为裸指针的两倍，它包含两个数据成员：一个指向共享资源的指针和一个指向控制块的指针。控制块中存放了引用计数、用户自定义删除器的副本以及分配器的副本(后两个要求用户显式指出)，而结构如下：

<div style="text-align:center">
    <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/13b6cad0-d3c6-44a9-b077-2a10bc137c14.png">
</div>
<div class="image-caption" align="center">shared_ptr 的内存布局</div>

从 shared_ptr 的内存布局可以看出，我们可以得出以下三个结论和两个问题：

**结论一：用户定义的 deleter 和 allocator 与 shared_ptr 的类型无关**

用户自定义的 deleter 和 allocator 并不属于 shared_ptr 类型的一部分。换言之，不论用户定义的 deleter 是函数指针、functor 抑或是 lambda，都不影响 shared_ptr 的最终大小。其次，对于指向同一类型对象，但是定义了不同 deleter 的 `shared_ptr`而言，它们之间没有类型差异，可以共存于同一个容器当中，例如下面的代码是可以通过编译的：

```C++
shared_ptr<Object> p1(new Object, customDeleter1);
shared_ptr<Object> p2(new Object, customDeleter2);
shared_ptr<Object> p3(new Object);
std::vector<shared_ptr<Object>> vec{p1, p2, p3};
```

**结论二：对 Object 的读写无法是原子的**

对于 shared_ptr 而言，对对象的读写操作分成了两个部分，一个是对 `Object` 的访问，另一个则是对 `reference count`的更新操作。**对于后者而言，在主流平台上的实现都是原子操作，而且没有用到锁机制，性能不俗**。但是由于读写 `Object` 和更新 `reference count` 是两个不同的操作，`shared_ptr` 没有提供额外的机制来保证这两个操作的原子性，因此不论`Object`的读写是不是线程安全的，`shared_ptr`本身也不是 100% 线程安全的。因此在多线程环境下读写同一个`shared_ptr`还是需要使用 `mutex` 进行保护的。

**结论三：从原则上，Object 与 Control Block 在空间上不必是相邻，这在内存分配时可能会带来潜在的效率及安全问题**

从内存布局上可以看出, `shared_ptr`中的`Object`和`Control Block`在空间上不必是相邻的，这会带来一些运行效率上的问题，例如`auto ptr = shared_ptr<Object>(new Object)`可能会涉及两次内存分配，带来一些额外的开销，也可能会带来安全性的问题，例如 `new Object` 执行成功了，但是因为某种原因导致`new`出来的指针没有及时存入到`shared_ptr`中，造成了内存的泄漏。关于这个问题的解决方案，我会在后面的 `make_shared`  部分来讨论。

**问题一：Control Block 中的 weak count 和 reference count 有什么区别？**

在 C++ 中，析构与释放内存并不是同义词。对于`new`和`delete`而言，前者的工作是先开辟一片内存空间，然后在该内存空间上执行构造函数，构造完毕后将句柄返回给调用者，而后者的工作则是先在指定的内存空间上执行析构函数，完成一些清理善后工作，之后将这片内存空间归还给操作系统。由此可以看出，析构和内存释放其实是两个过程。回到这个问题上，**`Control Block`中的`reference count`决定了`Object`会在何时被析构，而`weak count`决定了`Control Block`所占用的内存区域何时被释放**。

**问题二：什么样的行为会导致 Control Block 的创建？**

对于指向同一个`Object`的多个`shared_ptr`而言，维护它们的`Control Block`的唯一性至关重要。以下的三种构造方式会触发`Control Block`的创建：

> * 使用裸指针来构造`shared_ptr`
> * 使用`unique_ptr`来构造`shared_ptr`
> * 使用`make_shared`来构造`shared_ptr`

因此，使用同一个裸指针反复创建多个`shared_ptr`不仅不能使`shared_ptr`发挥它们应有的功能，还可能会带来安全问题：只要其中一个`shared_ptr`被析构，那么对剩下的任何一个`shared_ptr`执行解引用操作都会导致未定义行为

除了上述的三种行为外，使用`weak_ptr`来构造`shared_ptr`、或者通过拷贝构造、拷贝赋值、移动构造、移动赋值的方式构造`shared_ptr`也不会导致`Control Block`的创建。其中，**出于效率的考虑，使用移动赋值和移动构造也不会更新`reference count`信息**。


#### make_shared 的优劣

针对前面结论三中所提到的效率及安全问题，我们先上一小段代码来进一步说明问题：

```C++
void Process(std::shared_ptr<Object> spw, int priority){
    //... do something;
}
int computePriority(const string& key){
    int priority = 0;
    // ... calculate priority by key
    return priority;
}
int main(){
 	//...   
    Process(std::shared_ptr<Object>(new Object), computerPriority("Window"));
}
```

对于上述代码，当执行语句`Process(std::shared_ptr<Object>(new Object), computerPriority("Window"));`,需要先进行参数的计算，但参数计算顺序是不确定的。换句话讲，语句`Process(std::shared_ptr<Object>(new Object), computePriority("Window"));`的执行顺序可能是：

> 1. 执行 `new Object`，在堆内存上构造出 Object 对象
> 2. 执行 computePriority("Window") 函数
> 3. 将 `new Object`所得到的指针包装成 `shared_ptr`对象
> 4. 将参数传递给 `Process`执行

在上述过程中，如果`computePriority`抛出了异常，那么就会导致步骤 1 中 `new` 出来的指针无法及时存放到`shared_ptr`中，从而无法自动执行析构，造成了内存的泄漏。

对于上述问题，一个最简单的方式便是将`new Object`从`Process`当中分离出来，如：

```C++
// 此处省略了 Process 与 computePriority 的代码
int main(){
    //...
    int priority = computerPriority("Window");
    auto ptr = shared_ptr<Object>(new Object);
    Process(std::move(ptr), priority);	//使用移动构造以避免更新 ptr 的 reference count
}
```

这种方式解决了前面所提到的内存泄漏的问题，不过依然为`Object` 和 `Control Block` 分别分配了内存空间。一种更好的方式是使用 C++ 标准库提供的 `make_shared`来实现，如下：

```C++
int main(){
    //...
	Process(std::make_shared<Object>(), computePriority("Window"));
}
```

相较于`shared_ptr<Object>(new Object)`的方式，`make_shared<Object>()`会一次性分配好`Object`和`Control Block`的内存空间，并在该内存空间上构造`Object`的对象。这样既解决了前面可能因函数求参顺序不确定而导致的内存泄漏的问题，同时也提高了运行的效率，减少了一次内存的分配。除了安全和效率两个优点外，`make_shared`还有助于提高程序的可维护性，一方面使用`make_shared`来构造`shared_ptr`可以避免产生任何形式的裸指针，也就降低了用户因误用裸指针而导致的`shared_ptr`出现`Control Block`信息不一致的可能性，另一方面结合 `auto`关键字也可以使得`shared_ptr`与特定类型名称的耦合程度尽可能低，代码改起来工作量也会少一些。

当然，`make_shared`也不是银弹，也有其自身的局限性。首先，**使用`make_shared`构造`shared_ptr`无法指定 deleter 和 allocator**。其次，将`Object`和`Control Block`的内存空间合并虽然提高了效率，但可能会导致这片内存区域迟迟无法被释放。正如前面`weak count`与`reference count`的作用时所提到的，**当`Object` 和`Control Block`不属于同一片内存空间时，一旦`reference count`降为零，则会析构`Object`并回收其所占用的内存；当`weak count`降为零时，则会析构并回收`Control Block`所占用的内存。若`Object`和`Control Block`属于同一片内存空间时，则当`reference count`降为零时，会析构`Object`对象，但只有当`reference count`和`weak count`都为零时，才会回收这片内存空间**。因此，如果当对象占用内存较大，且指向该对象`weak_ptr`的生命周期要远长于指向该对象的`shared_ptr`时，使用`make_shared`就不那么合理了。

#### enable_shared_from_this 和 shared_from_this 的实现

使用`make_shared`可以替代`new`表达式来构造`shared_ptr`对象，进而降低指向同一对象的`shared_ptr`出现`Control Block`不一致的问题。但这种作法并非万无一失，因为最常被用来构造`shared_ptr`对象的指针除了`new`句柄，也包括了`this`指针，例如：

```C++
std::vector<shared_ptr<Object>> vec;
class Object{
public:
    void Process(){
        //...
        vec.emplace_back(this);
    }
};
```

而解决方法是先继承标准库中的模板基类`enable_shared_from_this<T>`，并使用`shared_from_this`代替`this`指针，具体如下：

```C++
class Object : public enable_shared_from_this<Object>{
public:
    void Process(){
        //...
        vec.emplace_back(shared_from_this());
    }
};
```

在上述代码中，`Object`继承了一个由`Object`所实例化的模板基类`enable_shared_from_this<Object>`。这种做法在 CRTP Idiom 中会经常使用到。CRTP 可以通过静态绑定的方法提升继承多态的性能。`enable_shared_from_this`的一个可能实现如下：

```C++
template<class D>
class enable_shared_from_this {
protected:
    constexpr enable_shared_from_this() { }
    enable_shared_from_this(enable_shared_from_this const&) { }
    enable_shared_from_this& operator=(enable_shared_from_this const&) {
        return *this;
    }
public:
    shared_ptr<T> shared_from_this() { return shared_ptr<T>(self_); }
    shared_ptr<T const> shared_from_this() const { return shared_ptr<T const>(self_); }

private:
    weak_ptr<D> self_;
    friend shared_ptr<D>;
};

template<typename T>
shared_ptr<T>::shared_ptr(T* ptr) {
    // ...
    // Code that creates control block goes here.
    // ...

    // NOTE: This if check is pseudo-code. Won't compile. There's a few
    // issues not being taken in to account that would make this example
    // rather noisy.
    if (is_base_of<enable_shared_from_this<T>, T>::value) {
        enable_shared_from_this<T>& base = *ptr;
        base.self_ = *this;
    }
}
```

在 `enable_shared_from_this`中，包含了一个类型为 `weak_ptr<D>`的指针`self_`。当调用`shared_from_this`函数时，会利用`self_`来构造`shared_ptr`，这种做法并不会触发`shared_ptr`创建`Control Block`的行为。不过，这种做法要求当前对象在调用`shared_from_this()`之前必须具备`Control Block`，否则将触发未定义行为，通常会抛出`bad_weak_ptr`异常。换言之，使用`shared_from_this`的对象最好也由`shared_ptr`管理，否则可能会引发异常。例子可见下面代码：

```C++
class Object: public enable_shared_from_this<Object>{
public:
    Object(const string& str):name(str){}
    void test(){
        auto ptr = shared_from_this();
        cout << ptr->name << endl;
    }    
private:
    const string name;
};

int main(){
    //Object* ptr = new Object("Hello");	//使用 new 句柄直接调用 test 函数将会引发 bad_weak_ptr 异常
    auto ptr = make_shared<Object>("Hello");
    ptr->test();
    //delete ptr;
}
```

#### weak_ptr 的内存布局及相关应用

`weak_ptr`通常作为`shared_ptr`的补充，它并不会改变`shared_ptr`的引用计数。它的大小和`shared_ptr`相同，且和对应的`shared_ptr`指向同一个控制块，不过与 shared_ptr 不同的是，`weak_ptr` 不能直接执行解引用操作。`weak_ptr`的常用操作主要有以下三个：

**作用一: 缓存优化**

通常对于访问数据库或是读取文件等耗时的操作，一般都会使用缓存的方式来提高效率。**如果使用`shared_ptr`来管理数据库连接或是文件句柄，则在缓存时要额外注意资源对象的生命周期问题**，而`weak_ptr`为我们提供了一套解决方案，例如：

```C++
std::unique_ptr<const Object> loadObject(ObjectID id);
std::shared_ptr<const Object> fastLoadObject(ObjectID id){
    static std::unordered_map<ObjectID, std::weak_ptr<const Object>> cache;
    auto objPtr = cache[id].lock(); 
    // to cached object (or null if object's not in cache)     
    if (!objPtr) { // if not in cache,         
        objPtr = loadObject(id); // load it
        cache[id] = objPtr; // cache it     
    }
    return objPtr;
}
```

使用`weak_ptr`来做缓存可以避免额外延长资源的生命周期，使得资源在使用结束后能够及时地被回收，同时**使用`weak_ptr`的`lock`函数可以判断当前的`weak_ptr`所指向的对象是否已被删除，如果被删除，则返回空的`shared_ptr`对象。如果没有被删除，则`weak_ptr`会提升为`shared_ptr`，进而我们就可以对其进行相关的操作**。这种让强弱引用相互搭配以避免影响资源回收的做法比较常见，典型有如 lua 语言中，使用对偶表示来实现私有性时，将外部表设置为弱引用键表，这样才能让 GC 在适当的时候能够顺利的回收内存。

**作用二：解决悬空指针问题**

关于如何使用`weak_ptr`解决悬空指针问题，主要有三种方法:

* 利用`weak_ptr`的`lock`函数对`weak_ptr`进行提升，具体的做法可见前面的**作用一:缓存优化**的内容；
* 利用`weak_ptr`构造`shared_ptr`，具体的做法可以见前面`enable_shared_from_this`的实现，这里不多做赘述；
* 利用`weak_ptr`的`expired`函数，如下：

```C++
void process(weak_ptr<Object> objPtr){
    //...
    if(objPtr.expired()){
        //do something
    }
    //...
}
```

前两种方法不仅涉及了对指针是否悬空的判断，还涉及到了对`Object`的操作。而在第三种方法中，`expired()`只能判断当前的`weak_ptr`是否悬空，但由于`weak_ptr`不能直接解引用，因此如果需要对`Object`进行相应的操作，就必须手动将`weak_ptr`提升为`shared_ptr`，这样就需要使用锁机制来保证原子性。

**作用三：解决环形引用问题**

<div style="text-align:center">
    <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/68c7521c-60c3-404f-a4df-3b424c0d2c54.png" width="80%">
</div>
<div class="image-caption" align="center">环形引用</div>

当上图中 B 也持有了一个指向 A 的 `shared_ptr `时，就会引发"环形引用"问题：A 和 B 之间的相互引用会使得它们之间彼此的 `reference count` 至少为1，这就使得 A 和 B 所占用的资源迟迟不能被回收，造成了内存的泄漏。解决这一问题的方法便是让 B 持有一个指向 A 的 `weak_ptr` 而非 `shared_ptr`。由于 `weak_ptr` 不会增加 `reference count`，因此 A 所占有的资源可以先于 B 被释放，一旦 A 释放了，那么 B 被释放也只是时间上的问题。

#### unique_ptr 与 PImpl Idiom 的相关问题

**unique_ptr 的类型**

默认 deleter 的`std::unique_ptr` 的大小与裸指针一致，而且大部分操作(例如解引用)采用了相同的指令，因此从效率及空间利用率上来讲，`std::unique_ptr`都可以代替裸指针使用。若用户采用了自定义的 deleter，则`std::unique_ptr` 的大小还会受到 deleter 的影响:

* 当 deleter 是函数指针时，`std::unique_ptr` 会增加 4 或 8 个字节的大小(取决于指针大小)
* 当 deleter 是函数对象时，`std::unique_ptr` 所增加的大小取决于函数对象中所存储的数据成员
* 当 deleter 是无捕获列表的 lambda 表达式时，`std::unique_ptr` 则不会增加额外的大小。	

因此在自定义 deleter 时要优先考虑无捕获列表的 lambda 表达式

`std::unique_ptr`提供了指向 T 类型数组的类型 —— `std::unique_ptr<T[]>`。根据定义方式的不同，`std::unique_ptr`也提供了不同的接口来针对不同的实现。例如数组型的 unique_ptr 不支持解引用运算(比如 * 和 ->)，但支持 [] 运算

另外，`std::unique_ptr`可以很方便的转化为 `std::shared_ptr`，且转换的过程会创建 `Control Block`, 但反之则不行

**unique_ptr 与 PImpl Idiom**

正如前面所说，deleter 作为 `unique_ptr` 类型的一部分，不仅体现在 deleter 的类型会影响 `unique_ptr` 所占用空间的大小，还体现在了 PImpl Idiom 上。 在 C++ 中，PImpl 是一种常用的技巧，合理运用可以减少编译时间，例如：

```C++
//Object.h
class Object {
public:
     Object();
     …
private:
     std::string name;
     std::vector<double> data;
     Gadget g1, g2, g3; // Gadget is some user-defined type
};
```

在上述代码中，每当 Gadget.h 发生变更时，所用使用到 Object的文件都需要重新编译。现在采用 `unique_ptr`和 pImpl 的方法来解除 Object 和 Gadget 依赖关系，如下：

```C++
//Object.h
class Object {
public:
    Object();
 	…
private:
	struct Impl; // declare implementation struct
	std::unique_ptr<Impl> pImpl;
};
//Object.cpp
#include "Object.h" 
#include "gadget.h"
#include <string>
#include <vector>
struct Object::Impl { // as before
	std::string name;
	std::vector<double> data;
    Gadget g1, g2, g3;
};
Object::Object() : pImpl(std::make_unique<Impl>()) {}
//client.cpp
#include "Object.h"
int main(){
    Object w; // error!
}
```

对上述代码进行编译，我们会得到一个错误信息：

```bash
$ error: invalid application of ‘sizeof’ to incomplete type ‘Object::Impl’
```

在上述代码中，当需要析构对象 w 时，由于我们没有为 `Object` 显式定义析构函数，因此编译器会为我们生成相应的析构函数并将其内联到相应的位置，并调用 `unique_ptr<Impl>` 的析构函数。由于 `unique_ptr<Impl>` 采用的是 default deleter，它会先对中所持有的`Impl*`进行静态类型检查`static_assert`，以确保`Impl*`指向了一个完全类型。不过**因为这段析构代码是由编译器生成并内联到了指定位置的，因此编译器看不到位于 `Object.cc` 文件中的 `Impl` 的详细定义，进而认为`Impl*`指向了一个不完全类型，触发了编译错误**

一旦明白了原因，那么解决方案就很清晰了：只需要在实现文件`Object.cc`的 Impl 定义的后面补上析构函数的定义即可，具体做法如下：

```C++
//Object.h
class Object { 
public:
    Object();
    ~Object(); // dtor is needed—see below
 	…
private:
	struct Impl; // declare implementation struct
	std::unique_ptr<Impl> pImpl;
};
//Object.cpp
#include "Object.h" 
#include "gadget.h"
#include <string>
#include <vector>
struct Object::Impl { // as before
	std::string name;
	std::vector<double> data;
    Gadget g1, g2, g3;
};
Object::Object() 
: pImpl(std::make_unique<Impl>()) // std::unique_ptr
{}
Object::~Object() = default;
//Client.cpp
#include "Object.h"
int main(){
    Object w; // no error
}
```

通过显式地定义了函数体，编译器在析构时就能够看到 `Impl` 的定义，自然也就不会产生 `delete incomplete type` 的错误了

而使用 shared_ptr 则没有上述的限制，例如：

```C++
//Object.h
class Object { 
public:
 Object();
 … // no declarations for dtor or move operations
private:
 struct Impl; 
 std::shared_ptr<Impl> pImpl; // std::shared_ptr
}; 
// Client.cpp
#include "Object.h"
int main(){
    Object w1;
	auto w2(std::move(w1)); // move-construct w2
	w1 = std::move(w2); // move-assign w1
}
```

上述代码可以正常通过编译并且运行。

这一切都源自于 `unique_ptr` 和 `shared_ptr`之间关于 deleter 的支持方式上的差异。对于 `unique_ptr` 而言，deleter 以合成（composition）的方式包含于 `unique_ptr` 之中，需要在编译期执行类型检查，这种的 deleter 被称之为 static deleter。而对于 `shared_ptr` 而言，deleter 以聚合（aggregation）的方式存在于`shared_ptr`之中，不需要在编译期执行检查，这种 deleter 被称之为 dynamic deleter。

关于 `unique_ptr` 和 `shared_ptr` 在不同场合下对类型的要求可见下表，其中 I 表示 `Incomplete Type`，C 表示 `Complete Type`，而 N/A 表示不支持该函数

|                                       | unique_ptr | shared_ptr |
| ------------------------------------- | ---------- | ---------- |
| default constructor : P()             | I          | I          |
| copy constructor  : P(const P&)       | N/A        | I          |
| move constructor : P(P&&)             | I          | I          |
| destructor : ~P()                     | C          | I          |
| P(A*)                                 | I          | C          |
| copy assignment : operator=(const P&) | N/A        | I          |
| move assignment : operator=(P&&)      | C          | I          |
| reset()                               | C          | I          |
| reset(A*)                             | C          | C          |




#### Reference
[0]. 《Effective Modern C++》
[1]. 《Linux 多线程服务端编程 —— 使用 muduo C++ 网络库》
[2]. [How std::enable_shared_from_this::shared_from_this works](https://stackoverflow.com/questions/34061515/how-stdenable-shared-from-thisshared-from-this-works)
[3]. [Is std::unique_ptr required to know the full definition of T?](https://stackoverflow.com/questions/6012157/is-stdunique-ptrt-required-to-know-the-full-definition-of-t)
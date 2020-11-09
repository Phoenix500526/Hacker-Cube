---
title: 对 muduo 网络库单例模式的思考与实践(上)
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-09 14:16:56
password:
summary:
tags: [muduo网络库, C++, Singleton]
categories: muduo源码剖析
---

#### 文前导读
本文是 muduo 网络库源码剖析系列文章的第七篇文章，主要涉及了对 muduo 网络库中单例模式的分析，以及 C++11 标准下一种更加简洁的单例模式实现，其中包含了以下内容：
> * 对 muduo 网络库下单例模式的思考
>   * 线程全局单例模式 Singleton 的实现
>   * 线程局部单例模式 ThreadLocalSingleton 的实现
> * tmuduo 中单例模式的实现

本文中所涉及的代码来源于我的个人项目：tmuduo。本着“纸上学来终觉浅，绝知此事要躬行”的想法，我将 muduo 网络库重新实现了一遍，并在上面验证了自己的不少猜想。项目的仓库地址为 git@github.com:Phoenix500526/Tmuduo.git, 欢迎各位fork + star，一起加入学习
<!-- more -->

#### 对 muduo 网络库下单例模式的思考

###### Singleton 的实现

在 muduo 网络库的源代码中，单例模式分为两种，一种是线程全局单例模式，另一种是线程局部单例模式，相关的代码分别存放在 base/Singleton.h 和 base/ThreadLocalSingleton.h 文件中。我们先看下 Singleton 的实现
```C++
//Singleton.h
template<typename T>
class Singleton : noncopyable{
 public:
  Singleton() = delete;
  ~Singleton() = delete;
  static T& instance()
  {
    pthread_once(&ponce_, &Singleton::init);
    assert(value_ != NULL);
    return *value_;
  }

private:
  static void init()
  {
    value_ = new T();
    if (!detail::has_no_destroy<T>::value)
    {
      ::atexit(destroy);
    }
  }
  static void destroy()
  {
    // T_must_be_complete_type 的作用是检测 T 是不是不完全类型
    // 若 T 是不完全类型(仅有声明没有定义)，则 sizeof(T) = 0
    // 则此时 T_must_be_complete_type 相当于 char[-1]
    // 利用此类型定义了一个 dummy 对象会引发变异错误。
    typedef char T_must_be_complete_type[sizeof(T) == 0 ? -1 : 1];
    T_must_be_complete_type dummy; (void) dummy;

    delete value_;
    value_ = NULL;
  }
private:
  static pthread_once_t ponce_;
  static T*             value_;
};
template<typename T>
pthread_once_t Singleton<T>::ponce_ = PTHREAD_ONCE_INIT;
template<typename T>
T* Singleton<T>::value_ = NULL;
```
从上述代码可以看出，Singleton 是一个模板类，且用户只能调用该类的静态成员函数`instance`。当用户调用 `Singleton<T>::instance()` 时，muduo 调用 `Singleton<T>::init` 进行 T 的初始化，而 `pthread_once` 保证了这一过程仅会在首次调用 `instance` 方法时发生。对于其中 `has_no_destroy` 的具体实现，我们先按下不表，留到下一篇文章再回来讨论。现在只需要知道， `has_no_destroy<T>::value` 是用来判断类型 T 当中是否声明了函数 `no_destroy` 即可。因此，如果 T 中没有声明 `no_destroy` 函数，那么 `init` 会调用 `atexit` 将 `destroy` 注册到进程当中，这样当进程退出时就会自动执行后 `destroy` 进行对象的析构工作。心细的同学应该很容易发现，**如果单例对象中声明了 `no_destroy` 函数，那毫无疑问地会导致内存的泄露**。

###### ThreadLocalSingleton 的实现

在大体知道了 Singleton 的实现后，我们再来看看 ThreadLocalSingleton 的实现：
```C++
template<typename T>
class ThreadLocalSingleton : noncopyable{
public:
  ThreadLocalSingleton() = delete;
  ~ThreadLocalSingleton() = delete;
  static T& instance(){
    if (!t_value_)
    {
      t_value_ = new T();
      deleter_.set(t_value_);
    }
    return *t_value_;
  }

  static T* pointer(){
    return t_value_;
  }

private:
  static void destructor(void* obj){
    assert(obj == t_value_);
    typedef char T_must_be_complete_type[sizeof(T) == 0 ? -1 : 1];
    T_must_be_complete_type dummy; (void) dummy;
    delete t_value_;
    t_value_ = 0;
  }

  class Deleter{
   public:
    Deleter(){
      pthread_key_create(&pkey_, &ThreadLocalSingleton::destructor);
    }

    ~Deleter(){
      pthread_key_delete(pkey_);
    }

    void set(T* newObj){
      assert(pthread_getspecific(pkey_) == NULL);
      pthread_setspecific(pkey_, newObj);
    }

    pthread_key_t pkey_;
  };
  static __thread T* t_value_;
  static Deleter deleter_;
};
template<typename T>
__thread T* ThreadLocalSingleton<T>::t_value_ = 0;
template<typename T>
typename ThreadLocalSingleton<T>::Deleter ThreadLocalSingleton<T>::deleter_;
```
在上述代码中，我们可以看到，ThreadLocalSingleton 也是一个模板类，用户只能调用类中的静态方法 `instance` 以及 `pointer`。由于和 Singleton 的应用场景不同， ThreadLocalSingleton 只需要通过判断 `t_value_` 是否为空，就可以实现仅在首次调用 `instance` 方法进行初始化。基本的流程和前面 Singleton 是差不多的，其中最大的不同在于 ThreadLocalSingleton 多了一个 Deleter class，这是因为 `t_value_` 由 GCC 关键字 `__thread` 所修饰，因此不能自动调用析构函数，需要额外定义一个 Deleter 来完成这份工作。`deleter_` 实际上相当于一个 RAII 对象，在调用 `instance` 时将 `t_value_` 交由 `deleter_` 保管，当线程退出时析构 `deleter_` 时，会执行 `ThreadLocalSingleton::destructor` 来析构 `t_value_`

#### tmuduo 中单例模式的实现
在讨论 tmuduo 如何实现单例模式之前，我们需要先思考一个问题，一个单例对象需要符合哪些特质才能算得上是一个合格的单例？我想至少有以下几点：
> * 该对象必定是不可拷贝的
> * 用户只能够通过 instance 方法获得唯一的实例，且在若干次调用 instance() 方法中，仅会在首次调用时进行初始化
> * 必须是资源安全的，在程序退出时必须能够正确地析构，不会造成任何的资源泄露
> * 若是线程全局单例模式，则还需要考虑线程安全的问题。若是线程局部单例模式，则需要考虑线程局部存储的问题

第一个问题非常好解决，我们只需要继承 noncopyable 标签类即可。真正需要注意的其实是第二和第三个问题。
如果看过我之前的文章 《对 muduo 网络库中的线程模型的思考与实践》，应该还记得其中提到了三个变量存储期概念，分别是 static stroage duration，thread storage duration 以及 dynamic storage duration。任何被声明为 static 或 thread_local 的局部变量具有 static storage duration 或 thread storage duration(这里的或指的是互斥或而非逻辑或，以下或如未特殊声明，均代表互斥或)，且他们都不被分配在堆内存上。
另外，在 C++11 的标准中，关于被声明为 `static` 或 `thread_local` 的局部变量，有如下的规定：
> Variables declared at block scope with the specifier static or thread_local (since C++11) have static or thread (since C++11) storage duration but are **initialized the first time control passes through their declaration (unless their initialization is zero- or constant-initialization, which can be performed before the block is first entered). On all further calls, the declaration is skipped.**
>
> If the initialization throws an exception, the variable is not considered to be initialized, and initialization will be attempted again the next time control passes through the declaration.
>
> If the initialization recursively enters the block in which the variable is being initialized, the behavior is undefined.
>
> **If multiple threads attempt to initialize the same static local variable concurrently, the initialization occurs exactly once** (similar behavior can be obtained for arbitrary functions with std::call_once).

上述 C++11 标准保证了 `static` 或 `thread_local` 修饰局部变量只会在程序的控制流首次进入相应的块作用域才进行实例化工作，而且在实例化成功后，在程序或线程退出时能够自动调用相应的析构函数。另外，由于 `static` 或 `thread_local` 修饰的局部变量并不存储在堆内存当中，因此自然也不会造成任何的内存泄露。最后，对于线程全局单例模式，标准保证了 static local variable 的初始化是线程安全的，而 thread_local 本身就代表了线程局部存储。有了 C++11 标准的保驾护航，第二、第三和第四个问题将不再是问题，我们可以以一种非常简洁的方式来实现 tmuduo 中的单例模式

```C++
// Singleton.h 的实现
template <typename T>
class Singleton : noncopyable {
 public:
  Singleton() = delete;
  ~Singleton() = delete;
  static T& instance() {
    static T instance;
    return instance;
  }
};

// ThreadLocalSingleton.h 的实现
template <typename T>
class ThreadLocalSingleton : noncopyable {
 public:
  ThreadLocalSingleton() = delete;
  ~ThreadLocalSingleton() = delete;
  static T& instance() {
    thread_local T instance;
    return instance;
  }
};
```
为了验证 Singleton 以及 ThreadLocalSingleton 的正确性，我在自身代码的基础上运行 muduo 中的测试。与 muduo 本身的测试结果相当，具体测试代码可见:tmuduo/test/Singleton_test.cc 以及 tmuduo/test/ThreadLocalSingleton_test.cc
---
title: clang 的线程安全注解TSA
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-05 17:42:56
password:
summary:
tags: [clang, TSA, 多线程]
categories: 瑞士军刀
---
#### 文前导读
本文主要涉及到 clang 编译器的线程安全注解功能(Thread Safety Annotation, 以下简称TSA)，主要包含以下内容:
> * 什么是 TSA？
> * TSA 常用的宏定义(按照修饰对象来分类)
> * 使用 TSA 的注意事项

严格来讲，这一篇文章并不涉及 TSA 的所有宏定义，只是解释了一些基本的概念和常用的几个宏定义。我个人认为对于研发工具的学习应当从实际应用出发，先了解常用的功能如何使用，并在后续的开发中陆续补充新的用法。想学语言一样学习开发工具，一上来就抱着文档统统啃下的做法并不务实。
<!-- more -->

#### 什么是 TSA ?
在进行 C++ 并发编程中，最原始的模型应该就是基于锁来对共享数据进行保护的并发模型了。由于这种模型很容易出问题，因此需要采用各种动态的或静态的检查工具来避免程序出现 data race。而 clang 的 TSA 就是一个简单易用的静态检查工具。我们通过代码注解(annotation)的方式来告知编译器，哪些成员变量或成员函数受到了哪个 mutex 的保护。这样当开发者忘记加锁或尝试重复加锁时，编译器能够及时发出警告。这在现代程序开发时非常有用，因为一个程序往往开发和维护的人未必是同一个人。注解不仅能帮助编译器理解开发者的意图，还可以帮助维护者理解这一意图，这样也就避免了犯非常低级的错误。

#### TSA 常用的宏定义有哪些？
clang 提供的 TSA 有两种用法，一种属于 GNU 风格，例如\_\_attribute\_\_((...))), 另一种则是属于 C++11 风格，例如[[...]],但不管是哪一种风格，一种推荐的做法是利用宏来包装这些相应的关键字，这样如果编译器不是 clang 的话，这些宏会被自动置空。接下来对 TSA 部分的介绍也会按照这种宏定义模式来讨论，并通过代码注释来进行相应的说明。更多的宏定义可以参考 clang 的官方文档[1]

###### 修饰类、结构体以及`typedef`别名的宏
宏`CAPABILITY`表明某个类对象可以当作 capability 使用，其中 x 的类型是 string，能够在错误信息当中指出对应的 capability 的名称, 而宏`SCOPED_CAPABILITY`用于修饰基于 RAII 实现的 capability。
```C++
#define CAPABILITY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(capability(x))
#define SCOPED_CAPABILITY \
  THREAD_ANNOTATION_ATTRIBUTE__(scoped_lockable)

class CAPABILITY ( "mutex" ) Mutex {
public :
void lock ( ) ACQUIRE ( t hi s ) ;
void readerLock ( ) ACQUIRE SHARED( t hi s ) ;
void unlock ( ) RELEASE( t hi s ) ;
void readerUnlock ( ) RELEASE SHARED( t hi s ) ;
} ;
```
 这里简单地说明一下 capability： capability 是 TSA 中的一个概念，用来为临界资源的访问提供相应的保护。这样讲可能有点抽象，你可以简单地将其理解成为一个标签，这个标签可以被贴到任何锁上面，不论它是标准库的 mutex 还是你自己实现的 Mutex。一旦一个锁被贴上了这个标签，TSA 就会对这个锁进行重点关注。因此语句 `class CAPABILITY ( "mutex" ) Mutex` 表明 Mutex 类型的对象可以作为一个 capability，而且它的名称就是 "mutex"。

###### 修饰数据成员的宏 GUARDED_BY
宏`GUARD_BY`用于修饰对象，表明该对象需要受到 capability 的保护, 而宏`PT_GUARDED_BY(mutex)` 则用于修饰指针类型变量，在更改指针变量**所指向的内容**前需要加锁，否则发出警告。
```C++
#define GUARDED_BY(x) \
	THREAD_ANNOTATION_ATTRIBUTE__(guarded_by(x))
#define PT_GUARDED_BY(x) \
	THREAD_ANNOTATION_ATTRIBUTE__(pt_guarded_by(x))
//示例用法
int *p1             GUARDED_BY(mu);
int *p2             PT_GUARDED_BY(mu);
unique_ptr<int> p3  PT_GUARDED_BY(mu);

void test() {
  p1 = 0;             // Warning!
  *p2 = 42;           // Warning!
  p2 = new int;       // OK.
  *p3 = 42;           // Warning!
  p3.reset(new int);  // OK.
}
```

###### 修饰函数/方法(成员函数)的宏
宏`REQUIRES`声明调用线程必须拥有对指定的 capability 具有独占访问权。可以指定多个 capabilities。函数/方法在访问资源时，必须先上锁，再调用函数，然后再解锁(注意，不是在函数内解锁)
```C++
#define REQUIRES(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(requires_capability(__VA_ARGS__))
//示范用法
Mutex mu1, mu2;
int a GUARDED_BY(mu1);
int b GUARDED_BY(mu2);
void foo() REQUIRES(mu1, mu2) {
  a = 0;
  b = 0;
}
void test() {
  mu1.Lock();
  foo();         // Warning!  Requires mu2.
  mu1.Unlock();
}
```

宏`ACQUIRE`表示一个函数/方法需要持有一个 capability，但并不释放这个 capability。调用者在调用被 ACQUIRE 修饰的函数/方法时，要确保没有持有任何 capability，同时在函数/方法结束时会持有一个 capability(加锁的过程发生在函数体内),而宏`RELEASE` 则和宏`ACQUIRE` 作用相反，它们表示调用方在调用该函数/方法时需要先持有锁，而当函数执行结束后会释放锁(释放锁的行为发生在函数体内)，具体例子如下：
```c++
#define ACQUIRE(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(acquire_capability(__VA_ARGS__))
#define RELEASE(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(release_capability(__VA_ARGS__))
//示范用法
MutexLock mu;
class MyClass{
public:
	void doSomething(){
		cout << "x = " << x << endl;
	}
	MyClass() = default;
	void init(const int a){
		x = a;
	}
	void cleanup(){
		x = 0;
	}
private:
	int x;
};
MyClass myObject GUARDED_BY(mu);

void lockAndInit(MyClass& myObject) ACQUIRE(mu) {
  mu.lock();
  myObject.init(10);
}

void cleanupAndUnlock(MyClass& myObject) RELEASE(mu) {
  myObject.cleanup();
}                          // Warning!  Need to unlock mu.

void test() {
	MyClass myObject;	//局部对象掩盖了全局的 myObject 对象，而全局的 myObject 对象受到了 mu 的保护
	lockAndInit(myObject);
	myObject.doSomething();
	cleanupAndUnlock(myObject);
	myObject.doSomething(); 
}

int main(void){
	test();
	myObject.doSomething();	// Warning! MyObject is guarded by mu
	return 0;
}
```

宏`EXCLUDES`用于显式声明函数/方法不应该持有某个特定的 capability。由于 mutex 的实现通常是不可重入的，因此 EXCLUDES 通常被用来预防死锁
```C++
#define EXCLUDES(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(locks_excluded(__VA_ARGS__))
//实例用法
Mutex mu;
int a GUARDED_BY(mu);
void clear() EXCLUDES(mu) {
  mu.Lock();
  a = 0;
  mu.Unlock();
}
void reset() {
  mu.Lock();
  clear();     // Warning!  Caller cannot hold 'mu'. 
  mu.Unlock();
}
```

宏`ASSERT_*`表示在运行时检测调用线程是否持有 capability，主要有以下两个
```C++
#define ASSERT_CAPABILITY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(assert_capability(x))
#define ASSERT_SHARED_CAPABILITY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(assert_shared_capability(x))
```

宏`NO_THREAD_SAFETY_ANALYSIS`表示关闭某个函数/方法的 TSA 检测，通常只用于两种情况：1，该函数/方法可以被做成非线程安全；2、函数/方法太过复杂，TSA 无法进行检测
```C++
#define NO_THREAD_SAFETY_ANALYSIS \
  THREAD_ANNOTATION_ATTRIBUTE__(no_thread_safety_analysis)__
```

宏`RETURN_CAPABILITY`通常用于修饰那些被当作 capability getter 的函数，这些函数会返回 capability 的引用或指针
```C++
#define RETURN_CAPABILITY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(lock_returned(x))
```

#### 使用 TSA 的一些注意事项
不过 TSA 是静态检查工具，因此对它的期望不应当过高，在使用的过程当中依然有一些注意事项需要了解：

1. 一般而言，注解通常被当作函数接口的一部分进行解析，因此最好放在头文件当中，而不是 .cc 文件当中。(NO_THREAD_SAFETY_ANALYSIS 除外)

2. TSA 的解析与检测主要在编译期间执行，因此不能对运行时才能确定的条件语句进行检测。例如以下做法是错误的：
```C++
bool b = needsToLock();
if (b) {
	mu.Lock();
}
...  // Warning!  Mutex 'mu' is not held on every path through here.
if (b) {
	mu.Unlock();
}
```

3. TSA 仅依赖于函数的属性的声明，它并不会将函数调用展开并内联到指定位置，因此下面的做法也是错误的(它使用 mu.lock() 进行显式的上锁，却希望使用函数调用来进行解锁)：
```C++
template<class T>
class AutoCleanup {
	T* object;
	void (T::*mp)();
public:
     AutoCleanup(T* obj, void (T::*imp)()) : object(obj), mp(imp) { }
     ~AutoCleanup() { (object->*mp)(); }
   };
   
   Mutex mu;
void foo() {
   mu.Lock();
   AutoCleanup<Mutex>(&mu, &Mutex::Unlock);
     // ...
}  // Warning, mu is not unlocked.
```

4. TSA 无法追踪指针的指向，因此当两个指针指向一个互斥锁时，会导致警告的发生，如下：
```C++
class MutexUnlocker {
     Mutex* mu;
public:
     MutexUnlocker(Mutex* m) RELEASE(m) : mu(m)  { mu->Unlock(); }
     ~MutexUnlocker() ACQUIRE(mu) { mu->Lock(); }
};
   
Mutex mutex;
void test() REQUIRES(mutex) {
	{
       MutexUnlocker munl(&mutex);  // unlocks mutex
       doSomeIO();
     }                              // Warning: locks munl.mu
}
```
mun1 中的成员变量 mu 在析构的时候被释放，但 TSA 并不能意识到 mutex 与 mun1.mu 指向了同一个互斥锁。因此，会显示出警告信息：`mun1.mu unlocked`。

#### 参考资料
1. [Clang 12 documentation - Thread Safty Analysis](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html#reference-guide) 
2. [clang的线程安全分析模块 thread safety analysis](https://my.oschina.net/u/4397303/blog/3281876)
3. [clang_thread_safety_annotation.pdf](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/42958.pdf)
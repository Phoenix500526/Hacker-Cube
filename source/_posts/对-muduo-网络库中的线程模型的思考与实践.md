---
title: 对 muduo 网络库中的线程模型的思考与实践
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-05 20:22:56
password:
summary:
tags: [muduo网络库, C++, 多线程]
categories: muduo源码剖析
---
#### 文前导读
本文是 muduo 网络库源码剖析系列文章的第四篇文章，主要涉及了 muduo 网络库中对线程模型的一些设计亮点，以及我如何使用标准线程库来对其进行重新实现，其中包含了以下内容：
> * 变量存储期的概念
> * pthread_t、pid_t 以及 thread::id 的比较及它们各自的问题
> * 如何更加简单方便地封装线程上下文
> * 用标准线程库实现 tmuduo 的线程模型

本文中所涉及的代码来源于我的个人项目：tmuduo。本着“纸上学来终觉浅，绝知此事要躬行”的想法，我将 muduo 网络库重新实现了一遍，并在上面验证了自己的不少猜想。项目的仓库地址为 git@github.com:Phoenix500526/Tmuduo.git, 欢迎各位fork + star，一起加入学习
<!-- more -->

#### 一、变量存储期的概念
C++ 标准中，变量一共有四种不同的存储周期(storage duration)，分别是：
* automatic storage duration: 拥有此生命周期的对象会在**代码块的开始处**分配内存，并在**代码块的结尾处**回收内存。
    * 任何没有被声明为 `static`, `extern` 或者 `thread_local` 的**局部变量**都具备此生命周期
* static storage duration: 拥有此生命周期的对象会在**程序开始时**分配内存，并在**程序结束时**回收内存。
    * 任何声明在命名空间中的对象以及被修饰为 static 和 extern 的对象均具有此生命周期
    * 在整个程序运行过程中始终只有一个实例
* thread storage duration: 拥有此生命周期的对象会在**线程开始时**分配内存，并在**线程结束时**回收内存。
    * thread_local 可以和 static 以及 extern 连用，此时 static 和 extern 都只表示链接性。
    * 每个线程都有自己的一份 thread_local 对象实例。
* dynamic storage duration：拥有此生命周期的对象会在**动态申请内存**处分配内存，并在**析构**时回收内存。

###### `thread_local` 与 `__thread` 的比较
`thread_local` 关键字总是很容易让人联想到另一个和它非常类似的关键字`__thread`，二者之间究竟有何区别？
先来看看`__thread`关键字，\_\_thread 是 GCC 内置的线程局部存储设施，它的实现非常高效，其存储效率可以媲美全局变量。然而使用起来却有以下限制：
* 只能修饰 POD 类型，不能修饰 class 类型(因为 \_\_thread 不会自动调用构造和析构函数)
* 只能使用编译期常量进行初始化
* \_\_thread 只能修饰全局变量和静态变量，但是不能修饰局部变量或者 class 的普通成员变量
\_\_thread 变量在每个线程中都有一份独立实体，各个线程中的变量值互不干扰。利用这个特点，\_\_thread关键字常常被用来修饰那些“值可能会变，带有全局性，但是又不值得用全局锁保护的变量”。
而 C++11 的 `thread_local` 在使用限制上则要宽松不少：
* `thread_local` 可以修饰非 POD 类型的变量，它会自动调用构造函数和析构函数
* `thread_local` 除了全局变量以外，还可以修饰局部变量，并且只会在线程生命周期中只会有一个实例。

综上所述，如果你使用 C++11 标准进行开发，使用 `thread_local` 会比使用 `__thread` 要好用不少。


#### 二、一种更好的标识线程的方法

如果阅读过 Linux 内核源码实现的朋友应该知道，其实在 Linux 系统下，进程和线程之间并无本质区别：二者实际上都是用同一个结构体 `task_struct` 来表示一个执行任务的实体。虽然二者的创建方式各不相同，进程创建调用 `fork` 系统调用，而线程创建主要通过 `pthread_create` 函数，但这两个方法最终都会调用到 `do_fork` 来完成具体的创建操作，最大的区别仅在于传进的参数不同。换句话讲，**Linux 系统中所谓的线程，只不过是看起来像线程，其本质是和其他进程共享部分资源(虚拟内存、文件描述符以及页表等)的进程**。

既然在 Linux 系统下，进程和线程在本质上没有区别，那么是否可以使用用来描述进程的 `pid_t` 类型来实现 Linux 系统下的线程标识呢？答案是肯定的。但在这之前，我们先来看看为何要如此大费周章地完成这一事情。在 Linux 系统下, 我们想要去标识一个线程,可供选择的有 `pthread_t`, `pid_t` 以及 `thread::id`. 我们先来看看 `pthread_t` 类型.POSIX 线程库中提供了 `pthread_self` 函数来获得一个线程的线程 id. 不过由于某些原因, `pthread_t` 的类型是不确定的,可能是整数、指针或是结构体, 这取决于你使用的具体系统。由于类型的不确定，POSIX 提供了 `pthread_equal` 函数来比较两个线程标识是否相同，这带来很多问题：
* 如果不知道具体类型，自然也就难以打印输出。对于 `printf` 函数，不确定类型自然无法打印。对于 C++ 的流，虽然使用时可以不考虑类型差异，但你也很难对输出的结果有合理的预期。
* 无法进行大小和相等性的比较，因此不能过作为 hash 的 key 进行存储
* POSIX 没有提供一个专门用来表示非法线程的专属ID.
* `pthread_t` 只能保证在当前进程内唯一，并不能保证全局唯一，甚至连在同一个进程内先后创建的不同线程都可能拥有相同的线程ID

接下来再来看看 `pid_t`, 和 `pthread_t` 相比，它具有以下特点：
* 类型明确，pid_t 就是一个小整数类型，这也就意味着你可以很方便地将其打印到日志中，同时可以对其值有合理的预期(不必去猜测这个打印出来的 id 到底是数字还是指针)
* 在现代 linux 系统中，`pid_t` 代表了任务调度 id，而线程是任务的基本调度单位，因此可以很方便地将它们对应起来，这样就可以利用 linux 的 /proc 系统来获取相应的信息
* linux 对 `pid_t` 的分配采用了轮回递增的方法，因此任何时刻都是全局唯一的，短时间内也不会有重复
* 0 代表非法值，因为 linux 中的第一个进程的 pid 是 1；
虽然使用 `pid_t` 来标识线程有上述的好处，不过 linux 并没有提供相应的系统调用来完成这一工作，不过我们依然可以利用间接系统调用 `syscall` 来绕开这一限制，这在接下来的 一种封装线程上下文的简单尝试** 中会展示。

最后来看看`thread::id`。在 thread 标准库中也提供了 `thread::id` 来为进程提供唯一标识。`thread::id` 相对于 `pthread_t` 有以下特点:
* `thread::id` 类型明确，一方面重载了流操作运算符，可以使用流进行输出，也可以利用流将其转换为字符串类型，另一方面标准库也提供了 std::hash 来对 `thread::id` 进行散列，这样也可以得到一个类型为 `std::size_t` 的值，两种转换方法如下：
    ```C++
    // thread::id 转换为字符串类型
    auto myid = this_thread.get_id(); 
    stringstream ss; 
    ss << myid; 
    string mystring = ss.str();
    // thread::id 转换为 std::size_t 类型
    std::hash<std::thread::id>()(std::this_thread::get_id())
    ```
* 标准库提供了一个 `id()` 方法，能够获得一个可以用来标记 non-joinable 的线程，也就相当于有了判定非法线程的方法。
* 标准库提供了比较运算符，可以对 `thread::id` 进行比较

在了解了上述三种标识线程的方法后，我们也很容易根据自己的需求挑选出合适的线程标识类型。如果只考虑 linux 系统开发而不考虑可移植性，那么首选 `pid_t` 作为线程 
ID 的类型。如果有移植性的需要，那么可以考虑使用 `thread::id`。由于 muduo 网络库本身只考虑 linux 系统，因此采用了 pthread + pid_t 的方式来实现其线程模型。接下来，我会使用 thread + pid_t 的方式重新实现对应的线程模型。


#### 三、一种封装线程上下文的简单尝试
我曾经分析过 skynet 框架中的多线程模型(见《skynet 源码阅读笔记 —— 消息调度机制》一文）。skynet 与 muduo 类似，都是采用了基于回调的 Reactor 模型。由于 skynet 采用 C 语言实现。C 语言中没有闭包的概念，无法方便地实现可调用对象，因此只能定义回调接口，然后定义业务逻辑处理函数来实现接口，并通过回调函数的参数传递上下文(如 skynet 中的 skynet_context)。本来我是想要按照这种方式，利用类来封装线程上下文，并通过传参的方式来传递上下文。但是我在看到了 muduo 网络库的 CurrentThread.{cc,h} 的实现后，感觉又学到了一种新的封装线程上下文的方法：利用 `thread_local` 来存储相关的上下文(在这里主要是线程id和线程名称)，而利用命名空间来"封装"操作上下文的方法。这种做法非常巧妙，相当于构造了一个生命周期贯穿于线程始末的闭包，它有以下的好处：
* 函数无需传参亦可访问到自身线程上下文信息。如果采用类的方式进行封装，那么在线程中的低阶函数就必须要以传参的方式去获得上下文。
* 使用 `thread_local` + 命名空间的方法，可以将变量声明为全局变量而无需担心命名污染。全局变量往往存取效率更高
* 比起类的封装，命名空间封装函数的方式在结构上更加松散， 松散的结构虽然降低一定的安全性，但是也带来了灵活性的大大提升。

```C++
//CurrentThread.h
namespace tmuduo {
namespace detail {
//利用间接系统调用获得线程 id
inline pid_t gettid() { return static_cast<pid_t>(::syscall(SYS_gettid)); }

}  // namespace detail

namespace CurrentThread {

extern thread_local pid_t t_cachedTid;
extern thread_local const char* t_threadName;

// cacheTid、isMainTHread、sleepUsec均定义在 Thread.cc 文件中，
// 因为需要拿到对应线程的 ID
void cacheTid();
bool isMainThread();
void sleepUsec(int64_t usec);  // for testing

inline pid_t tid() {
  if (__builtin_expect(t_cachedTid == 0, 0)) {
    cacheTid();
  }
  return t_cachedTid;
}

inline const char* name() { return t_threadName; }
}  // namespace CurrentThread
}  // namespace tmuduo

//CurrentThread.cc
namespace tmuduo {
namespace CurrentThread {

thread_local pid_t t_cachedTid = 0;
thread_local const char* t_threadName = "unknown";

void cacheTid() {
  if (t_cachedTid == 0) {
    //获取线程的 pid_t
    t_cachedTid = detail::gettid();
  }
}

bool isMainThread() { return tid() == ::getpid(); }

void sleepUsec(int64_t usec) {
  static const int kMicroSecondsPerSecond = 1000 * 1000;
  struct timespec ts = {0, 0};
  ts.tv_sec = static_cast<time_t>(usec / kMicroSecondsPerSecond);
  ts.tv_nsec = static_cast<long>(usec % kMicroSecondsPerSecond * 1000);
  ::nanosleep(&ts, NULL);
}

}  // namespace CurrentThread
}  // namespace tmuduo
```
在上述代码中，`gettid`展示了使用 `syscall` 来获取线程的一般方法(这也是 linux 中 man 关于 syscall 的一个示例用法)。另外，为了避免反复调用系统调用带来的开销，`CurrentThread` 采用 `t_cachedTid` 缓存 `gettid` 的执行结果，因此性能无忧。另外，`__builtin_expect` 是 GCC 的一个内置函数，其作用是供程序员将分支信息提供给编译器，以方便编译器调整取指令的顺序进行优化，这样可以减少 cache 产生控制冒险。

#### 四、tmuduo 线程类 Thread 的实现
在讨论 tmuduo 如何使用标准线程库来封装线程之前，我们需要先明确几个问题：
* 为什么要封装线程对象？thread 不好用吗？
* 标准线程库比起 POSIX 线程库好在哪？
对于第一个问题，之所以要封装线程，是因为我们希望让线程在初始化的过程当中去执行一些簿记工作，这样有利于我们进行后期调试。由于标准库不会假定你要做何种工作，因此我们需要自己为 thread 对象做一层薄薄的封装
对于第二个问题：标准线程库参照了 java 中线程库的设计，历史包袱远远少于 POSIX 线程库，淘汰掉了相当一部分不合理的接口。其次标准库本身与系统无关，具有良好的移植性。最后，标准线程库中的许多函数实现是有语言标准保证的，我们无需做过多额外工作，因此能够减少不少编码工作。

举个例子，在 muduo 网络库中，Thread 采用了 pthread 进行设计，将 Thread 的构造函数和启动函数分离开。这主要是因为当执行 `pthread_create` 函数创建线程后，线程并不一定是马上开始运行的，尽管当 CPU 低负载的大部分情况下，线程是马上开始运行的。但是当 CPU 负载提高，并发度增加时，有可能会出现线程创建完成后返回，但是线程函数并为开始执行，而此时主线程去访问新线程时就会产生错误。因此，muduo 网络库中的 Thread 对象为了杜绝这种情况，将构造和启动函数分离，并在启动函数中使用了 CountDownLatch 进行同步，只有当创建好的线程开始运行时，start 函数才会返回。而标准库则保证在 thread 的构造函数返回时，线程函数已经开始运行。(具体可见我在 stackoverflow 上的一个讨论：[Is that the thread object can be moved in C++ 11 reasonable?](https://stackoverflow.com/questions/64461444/is-that-the-thread-object-can-be-moved-in-c-11-reasonable))
在明确了构造即启动的语义后，Thread 中可以减少很多用于同步变量以及函数，例如不再需要 `start` 函数，只需要实现构造函数即可，又例如不再需要用于标识线程运行状态的 `started_`，因为我们可通过 `joinable` 来判断一个线程的状态。下面我们来看代码实现：

```C++
//Thread.h
class Thread : noncopyable {
 public:
  using ThreadFunc = std::function<void()>;
  Thread(ThreadFunc func, const std::string& name = std::string());
  ~Thread();
  Thread(Thread&& rhs) noexcept;
  Thread& operator=(Thread&& rhs) noexcept;
  void join();
  static inline int numCreated() { return numCreated_.load(); }

 private:
  //线程实体
  std::thread thread_;
  static std::atomic<int> numCreated_;
  void runInThread(ThreadFunc func, const std::string& name);
};

//Thread.cc
std::atomic<int> Thread::numCreated_(0);

void Thread::runInThread(ThreadFunc func, const std::string& name) {
  CurrentThread::t_cachedTid = CurrentThread::tid();
  if (name.empty()) {
    char buf[32];
    snprintf(buf, sizeof buf, "Thread%d", Thread::numCreated());
    CurrentThread::t_threadName = buf;
  } else {
    CurrentThread::t_threadName = name.c_str();
  }
  //::prctl(PR_SET_NAME, threadName)：表示用 threadName
  //为当前线程命名，threadName 的长度
  //不得超过 16 bytes。当名字长度超过 16 个字节时会默认截断
  ::prctl(PR_SET_NAME, CurrentThread::t_threadName);
  try {
    func();
    CurrentThread::t_threadName = "finished";
  } catch (const Exception& ex) {
    CurrentThread::t_threadName = "crashed";
    fprintf(stderr, "exception caught in Thread %s\n", name.c_str());
    fprintf(stderr, "reason: %s\n", ex.what());
    fprintf(stderr, "stack trace: %s\n", ex.stackTrace());
    abort();
  } catch (const std::exception& ex) {
    CurrentThread::t_threadName = "crashed";
    fprintf(stderr, "exception caught in Thread %s\n", name.c_str());
    fprintf(stderr, "reason: %s\n", ex.what());
    abort();
  } catch (...) {
    CurrentThread::t_threadName = "crashed";
    fprintf(stderr, "unknown exception caught in Thread %s\n", name.c_str());
    throw;  // rethrow
  }
}

Thread::Thread(ThreadFunc func, const std::string& name)
    : thread_(&Thread::runInThread, this, std::move(func), name) {
  ++numCreated_;
}

Thread::Thread(Thread&& rhs) noexcept : thread_(std::move(rhs.thread_)) {}

Thread& Thread::operator=(Thread&& rhs) noexcept {
  if (this != &rhs) {
    thread_ = std::move(rhs.thread_);
  }
  return *this;
}

void Thread::join() {
  if (thread_.joinable()) {
    --numCreated_;
    thread_.join();
  }
}

Thread::~Thread() {
  if (thread_.joinable()) {
    --numCreated_;
    thread_.detach();
  }
}
```

#### 测试结果
上述代码的实现经过测试，运行良好。由于测试代码比较冗长，且案例覆盖上存在重复的部分，这里就不贴出代码。具体的测试代码位于 tmuduo/test/Thread_test.cc 中，编译后的可执行文件为 tmuduo/build/bin/Thread_test，其执行结果可以通过 `top -H -p pid` 来观测，具体结果如下：

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/f86e74f0-ec09-498e-bc40-c7d1bffc2caa.png"/>
</div>
<div class="image-caption" align="center">Thread_test 运行结果</div>
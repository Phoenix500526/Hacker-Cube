---
title: 对 muduo 网络库线程池的思考与实践
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-06 14:51:42
password:
summary:
tags: [muduo网络库, 线程池]
categories: muduo源码剖析
---

#### 文前导读
本文是 muduo 源码剖析系列的第六篇文章，主要探讨了 tmuduo 网络库的线程池模型，其中包含了以下内容：
> * 为什么要重新设计 ThreadPool
> * SyncQueue 的实现
> * ThreadPool 的实现

本文中所涉及的代码来源于我的个人项目：tmuduo。本着“纸上学来终觉浅，绝知此事要躬行”的想法，我将 muduo 网络库重新实现了一遍，并在上面验证了自己的不少猜想。项目的仓库地址为 git@github.com:Phoenix500526/Tmuduo.git, 欢迎各位fork + star，一起加入学习
<!-- more -->

#### tmuduo 线程池的设计
如果你看过我之前的文章，那么很容易发现这篇文章和之前的文章不太一样。之前的文章大部分都是谈论 muduo 相应部分的设计，然后提出我自己的想法并实践，最终将实践结果,也就是 tmuduo 的代码，展示出来。而这篇文章跳过 muduo 网络库设计的部分，直接谈论了 tmuduo 网络库中的 ThreadPool 设计，主要原因有二：
>* muduo 网络库中的线程池依赖于 Thread 的实现，而 tmuduo 中的 Thread 采用和 muduo 不一样的实现方式，在这样子的前提下去修改 ThreadPool 的代码，改出来的东西完全是四不像。
>* muduo 网络库中的 ThreadPool 有点不太符合我的编程美学。muduo 网络库中 ThreadPool 采用 deque 作为任务队列，把任务队列的所有同步工作一起放到了 ThreadPool 中去完成。这种做法让我感到不太舒服，于是我在 tmuduo 网络库中实现了一个 SyncQueue，将同步工作移交给 SyncQueue，而 ThreadPool 只需要使用即可。

基于以上两个理由，我决定脱离 muduo 的 ThreadPool 限制，实现一个自己版本的 ThreadPool，但为了能融入 tmuduo 的整体框架之中，我还是保留了 muduo 的 ThreadPool 测试代码，并让 tmuduo 的 ThreadPool 通过了该测试。我自己对 tmuduo 的线程池部分有一个未实现的想法：那就是看看是否能引入类似于 skynet 网络框架的设计，为线程池中的不同线程分配不同的优先级，并让他们根据优先级来取出不同数量的任务(有点类似于操作系统的优先级队列轮转调度算法)，这样可以更好地利用到 CPU 的资源。不过由于时间和精力的原因，这个想法还没有明确的时间计划，以后有机会再补充吧。

线程池在实现上大概可分为两种，一种是半同步半异步线程池，而另一种则是领导者追随者线程池。tmuduo 的 ThreadPool 属于前者，本质上也就是个生产者消费者模型。大体可分两层：同步服务层和异步服务层。上层会调用线程池的相应接口，将任务添加到同步服务层中，而线程池中的线程则属于异步服务层，它们会在空闲的时候异步地从同步队列中取出任务来执行。

#### 同步队列 SyncQueue 的实现
同步队列的代码主要放在 base/SyncQueue.h 当中
```C++
template <typename T>
class SyncQueue : noncopyable {
 public:
  SyncQueue(int maxsize)
      : mutex_(), full_(), empty_(), maxSize_(maxsize), running_(true) {}
  ~SyncQueue() = default;
  void put(const T& x) { add(x); }
  void put(T&& x) { add(std::forward<T>(x)); }
  void take(T& t) {
    UniqueLock lock(mutex_);
    while (queue_.empty() && running_) {
      empty_.wait(lock);
    }
    if (!running_) return;
    t = queue_.front();
    queue_.pop_front();
    full_.notify_one();
  }
  size_t size() const {
    UniqueLock lock(mutex_);
    return queue_.size();
  }

  void stop() NO_THREAD_SAFETY_ANALYSIS {
    {
      UniqueLock lock(mutex_);
      running_ = false;
    }
    full_.notify_all();
    empty_.notify_all();
  }

 private:
  template <typename F>
  void add(F&& x) {
    UniqueLock lock(mutex_);
    while (queue_.size() >= maxSize_ && running_) {
      full_.wait(lock);
    }
    if (!running_) return;
    queue_.push_back(std::forward<F>(x));
    empty_.notify_one();
  }

 private:
  std::list<T> queue_;
  mutable Mutex mutex_;
  Condition full_ GUARDED_BY(mutex_);   //同步队列已满
  Condition empty_ GUARDED_BY(mutex_);  //同步队列已空
  int maxSize_;
  bool running_;
};
```
SyncQueue 整体的实现也不复杂，如果看了之前的文章：[《clang 的线程安全注解TSA》](https://hacker-cube.com/2020/11/05/clang-%E7%9A%84%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E6%B3%A8%E8%A7%A3TSA/)以及 [《对 muduo 网络库中互斥量与条件变量的思考与实践》](https://hacker-cube.com/2020/11/05/%E5%AF%B9-muduo-%E7%BD%91%E7%BB%9C%E5%BA%93%E4%B8%AD%E4%BA%92%E6%96%A5%E9%87%8F%E4%B8%8E%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F%E7%9A%84%E6%80%9D%E8%80%83%E4%B8%8E%E5%AE%9E%E8%B7%B5/)，理解上应该不会有什么太大的问题。
这里有个地方稍微提一下：出于效率的考虑，running\_ 不需要是 atomic 类型。这主要是因在同步队列当中，只有在初始化队列和停止队列时才会修改到 running\_ 而使用频繁的 `take` 和 `put` 函数会不断访问 running\_ 的值，因此我将 running\_ 设置为普通的 bool 类型，而在 `stop` 函数中采用上锁访问的方式来避免 data race.


#### ThreadPool 的实现
ThreadPool 的具体代码放在 tmuduo/base/ThreadPool.{h, cc} 当中，有需要的可以自行下下来研究。这里出于篇幅问题，只展示了一些关键代码。
```C++
//ThreadPool.h:
class ThreadPool : noncopyable {
 public:
  using Task = std::function<void()>;
  ThreadPool(int maxSize, const std::string& name = std::string("ThreadPool"));
  ~ThreadPool();
  void setThreadInitCallback(const Task& cb) { threadInitCallback_ = cb; }
  void start(int numThreads);
  void stop();
  void run(Task&& f);
  //一些其他的函数
 private:
  void runInThread();
  std::string name;
  Task threadInitCallback_;
  std::vector<std::unique_ptr<Thread>> threads_;
  SyncQueue<Task> queue_;
  std::atomic<bool> running_;
};

//ThreadPool.cc
ThreadPool::ThreadPool(int maxSize, const std::string& name)
    : name_(name), queue_(maxSize), running_(false) {
  assert(maxSize > 0);
}

ThreadPool::~ThreadPool() {
  if (running_) {
    stop();
  }
}

void ThreadPool::start(int numThreads) {
  assert(threads_.empty() && numThreads >= 0);
  running_ = true;
  for (int i = 0; i < numThreads; ++i) {
    threads_.emplace_back(
        new tmuduo::Thread(std::bind(&ThreadPool::runInThread, this),
                           name_ + std::to_string(i + 1)));
  }
  if (0 == numThreads && threadInitCallback_) {
    threadInitCallback_();
  }
}

void ThreadPool::stop() {
  queue_.stop();
  running_ = false;
  for (auto& thr : threads_) {
    thr->join();
  }
}

void ThreadPool::run(Task&& task) { queue_.put(std::forward<Task>(task)); }

void ThreadPool::runInThread() {
  try {
    if (threadInitCallback_) {
      threadInitCallback_();
    }
    while (running_) {
      Task task;
      queue_.take(task);
      if (task) {
        task();
      }
    }
  } catch (const Exception& ex) {
    fprintf(stderr, "exception caught in ThreadPool %s\n", name_.c_str());
    fprintf(stderr, "reasion: %s\n", ex.what());
    fprintf(stderr, "stack trace: %s\n", ex.stackTrace());
    abort();
  } catch (const std::exception& ex) {
    fprintf(stderr, "exception caught in ThreadPool %s\n", name_.c_str());
    fprintf(stderr, "reasion: %s\n", ex.what());
    abort();
  } catch (...) {
    fprintf(stderr, "unknown exception caught in ThreadPool %s\n",
            name_.c_str());
    throw;
  }
}
```
上述代码的实现也比较简单，如果看过之前的文章[《对-muduo-网络库中的线程模型的思考与实践》](https://hacker-cube.com/2020/11/05/%E5%AF%B9-muduo-%E7%BD%91%E7%BB%9C%E5%BA%93%E4%B8%AD%E7%9A%84%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B%E7%9A%84%E6%80%9D%E8%80%83%E4%B8%8E%E5%AE%9E%E8%B7%B5/) 了解 tmuduo 的 Thread 实现，那么只要捋清线程的 `start` 以及 `stop` 基本就能够将这个 ThreadPool 实现出来。

回顾到之前提到的 ThreadPool 的双层结构，对应到代码上则是：

> 上层  ：创建 ThreadPool 的线程
> 同步层：ThreadPool 中的 SyncQueue 
> 异步层：ThreadPool 的 threads\_ 数组

上层通过调用 `run` 函数将任务添加到同步层中，异步层中的线程的 `runInThread` 则会在空闲的时候将任务取出并执行。
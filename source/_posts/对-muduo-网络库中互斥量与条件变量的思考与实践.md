---
title: 对 muduo 网络库中互斥量与条件变量的思考与实践
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-05 20:18:54
password:
summary:
tags: [muduo网络库, 多线程, mutex, condition_variable]
categories: muduo源码剖析
---
#### 文前导读
本文是 muduo 网络库源码剖析系列文章的第三篇文章，主要探讨了 muduo 网络库对互斥量和条件变量的封装，以及我如何使用 C++11 的 mutex 以及 condition_variable 来对其进行重新实现，其中包含了以下内容：
> * 基于 clang 的线程安全注解实现的 Mutex 类
> * 基于 condition_variable 实现的 Condition 类
> * RAII 对象测试的一些 tips

本文中所涉及的代码来源于我的个人项目：tmuduo。本着“纸上学来终觉浅，绝知此事要躬行”的想法，我将 muduo 网络库重新实现了一遍，并在上面验证了自己的不少猜想。项目的仓库地址为 git@github.com:Phoenix500526/Tmuduo.git, 欢迎各位fork + star，一起加入学习
<!-- more -->
#### 基于 clang 的线程安全注解实现的 Mutex 类
在并发编程当中，data race 往往是个非常令人头疼的问题。通常为了避免 data race，我们通常会使用静态的检查工具(如 clang 的 TSA)或者动态的检查工具(如 Valgrind) 等来检查代码。为了能够在 tmuduo 的实现中引入 TSA， 我对标准库的 mutex 进行了一层封装。具体的代码如下：
```C++
class CAPABILITY("mutex") Mutex : noncopyable {
 public:
  Mutex() : mutex_(), holder_(0) {}
  ~Mutex() { assert(holder_ == 0); }

  void lock() ACQUIRE() {
    mutex_.lock();
    assignHolder();
  }

  void unlock() RELEASE() {
    unassignHolder();
    mutex_.unlock();
  }

  bool isLockedByThisThread() const { return holder_ == CurrentThread::tid(); }

  void assertLocked() ASSERT_CAPABILITY(this) {
    assert(isLockedByThisThread());
  }

 private:
  friend class UniqueLock;
  //仅供 UniqueLock 使用，严禁用户调用
  std::mutex& getMutex() { return mutex_; }
  void assignHolder() { holder_ = CurrentThread::tid(); }
  void unassignHolder() { holder_ = 0; }
  std::mutex mutex_;
  pid_t holder_;
};
```
对于上述代码，只要了解 TSA 的相关宏定义以及 `mutex` 的使用，应该不难理解。这里不过多地解释我自己写的代码。在我最初实现 Mutex 时，我并没有保留 `isLockedByThisThread()` 和 `assertLocked()`，而且我使用的是 `thread::id` 作为 `holder_` 的类型。后来，在实现 tmuduo 的过程当中，随着对项目理解的加深，我对 Mutex 做出了两点改变：
* 使用 pid_t 而非 thread::id 来保存 holder_
* 保留了`isLockedByThisThread()`和`assertLocked`这两个函数
其中关于第一个改变的原因，由于涉及到线程方面的一些内容，我会放到下一篇文章中来讨论，而这里主要谈谈`isLockedByThisThread()`和`assertLocked`这两个函数的意义。

通常情况下，如果一个函数既有可能在已加锁的情况下访问，也可能在未加锁的情况下访问，那么出于执行效率的考虑，你应当将其拆分成为两个函数，例如：

```C++
MutexLock mutex;
std::vector<Foo> foos;
void post(const Foo& f){
    MutexLockGuard lock(mutex);
    postWithLockHold(f);    //编译器会自动内联优化，因此不必担心开销
}
//引入这个函数是为了体现代码作者的意图，尽管 push_back 通常可以手动内联
void postWithLockHold(const Foo& f){
    foos.push_back(f);
}
```
对于上述代码，可能会导致出现两个问题（此处的无锁和加锁是指函数调用外部，而非函数内部）：
* 本该使用无锁版本 post， 结果误用了加锁版本 postWithLockHold，导致了死锁【之所以带上了 WithLockHold 这个显眼的后缀就是为了避免误用。但是如果误用了，可以用 gdb 利用函数调用栈来进行调试，如果两个函数先后占有了同一个 mutex 就会引发死锁】
* 本该使用加锁版本 postWithLockHold，结果误用了无锁版本 post，导致了数据的损坏。【因此，MutexLock 提供了 `assertLocked( )`来进行检查】，例如：
```C++
void postWithLockHold(const Foo& f){
    assertLocked();
    foos.push_back(f);
}
```
回到问题本身，在 Mutex 的封装下，`assertLocked` 的实现可以看做是对 TSA 的一点点补充，TSA 是编译期的静态检查，对于运行时产生的错误无能为力。在实现 Mutex 时候，我们有一个基本原则：那就是**尽可能让错误尽早出现**。不管是利用 TSA 让编译器能够在编译期检测出错误，还是实现 `assertLocked` 让代码在运行时一旦产生错误就立即终止程序，都是为了尽量避免将错误留给用户去处理。

当然，由于目前 Mutex 资源需要手动分配与释放， 用起来不太方便。我们可以采用 RAII 的方式对其进行包装，如下：
```C++
class SCOPED_CAPABILITY UniqueLock : noncopyable {
 public:
  explicit UniqueLock(Mutex& mutex) ACQUIRE(mutex)
      : mutex_(mutex), lck_(mutex.getMutex()) {
    mutex_.assignHolder();
  }
  ~UniqueLock() RELEASE() { mutex_.unassignHolder(); }

  std::unique_lock<std::mutex>& getUniqueLock() { return lck_; }

 private:
  Mutex& mutex_;
  std::unique_lock<std::mutex> lck_;
};

#define UniqueLock(x) error "Missing guard object name"
```
上述代码也比较好理解，这里主要讲一下宏`UniqueLock(x)`的意义。对于 RAII 对象而言，通常有两种情况会导致诡异的bug：
* 由于编译器优化，而使得 RAII 对象的生命周期大大延长
* 不小心将 RAII 对象定义为匿名对象，从而导致提前析构
对于第一个问题，我们留到后面将 Mutex_test 的时候来讨论，而宏`UniqueLock(x)`正是为了避免解决第二个问题而存在的。我们来看一个例子：
```C++
void doit(){
    MutexLockGuard(mutex);    //因为遗漏了变量名，结果产生了一个临时对象然后又立马销毁了，结果没能锁住临界区
    //正确写法为 MutexLockGuard lock(mutex);
    ...
}
```
如果定义了宏`UniqueLock(x)`，则上述代码是无法通过编译的。

#### 基于 condition_variable 实现的 Condition 类
在实现了 Mutex 后，我也对 condition_variable 做了一层薄薄的封装。虽然 Condition 本身没有引入 TSA，但标准库中的 condition_variable 需要依赖于 mutex，因此我们也需要对 condition_variable 做了一层封装才能够兼容我们自己实现的 Mutex 类。
```C++
class Condition : noncopyable {
private:
  std::condition_variable m_cond;

public:
  Condition() : m_cond() {}
  ~Condition() = default;
  void notify_all() { m_cond.notify_all(); }
  void notify_one() { m_cond.notify_one(); }
  void wait(UniqueLock& lck) { m_cond.wait(lck.getUniqueLock()); }
  template <class Predicate>
  void wait(UniqueLock& lck, Predicate pred) {
    m_cond.wait(lck.getUniqueLock(), pred);
  }
  template <class Rep, class Period>
  std::cv_status wait_for(UniqueLock& lck,
                          const std::chrono::duration<Rep, Period>& rel_time) {
    return m_cond.wait_for(lck.getUniqueLock(), rel_time);
  }
  template <class Rep, class Period, class Predicate>
  bool wait_for(UniqueLock& lck,
                const std::chrono::duration<Rep, Period>& rel_time,
                Predicate pred) {
    return m_cond.wait_for(lck.getUniqueLock(), rel_time, pred);
  }
};
```
代码整体来说比较简单，这里不过多赘述。

#### Mutex_test 的测试
回到我们之前提到的关于 RAII 对象生命周期的一个问题，那就是要特别注意因为编译器的内联优化而导致 RAII 对象作用域溢出的问题。由于测试代码比较冗长，因此只贴了一小部分代码，如果有需要，可以在 tmuduo/test/Mutex_test.cc 文件中查看完整代码。下面主要说一下测试当中怎么样去注意这个问题：
```C++
int foo() __attribute__((noinline));
int foo() {
  UniqueLock lock(g_mutex);
  if (!g_mutex.isLockedByThisThread()) {
    printf("FAIL\n");
    return -1;
  }
  ++g_count;
  return 0;
}
int main(void){
  ... //do something
  foo();
  if (g_count != 1) {
    printf("foo calls twice\n");
    abort();
  }
  ... //do something
}
```
在上述代码中，利用到了 `__attribute__((noinline))` 来显式指明了 `foo` 是非内联函数，因此编译器不会对其进行内联优化。如果编译器对 `foo` 执行了内联优化，则 lock 的作用域将会从 `foo` 的局部作用域扩张到 `mian` 的作用域当中，这就很容易导致一些诡异的 bug 的产生
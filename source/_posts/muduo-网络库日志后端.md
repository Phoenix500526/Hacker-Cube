---
title: muduo 网络库日志后端
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-03 22:47:50
password:
summary:
tags: [muduo网络库, 源码剖析, 日志系统, 多线程, C++]
categories: muduo源码剖析
---
#### 前言
muduo 网络库源码剖析系列的第二篇文章，主要着眼于 muduo 网络库中的日志系统后端的设计与实现
为了保证自己对 muduo 的代码有较为深入的理解，我自己写了一个 tmuduo 网络库，用来验证自己对 muduo 源码上的一些想法。
仓库地址为：git@github.com:Phoenix500526/Tmuduo.git，欢迎 fork、start 以及 follow，一起学习。

<!-- more -->
#### AsyncLogging.{h，cc} 的实现
AsyncLogging 是整个日志系统的后端线程，其中 `append()` 函数作为前端 Logger 类的输出回调函数。当用户使用如`LOG_DEBUG << "Hello world"`语句时，会将日志信息封装为 Buffer
<!-- more -->
```C++
// AsyncLogging.h 文件(节选)
class AsyncLogging : noncopyable{
 public:
  AsyncLogging(const string& basename,
               off_t rollSize,
               int flushInterval = 3);
  ~AsyncLogging() {
    if (running_) {
      stop();
    }
  }

  void append(const char* logline, int len);

  void start() {
    running_ = true;
    thread_.start();
    latch_.wait();
  }

  void stop() NO_THREAD_SAFETY_ANALYSIS {
    running_ = false;
    cond_.notify();
    thread_.join();
  }

 private:

  void threadFunc();
  typedef muduo::detail::FixedBuffer<muduo::detail::kLargeBuffer> Buffer;
  typedef std::vector<std::unique_ptr<Buffer>> BufferVector;
  typedef BufferVector::value_type BufferPtr;

  const int flushInterval_;
  std::atomic<bool> running_;
  const string basename_;
  const off_t rollSize_;
  muduo::Thread thread_;
  muduo::CountDownLatch latch_;
  muduo::MutexLock mutex_;
  muduo::Condition cond_ GUARDED_BY(mutex_);
  BufferPtr currentBuffer_ GUARDED_BY(mutex_);
  BufferPtr nextBuffer_ GUARDED_BY(mutex_);
  BufferVector buffers_ GUARDED_BY(mutex_);
};
```

```C++
// AsyncLogging.cc
AsyncLogging::AsyncLogging(const string& basename, off_t rollSize, int flushInterval)
  : flushInterval_(flushInterval),
    running_(false),
    basename_(basename),
    rollSize_(rollSize),
    thread_(std::bind(&AsyncLogging::threadFunc, this), "Logging"),
    latch_(1),
    mutex_(),
    cond_(mutex_),
    currentBuffer_(new Buffer),
    nextBuffer_(new Buffer),
    buffers_()
{
  currentBuffer_->bzero();
  nextBuffer_->bzero();
  buffers_.reserve(16);
}

void AsyncLogging::append(const char* logline, int len) {
  muduo::MutexLockGuard lock(mutex_);
  if (currentBuffer_->avail() > len) {
    currentBuffer_->append(logline, len);
  } else {
    buffers_.push_back(std::move(currentBuffer_));
    if (nextBuffer_) {
      currentBuffer_ = std::move(nextBuffer_);
    } else {
      currentBuffer_.reset(new Buffer); // Rarely happens
    }
    currentBuffer_->append(logline, len);
    cond_.notify();
  }
}

void AsyncLogging::threadFunc() {
  assert(running_ == true);
  latch_.countDown();
  LogFile output(basename_, rollSize_, false);
  BufferPtr newBuffer1(new Buffer);
  BufferPtr newBuffer2(new Buffer);
  newBuffer1->bzero();
  newBuffer2->bzero();
  BufferVector buffersToWrite;
  buffersToWrite.reserve(16);
  while (running_) {
    assert(newBuffer1 && newBuffer1->length() == 0);
    assert(newBuffer2 && newBuffer2->length() == 0);
    assert(buffersToWrite.empty());
    {
      muduo::MutexLockGuard lock(mutex_);
      if (buffers_.empty()) { // unusual usage!
        cond_.waitForSeconds(flushInterval_);
      }
      buffers_.push_back(std::move(currentBuffer_));
      currentBuffer_ = std::move(newBuffer1);
      //采用 swap 的方式，将数据移至局部对象 buffersToWrite 中，这样在处理的时候不会产生竞态问题，而且缩短了临界区。这同样也是为了避免日志前端的等待
      buffersToWrite.swap(buffers_);
      if (!nextBuffer_) {
        nextBuffer_ = std::move(newBuffer2);
      }
    }

    assert(!buffersToWrite.empty());

    if (buffersToWrite.size() > 25) {
      char buf[256];
      snprintf(buf, sizeof buf, "Dropped log messages at %s, %zd larger buffers\n",
               Timestamp::now().toFormattedString().c_str(),
               buffersToWrite.size()-2);
      fputs(buf, stderr);
      output.append(buf, static_cast<int>(strlen(buf)));
      buffersToWrite.erase(buffersToWrite.begin()+2, buffersToWrite.end());
    }

    for (const auto& buffer : buffersToWrite) {
      // FIXME: use unbuffered stdio FILE ? or use ::writev ?
      output.append(buffer->data(), buffer->length());
    }

    if (buffersToWrite.size() > 2) {
      // drop non-bzero-ed buffers, avoid trashing
      buffersToWrite.resize(2);
    }

    if (!newBuffer1) {
      assert(!buffersToWrite.empty());
      newBuffer1 = std::move(buffersToWrite.back());
      buffersToWrite.pop_back();
      newBuffer1->reset();
    }

    if (!newBuffer2) {
      assert(!buffersToWrite.empty());
      newBuffer2 = std::move(buffersToWrite.back());
      buffersToWrite.pop_back();
      newBuffer2->reset();
    }

    buffersToWrite.clear();
    output.flush();
  }
  output.flush();
}
```

整个 AsyncLogging 类的设计采用了双缓冲的方式实现，主要的思路是

> 采用两个缓冲：A 和 B。前端负责往缓冲 A 中写数据，而后端负责将缓冲 B 的数据写入文件。当 A 写完后，则交换 A 和 B，然后让后端将缓冲 A 写入文件，而前端则继续往 B 中填充数据，并不断重复这个过程。

这样的做法，好处是在新建日志的时候不必等待磁盘文件操作，另外批处理的方式也能避免后端线程被频繁唤醒。实际实现中，为了进一步减少日志前端的等待，使用了两个工作缓冲区 + 两个备用缓冲区，共四个缓冲区。

另外，日志系统的后端实现中仅使用了缓冲区，而没有使用消息队列，主要还是出于功能和性能的考虑。从功能上讲，后端线程需要负责间隔刷新，当时间间隔 (默认是 3 秒) 抵达时，不论是否有数据抵达，后端线程都需要刷新缓冲区以避免程序崩溃而导致日志丢失。其次，从性能上讲，只有单个消费者，因此不使用阻塞队列，就可以避免每次有日志信息抵达就要去 notify 后端线程一次。

对于 AsyncLogging 的实现而言并不复杂，真正有难度的点在于对日志刷新情况的把握，主要可以分为：

> 正常情况：
>
> * 前端未写满缓冲区就发生了超时，此时程序应当有何种行为？
> * 前端在发生超时以前就写满了缓冲区，此时程序应当有何种行为？
> * 前端产生了过多的日志导致写超出的情况，此时程序应当有何种行为？
> * 后端因写入速度太慢，导致写超出的情况，，此时程序应当有何种行为？
>
> 异常情况：出现了死锁或者死循环，导致后端日志消息堆积，此时程序应当有何种行为？

对于上述的情况，陈硕在他的书 《Linux 多线程服务端编程  —— 使用 muduo C++ 网络库》中有详细的描述，并配有时序图，在此不赘述。

#### LogFile.{h, cc} 的源码及实现

```C++
//LogFile.h
class LogFile : noncopyable {
 public:
  LogFile(const string& basename,
          off_t rollSize,
          bool threadSafe = true,
          int flushInterval = 3,
          int checkEveryN = 1024);
  ~LogFile();

  void append(const char* logline, int len);
  void flush();
  bool rollFile();

 private:
  void append_unlocked(const char* logline, int len);
  static string getLogFileName(const string& basename, time_t* now);
  const string basename_;
  const off_t rollSize_;
  const int flushInterval_;
  const int checkEveryN_;
  int count_;
  std::unique_ptr<MutexLock> mutex_;
  time_t startOfPeriod_;
  time_t lastRoll_;
  time_t lastFlush_;
  std::unique_ptr<FileUtil::AppendFile> file_;
  //代表一天的秒数
  const static int kRollPerSeconds_ = 60*60*24;
};
```

```C++
// LogFile.cc
LogFile::LogFile(const string& basename, off_t rollSize, bool threadSafe, int flushInterval, int checkEveryN)
  : basename_(basename),
    rollSize_(rollSize),
    flushInterval_(flushInterval),
    checkEveryN_(checkEveryN),
    count_(0),
    mutex_(threadSafe ? new MutexLock : NULL),
    startOfPeriod_(0),
    lastRoll_(0),
    lastFlush_(0)
{
  assert(basename.find('/') == string::npos);
  rollFile();
}

LogFile::~LogFile() = default;

void LogFile::append(const char* logline, int len) {
  if (mutex_) {
    MutexLockGuard lock(*mutex_);
    append_unlocked(logline, len);
  } else {
    append_unlocked(logline, len);
  }
}

void LogFile::flush() {
  if (mutex_) {
    MutexLockGuard lock(*mutex_);
    file_->flush();
  } else {
    file_->flush();
  }
}

void LogFile::append_unlocked(const char* logline, int len) {
  file_->append(logline, len);

  if (file_->writtenBytes() > rollSize_) {
    rollFile();
  } else {
    ++count_;
    if (count_ >= checkEveryN_) {
      count_ = 0;
      time_t now = ::time(NULL);
      time_t thisPeriod_ = now / kRollPerSeconds_ * kRollPerSeconds_;
      if (thisPeriod_ != startOfPeriod_) {
        rollFile();
      }
      else if (now - lastFlush_ > flushInterval_) {
        lastFlush_ = now;
        file_->flush();
      }
    }
  }
}

bool LogFile::rollFile() {
  time_t now = 0;
  string filename = getLogFileName(basename_, &now);
  time_t start = now / kRollPerSeconds_ * kRollPerSeconds_;

  if (now > lastRoll_) {
    lastRoll_ = now;
    lastFlush_ = now;
    startOfPeriod_ = start;
    file_.reset(new FileUtil::AppendFile(filename));
    return true;
  }
  return false;
}

string LogFile::getLogFileName(const string& basename, time_t* now){
  string filename;
  filename.reserve(basename.size() + 64);
  filename = basename;

  char timebuf[32];
  struct tm tm;
  *now = time(NULL);
  gmtime_r(now, &tm); // FIXME: localtime_r ?
  strftime(timebuf, sizeof timebuf, ".%Y%m%d-%H%M%S.", &tm);
  filename += timebuf;

  filename += ProcessInfo::hostname();

  char pidbuf[32];
  snprintf(pidbuf, sizeof pidbuf, ".%d", ProcessInfo::pid());
  filename += pidbuf;

  filename += ".log";

  return filename;
}
```

对于 LogFile 而言，其主要的作用是实现文件滚动。文件滚动的触发条件有两个：

* 当文件的大小超过一定值后
* 每隔一天滚动一次

当满足滚动条件时，新建的日志文件名应满足：程序名.时间.主机名.进程id.log(例如：logtest.20200713-171322.hostname.1234.log)。

其中主要的函数有两个 `rollFile` 和 `append_unlocked`。主要说说 `append_unlocked()`函数。

对于滚动的第一个条件而言，每次写入日志信息时都会用 记录写入信息的大小。通过 `writtenBytes() `就可以知道当前往文件中写入了多少字节的数据。

对于第二个条件，`append_unlock()` 会记录所插入的日志条数，每当插入日志的记录数超过 checkEveryN 时就检测当前时间，并从中取出天数和 startOfPeriod_ 进行比较，如果不在同一天，则执行 `rollFile()`



#### 一些其他的问题

为什么 AsyncLogging 中的部分函数需要使用到锁，而 LogFile 当中要提供无锁版本？

这主要还是因为 AsyncLogging 和许多日志前端打交道，是一对多的问题，因此必须考虑线程安全性。而 LogFile 只和 AsyncLogging 打交道，是一对一的情况，主要以无锁版本为主。实际上在 muduo 网络库中的 example 以及测试代码中，AsyncLogging  使用的都是 LogFile 的 `append_unlocked()`函数
---
title: muduo 网络库日志前端
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-03 20:52:36
password:
summary:
tags: [muduo网络库, 源码剖析, 日志系统, C++]
categories: muduo源码剖析
---

#### muduo 网络库日志模块简介

muduo 网络库中提供了一个高效的异步日志模块，其本质是一个多生产者单消费者的模型，可以分为前端(frontend) 和后端(backend)。

* 前端：前端采用流的输出格式，并提供了相关级别的宏（LOG_DEBUG, LOG_INFO, etc）供用户使用，代码位于 Logging.{h, cc} 中。出于安全性以及性能的考虑，封装了一个简单的日志流 LogStream，代码位于 LogStream.{h, cc} 中。
* 后端：后端采用了双缓冲的方式(为了进一步提高性能，减少分配内存空间的开销，后端实际上准备了两个工作缓冲 + 两个后备缓冲，所以一共有四个缓冲区)，代码位于 AsyncLogging.{h, cc} 中。后端封装了 LogFile 文件来实现磁盘 IO，同时支持了间隔刷新以及文件滚动等功能，代码位于 LogFile.{h, cc} 中

日志模块的启动流程为：

1. 使用一个全局的 AsyncLogging 指针，指向一个初始化好 AsyncLogging 对象
2. 调用 Logger 类的静态成员函数 `setOutput()`，将 AsyncLogging 对象的 `append()` 函数注册到 Logger 类当中
3. 用户使用对应的日志宏进行日志的输出



#### 前端源码及亮点

前端 Logger 类的设计与实现【出于版面节省需要，对代码格式进行了改动。原版代码采用了对 grep/diff 友好的代码格式】：

```C++
// Logging.h
class Logger {
 public:
  //日志级别的数量
  enum LogLevel{TRACE, DEBUG, INFO, WARN, ERROR, FATAL, NUM_LOG_LEVELS,	};
  // 将文件名和长度封装成为内部类 SourceFile，清晰了语义同时，简化了构造函数的实现。
  class SourceFile {
   public:
    template<int N>
    SourceFile(const char (&arr)[N]): data_(arr), size_(N-1) {
      const char* slash = strrchr(data_, '/'); // builtin function
      if (slash) {
        data_ = slash + 1;
        size_ -= static_cast<int>(data_ - arr);
      }
    }
    explicit SourceFile(const char* filename): data_(filename) {
      const char* slash = strrchr(filename, '/');
      if (slash) {
        data_ = slash + 1;
      }
      size_ = static_cast<int>(strlen(data_));
    }
    const char* data_;
    int size_;
  };

  Logger(SourceFile file, int line);
  Logger(SourceFile file, int line, LogLevel level);
  Logger(SourceFile file, int line, LogLevel level, const char* func);
  Logger(SourceFile file, int line, bool toAbort);
  ~Logger();
  LogStream& stream() { return impl_.stream_; }
  static LogLevel logLevel();
  static void setLogLevel(LogLevel level);

  typedef void (*OutputFunc)(const char* msg, int len);
  typedef void (*FlushFunc)();
  static void setOutput(OutputFunc);
  static void setFlush(FlushFunc);
  static void setTimeZone(const TimeZone& tz);

 private:
    
class Impl {
 public:
  typedef Logger::LogLevel LogLevel;
  Impl(LogLevel level, int old_errno, const SourceFile& file, int line);
  void formatTime();
  void finish();

  Timestamp time_;
  LogStream stream_;
  LogLevel level_;
  int line_;
  SourceFile basename_;
};
  Impl impl_;
};

extern Logger::LogLevel g_logLevel;

inline Logger::LogLevel Logger::logLevel() {
  return g_logLevel;
}

#define LOG_TRACE if (muduo::Logger::logLevel() <= muduo::Logger::TRACE) \
  muduo::Logger(__FILE__, __LINE__, muduo::Logger::TRACE, __func__).stream()
#define LOG_DEBUG if (muduo::Logger::logLevel() <= muduo::Logger::DEBUG) \
  muduo::Logger(__FILE__, __LINE__, muduo::Logger::DEBUG, __func__).stream()
#define LOG_INFO if (muduo::Logger::logLevel() <= muduo::Logger::INFO) \
  muduo::Logger(__FILE__, __LINE__).stream()
#define LOG_WARN muduo::Logger(__FILE__, __LINE__, muduo::Logger::WARN).stream()
#define LOG_ERROR muduo::Logger(__FILE__, __LINE__, muduo::Logger::ERROR).stream()
#define LOG_FATAL muduo::Logger(__FILE__, __LINE__, muduo::Logger::FATAL).stream()
#define LOG_SYSERR muduo::Logger(__FILE__, __LINE__, false).stream()
#define LOG_SYSFATAL muduo::Logger(__FILE__, __LINE__, true).stream()
}  
```

```C++
// Logging.cc
__thread char t_errnobuf[512];
__thread char t_time[64];
__thread time_t t_lastSecond;

Logger::LogLevel initLogLevel() {
  if (::getenv("MUDUO_LOG_TRACE"))
    return Logger::TRACE;
  else if (::getenv("MUDUO_LOG_DEBUG"))
    return Logger::DEBUG;
  else
    return Logger::INFO;
}

Logger::LogLevel g_logLevel = initLogLevel();

const char* LogLevelName[Logger::NUM_LOG_LEVELS] ={"TRACE ", "DEBUG ", "INFO  ", "WARN  ", "ERROR ", "FATAL ",};

// helper class for known string length at compile time
class T {
 public:
  T(const char* str, unsigned len)
    :str_(str),
     len_(len)
  {
    assert(strlen(str) == len_);
  }

  const char* str_;
  const unsigned len_;
};

inline LogStream& operator<<(LogStream& s, T v) {
  s.append(v.str_, v.len_);
  return s;
}

inline LogStream& operator<<(LogStream& s, const Logger::SourceFile& v) {
  s.append(v.data_, v.size_);
  return s;
}

void defaultOutput(const char* msg, int len) {
  size_t n = fwrite(msg, 1, len, stdout);
  //FIXME check n
  (void)n;
}

void defaultFlush() {
  fflush(stdout);
}

Logger::OutputFunc g_output = defaultOutput;
Logger::FlushFunc g_flush = defaultFlush;
TimeZone g_logTimeZone;

Logger::Impl::Impl(LogLevel level, int savedErrno, const SourceFile& file, int line)
  : time_(Timestamp::now()), stream_(), level_(level), line_(line), basename_(file)
{
  formatTime();
  CurrentThread::tid();
  stream_ << T(CurrentThread::tidString(), CurrentThread::tidStringLength());
  stream_ << T(LogLevelName[level], 6);
  if (savedErrno != 0){
    //strerror_tl 对 strerror_r 进行了一层封装，将 saveErrno 对应的错误信息保存到 t_errnobuf 中
    stream_ << strerror_tl(savedErrno) << " (errno=" << savedErrno << ") ";
  }
}

void Logger::Impl::formatTime() {
  int64_t microSecondsSinceEpoch = time_.microSecondsSinceEpoch();
  time_t seconds = static_cast<time_t>(microSecondsSinceEpoch / Timestamp::kMicroSecondsPerSecond);
  int microseconds = static_cast<int>(microSecondsSinceEpoch % Timestamp::kMicroSecondsPerSecond);
  if (seconds != t_lastSecond) {
    t_lastSecond = seconds;
    struct tm tm_time;
    if (g_logTimeZone.valid()) {
      tm_time = g_logTimeZone.toLocalTime(seconds);
    }
    else {
      ::gmtime_r(&seconds, &tm_time); // FIXME TimeZone::fromUtcTime
    }

    int len = snprintf(t_time, sizeof(t_time), "%4d%02d%02d %02d:%02d:%02d",
        tm_time.tm_year + 1900, tm_time.tm_mon + 1, tm_time.tm_mday,
        tm_time.tm_hour, tm_time.tm_min, tm_time.tm_sec);
    assert(len == 17); (void)len;
  }

  if (g_logTimeZone.valid()) {
    Fmt us(".%06d ", microseconds);
    assert(us.length() == 8);
    stream_ << T(t_time, 17) << T(us.data(), 8);
  }
  else {
    Fmt us(".%06dZ ", microseconds);
    assert(us.length() == 9);
    stream_ << T(t_time, 17) << T(us.data(), 9);
  }
}

void Logger::Impl::finish() {
  stream_ << " - " << basename_ << ':' << line_ << '\n';
}

Logger::Logger(SourceFile file, int line):impl_(INFO, 0, file, line){}

Logger::Logger(SourceFile file, int line, LogLevel level, const char* func)
  : impl_(level, 0, file, line){
  impl_.stream_ << func << ' ';
}

Logger::Logger(SourceFile file, int line, LogLevel level):impl_(level, 0, file, line){}

Logger::Logger(SourceFile file, int line, bool toAbort):impl_(toAbort?FATAL:ERROR, errno, file, line){}

Logger::~Logger(){
  impl_.finish();
  const LogStream::Buffer& buf(stream().buffer());
  g_output(buf.data(), buf.length());
  if (impl_.level_ == FATAL) {
    g_flush();
    abort();
  }
}
```

**设计亮点1：利用了 `__thread` 关键字**

`__thread` 是 GCC 内置的线程局部存储设施，它的实现非常高效，其存储效率可以媲美全局变量。使用`__thread`变量有以下限制：

* 只能修饰 POD 类型，不能修饰 class 类型(因为 `__thread` 不会自动调用构造和析构函数)
* 只能使用编译期常量进行初始化
* `__thread` 只能修饰全局变量和静态变量，但是不能修饰局部变量或者 class 的普通成员变量

`__thread` 变量在每个线程中都有一份独立实体，各个线程中的变量值互不干扰。利用这个特点，`__thread`关键字常常被用来修饰那些 “值可能会变，带有全局性，但是又不值得用全局锁波保护的变量”。

Logging.cc 文件中利用 `__thread` 修饰的变量有 `t_errnobuf`，`t_time` 和 `t_lastSecond`。其中 `t_errnobuf` 主要供 `strerror_tl` 函数使用

```C++
const char* strerror_tl(int savedErrno) {
  return strerror_r(savedErrno, t_errnobuf, sizeof t_errnobuf);
}
```

`strerror_tl` 函数能够简化 `strerror_r` 的使用，同时也保证 `strerror_tl` 的线程安全性。

`t_time` 保留了具体的时间， `t_lastSecond` 保存了上次格式化时间的秒数。只有当两次格式化时间对象的间隔相差超过 1 秒时，才会重新获取当前时间。如果时间间隔低于 1 秒，则直接读取 t_time 中的时间，并更新微秒数。

**设计亮点2：利用局部匿名对象**

如前面说到的，日志模块的前端提供给多个用户使用，因此需要考虑竞态问题，例如同时存在两个并发线程 A 和 B，如何保证在执行下述代码时不会出现串话：

```C++
//线程 A
LOG_INFO << "this is thread A, calling  " << __func__ << ", result is " <<  func();
//线程 B
LOG_INFO << "this is thread B, calling  " << __func__ << ", result is " <<  func();
```

正确结果：

> this is thread A, calling func, result is 10
>
> this is thread B, calling func, result is 5

发生了竞态:

> this is thread A, this is thread B, calling func, reslut is 10
>
> calling func, result is 5

muduo 中采用匿名对象来解决这个问题：所有的 LOG_* 宏都对应着一个构造函数，当执行 `LOG_INFO << "this is thread A, calling  " << __func__ << ", result is " <<  func();"`时，会临时产生一个匿名对象。由于这个对象是局部的，所以在不同的线程之间是透明的，自然也就避免了竞态的问题。同时，匿名对象的生命期只在当前语句当中，因此一旦当前语句执行结束，就会调用对应的析构函数，将日志信息放入缓冲区中并发送给后端进行处理。换句话讲，使用日志宏得到的 Logger 对象都是一次性对象，用完就扔，需要了再创建。

LogStream 类的源码实现

```C++
// LogStream.h
template<int SIZE>
class FixedBuffer : noncopyable {
 public:
  FixedBuffer() : cur_(data_) {
    setCookie(cookieStart);
  }
  ~FixedBuffer() {
    setCookie(cookieEnd);
  }

  void append(const char* /*restrict*/ buf, size_t len) {
    // FIXME: append partially
    if (implicit_cast<size_t>(avail()) > len) {
      memcpy(cur_, buf, len);
      cur_ += len;
    }
  }

  const char* data() const { return data_; }
  int length() const { return static_cast<int>(cur_ - data_); }

  // write to data_ directly
  char* current() { return cur_; }
  int avail() const { return static_cast<int>(end() - cur_); }
  void add(size_t len) { cur_ += len; }

  void reset() { cur_ = data_; }
  void bzero() { memZero(data_, sizeof data_); }

  // for used by GDB
  const char* debugString();
  void setCookie(void (*cookie)()) { cookie_ = cookie; }
  // for used by unit test
  string toString() const { return string(data_, length()); }
  StringPiece toStringPiece() const { return StringPiece(data_, length()); }

 private:
  const char* end() const { return data_ + sizeof data_; }
  // Must be outline function for cookies.
  static void cookieStart();
  static void cookieEnd();
  void (*cookie_)();
  char data_[SIZE];
  char* cur_;
};
class LogStream : noncopyable {
  typedef LogStream self;
 public:
  typedef detail::FixedBuffer<detail::kSmallBuffer> Buffer;
  // 重载不同类型的 << 运算符,出于篇幅考虑，删去了函数声明
  ...
  void append(const char* data, int len) { buffer_.append(data, len); }
  const Buffer& buffer() const { return buffer_; }
  void resetBuffer() { buffer_.reset(); }

 private:
  void staticCheck();
  template<typename T>
  void formatInteger(T);
  Buffer buffer_;
  static const int kMaxNumericSize = 32;
};

class Fmt {
 public:
  template<typename T>
  Fmt(const char* fmt, T val);

  const char* data() const { return buf_; }
  int length() const { return length_; }

 private:
  char buf_[32];
  int length_;
};
```

```C++
//LogStream.cc (节选)
template<int SIZE>
void FixedBuffer<SIZE>::cookieStart() {}

template<int SIZE>
void FixedBuffer<SIZE>::cookieEnd() {}

template<typename T>
Fmt::Fmt(const char* fmt, T val){
  static_assert(std::is_arithmetic<T>::value == true, "Must be arithmetic type");
  length_ = snprintf(buf_, sizeof buf_, fmt, val);
  assert(static_cast<size_t>(length_) < sizeof buf_);
}

// Explicit instantiations
template Fmt::Fmt(const char* fmt, char);
template Fmt::Fmt(const char* fmt, short);
template Fmt::Fmt(const char* fmt, unsigned short);
template Fmt::Fmt(const char* fmt, int);
template Fmt::Fmt(const char* fmt, unsigned int);
template Fmt::Fmt(const char* fmt, long);
template Fmt::Fmt(const char* fmt, unsigned long);
template Fmt::Fmt(const char* fmt, long long);
template Fmt::Fmt(const char* fmt, unsigned long long);
template Fmt::Fmt(const char* fmt, float);
template Fmt::Fmt(const char* fmt, double);
```

**设计亮点1：如何应对程序崩溃而导致的日志丢失**

往文件中写日志的一个常见问题是：万一程序崩溃，那么最后若干条日志往往就会丢失。为了解决这个问题，muduo 网络库采用了两个策略来解决：

* 间隔刷新：每隔一个间隔时间就将内存中的日志信息 flush 到日志文件中
* 设置 Cookie：所谓的 Cookie 实际上是一个函数指针 `void (*cookie_)()` ，主要起到了一个标志物的作用。每一个 FixedBuffer 对象在生命周期的开始和结束，都会被打上对应的 Cookie(`cookieStart` 和 `cookieEnd`)。当程序崩溃时，我们可以借由这一对 cookie，利用 gdb 在 coredump 文件当中找到遗留在内存中尚未输入到文件当中的日志信息。

**设计亮点2：流的格式化问题**

muduo 的日志前端采用了流式风格进行打印，其好处有两个：

* 使用更加方便：使用时无需记忆各种各样的格式化控制符
* 表达更加灵活：流式风格可以很方便地支持链式输出，如 `LOG_INFO << “Hello” << " " << "World"；`

而局限性也是明显的：即流式风格的格式化操作非常麻烦。对于这个问题，可以通过 "增加一层间接性" 来实现。通过定义一个 Fmt 的模板类，并重载相应的 << 操作符即可实现。



#### 一些其他的有趣的内容
1.**implict_cast 与 down_cast**
  implicit_cast 与 down_cast 都是函数模板，其实现如下：
  ```C++
   template<typename To, typename From>
   inline To implicit_cast(From const &f) { return f; }
   
   // use like this: down_cast<T*>(foo);
   // so we only accept pointers
   template<typename To, typename From>     
   inline To down_cast(From* f) {                    
     // Ensures that To is a sub-type of From *.  This test is here only
     // for compile-time type checking, and has no overhead in an
     // optimized build at run-time, as it will be optimized away
     // completely.
     if (false) {
       implicit_cast<From*, To>(0);
     }
   
   #if !defined(NDEBUG) && !defined(GOOGLE_PROTOBUF_NO_RTTI)
     assert(f == NULL || dynamic_cast<To>(f) != NULL);  // RTTI: debug mode only!
   #endif
     return static_cast<To>(f);
   }
   
   // 智能指针 shared_ptr 的向下转型
   template<typename T>
   inline T* get_pointer(const std::shared_ptr<T>& ptr)
   {
     return ptr.get();
   }
   template<typename T>
   inline T* get_pointer(const std::unique_ptr<T>& ptr)
   {
     return ptr.get();
   }
   
   // Adapted from google-protobuf stubs/common.h
   // see License in muduo/base/Types.h
   template<typename To, typename From>
   inline ::std::shared_ptr<To> down_pointer_cast(const ::std::shared_ptr<From>& f) {
     if (false)
     {
       implicit_cast<From*, To*>(0);
     }
   
   #ifndef NDEBUG
     assert(f == NULL || dynamic_cast<To*>(get_pointer(f)) != NULL);
   #endif
     return ::std::static_pointer_cast<To>(f);
   }
    ```
    先来说说 `implicit_cast`，这个函数模板主要的功能是用来代替 static_cast 来实现类继承体系中的向上转型(up-cast)。**static_cast 在类的继承体系之间的类型转换时并不进行类型检查**，这会导致一些问题，例如：
   ```C++
   class Top{};
   class MiddleA : public Top{};
   class MiddleB : public Top{};
   class Bottom : public MiddleA, public MiddleB{};
   void func(MiddleA const& A){
       cout << "A" << endl;
   }
   void func(MiddleB const& B){
       cout << "B" << endl;
   }
   int main(){
       Bottom bot;
       func(static_cast<MiddleA const&>(bot));	//输出 A
       func(static_cast<MiddleB const&>(bot)); //输出 B
   }
   ```
   但是一旦有人在修改代码时，不慎将 bot 的类型改成 Top 类型，那么以下代码依然可以通过编译，但在运行时可能会崩溃（一个 Bottom 对象可以是 MiddleA 类型，也可以是 MiddleB 类型，但一个 Top 对象既不是 MiddleA 类型，也不是 MiddleB 类型）。
   ```C++
   ...
   int main(){
      	Top bot;
       func(static_cast<MiddleA const&>(bot));	//输出 A
       func(static_cast<MiddleB const&>(bot)); //输出 B
   }
   ```

   而当使用 `implicit_cast` 代替 `static_cast` 时，上述代码会在编译期产生错误。

   再来说说 `down_cast` ，这个函数模板的功能是用来代替 `dynamic_cast`进行类继承体系中的向下转型。既然 C++ 标准中提供了 `dynamic_cast` 又为何要自己实现 `down_cast`？

   首先在 《Google C++ 代码规范》中有如下说明：

   > 除单元测试外，不要使用 dynamic_cast，如果你需要在运行时确定类型信息，**说明设计有缺陷。**

   从其实现上可以看出，只有在 debug 模式下才会使用 `dynamic_cast` 进行向下转型，而在正式版本当中，则用 `static_cast` 来代替。这主要出于两点考虑：

   * dynamic_cast 基于 RTTI 实现，其性能开销比较大。事实上，当你能够确定 From 和 To 之间的关系时，完全可以使用 static_cast 来代替 dynamic_cast，而且 static_cast 在继承体系间的转换过程并不执行类型检查，能最大限度地提升性能。
   * dynamic_cast 可能会抛出异常，而 google 的 C++ 代码规范中也指明不要使用异常。

   另外，在其实现上，有一小段代码引起了我的注意：

   ```C++
   if (false) {
       implicit_cast<From*, To>(0);
   }
   ```

   这段代码巧妙地应用了 implicit_cast 来让编译器进行反向的类型检查，而 if(false) 这样的条件语句最终又会被编译器所优化，对性能不会产生任何的影响。

2.**Matthew Wilson 算法**
   在 LogStream.cc 文件中提供了一个 `convert(char buf[], T value)` 函数，用于作为 `itoa()`的替代。其实现如下：
   ```C++
   const char digits[] = "9876543210123456789";
   const char* zero = digits + 9;
   static_assert(sizeof(digits) == 20, "wrong number of digits");
   const char digitsHex[] = "0123456789ABCDEF";
   static_assert(sizeof digitsHex == 17, "wrong number of digitsHex");
   // Efficient Integer to String Conversions, by Matthew Wilson.
   template<typename T>
   size_t convert(char buf[], T value) {
     T i = value;
     char* p = buf;
     do {
       int lsd = static_cast<int>(i % 10);
       i /= 10;
       *p++ = zero[lsd];
     } while (i != 0);
   
     if (value < 0) {
       *p++ = '-';
     }
     *p = '\0';
     std::reverse(buf, p);
     return p - buf;
   }
   ```
   
   在这段代码中，比较巧妙的是采用了一个对称的 digits 数组来解决负数边界转换的问题。这里有一点需要指出，在 C99 标准之前，C 语言对负数的取余操作是 implementation-defined 的。但是在 C99 标准中规定了对负数取余时，商是向零取整。而 C++11 也采用了类似 C99 的表述，因此上述代码行为是可预期的。



#### References
1. 《Linux 多线程服务端编程 —— 使用 muduo C++ 网络库》 —— 陈硕
2.  [c++小技巧(三)更好的类型转换implicit_cast和down_cast](https://blog.csdn.net/xiaoC_fantasy/article/details/79570788)
3.  [Google Style Guide](https://github.com/google/styleguide)

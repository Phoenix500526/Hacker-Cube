---
title: 对 muduo 网络库异常对象的实践与思考
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-05 23:14:42
password:
summary:
tags: [muduo网络库, Exception, ABI, C++]
categories: muduo源码剖析
---

#### 文前导读
本文是 muduo 源码剖析系列文章的第五篇文章，主要探讨了如何设计异常类，使其携带尽可能多的有效信息。
> * 异常到底要不要用 —— Google 到底怎么看异常
> * C++ 的 mangle 与 demangle 过程
> * 如何让异常携带友好的线程函数栈调用信息

本文中所涉及的代码来源于我的个人项目：tmuduo。本着“纸上学来终觉浅，绝知此事要躬行”的想法，我将 muduo 网络库重新实现了一遍，并在上面验证了自己的不少猜想。项目的仓库地址为 git@github.com:Phoenix500526/Tmuduo.git, 欢迎各位fork + star，一起加入学习
<!-- more -->

在前一篇文章《对 muduo 网络库中的线程模型的思考与实践》中，在 Thread 的 `runInThread` 函数中涉及到了对异常的一些处理，而这篇文章的目的便是探讨自己在muduo网络库中所学到的异常实现的一点经验

#### 异常到底要不要用 —— Google 到底怎么看异常
在我最开始重新用自己的方式实现 tmuduo 的线程模型时，我并没有引入异常处理机制。在此之前，我对 Google 禁用异常的说法略有耳闻，而陈硕大佬作为 Google 的工程师，却在自己的开源项目中用到了异常！？带着好奇心，我去查看了《Google C++ Style Guide》<sub>[1]</sub>,在 Exception 的那章找到了如下内容：
> Exception:
>   We do not use C++ exceptions.
> ...
> Decision:
>   On their face, the benefits of using exceptions outweigh the costs, especially in new projects. However, for existing code, the introduction of exceptions has implications on all dependent code. If exceptions can be propagated beyond a new project, it also becomes problematic to integrate the new project into existing exception-free code. Because most existing C++ code at Google is not prepared to deal with exceptions, it is comparatively difficult to adopt new code that generates exceptions.
>   Given that Google's existing code is not exception-tolerant, the costs of using exceptions are somewhat greater than the costs in a new project. The conversion process would be slow and error-prone. We don't believe that the available alternatives to exceptions, such as error codes and assertions, introduce a significant burden.

这下就清楚了，Google 不用异常的原因是 Google 有很多旧代码并不是异常安全的，将这些代码中引入异常处理，其工作量之巨大远超过异常所能够带来的好处。**如果你所做的项目没有历史包袱，那么适当的使用异常是利大于弊的(关于异常的优缺点，可以直接查看Google C++ Style Guide中的 Pros and Cons，这里就不做搬运工了)**。


#### tmuduo 的异常实现
```C++
class Exception : public std::exception {
 public:
  Exception(std::string what): message_(std::move(msg)),
      stack_(CurrentThread::stackTrace(/*demangle = */ false)) {}
  ~Exception() noexcept override = default;

  const char* what() const noexcept override { return message_.c_str(); }

  const char* stackTrace() const noexcept { return stack_.c_str(); }

 private:
  std::string message_;
  std::string stack_;
};
```
注意：本节内容中所用的异常代码和 muduo 网络库中是高度类似的，我仅是对其做了一点点微小的调整以使其适应我自己编写的 Thread 类。为了能和之前的文章保持一致，此处还是用 tmuduo 的代码。
从上述代码来看，整个异常类的代码非常简单，重点还是在于 `CurrentThread::stackTrace` 函数的实现上。不过在展示 `CurrentThread::stackTrace` 的代码实现之前，需要先了解两个基本概念 mangle 与 demangle


#### C++ 的 mangle 与 demangle
先简单地解释一下一些相关的概念：
> mangle   : 将 C++ 的源程序标识符(original C++ source identifier)转换成C++ ABI 标识符(C++ ABI identifier)的过程
> demangle : 将 C++ 的 ABI 标识符转换为源程序标识符的过程
> ABI(Application Binary Interface): 从命名上来看，指的是程序二进制层面上的接口，包括如符号修饰标准、变量内存布局、函数调用方式等等这些所有跟可执行代码二进制兼容性相关的内容。
> API(Application Programming Interface): 指的往往是源码级别的接口，例如 POSIX 是一个 API 标准，POSIX 线程库下的 `pthread_create`函数是一个 API。

由于历史原因，不同的编译器之间往往有各自的 C++ ABI 标准，所以同一份 C++ 代码使用不同的编译器，甚至是同一个编译器的不同版本所编译出来的程序都可能存在 ABI 不兼容问题。而 mangle 规则也算是 ABI 的一部分。我们可以先来看看在 Ubuntu-16.04 下的 clang 3.8.0-2 的 mangle 规则：
```C++
//mangle_test.cc
#include <iostream>
using namespace std;
void foo(int var){}
void foo(float var){}
int main(){
    return 0;
}
```
我们执行 `clang++ -c mangle_test.cc && nm mangle_test.o` 后可以看到如下结果：
<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/44a2de37-3cd7-4fa0-8822-2ce8889287c7.png"/>
</div>
<div class="image-caption" align="center">mangle_test</div> 

可以看到我们所写的重载函数 foo 经过了 mangle 后转变成了 \_Z3foof 和 \_Z3fooi。

#### CurrentThread::stackTrace 的实现
CurrentThread::stackTrace 具体代码保存在 tmuduo/base/CurrentThread.cc 文件中
```C++
std::string stackTrace(bool demangle) {
  std::string stack;
  const int max_frames = 200;
  void* frame[max_frames];
 
  int nptrs = ::backtrace(frame, max_frames);
  char** strings = ::backtrace_symbols(frame, nptrs);
  if (strings) {
    size_t len = 256;
    char* demangled = demangle ? static_cast<char*>(::malloc(len)) : nullptr;
    for (int i = 1; i < nptrs; ++i) {
      if (demangle) {
        char* left_par = nullptr;
        char* plus = nullptr;
        for (char* p = strings[i]; *p; ++p) {
          if (*p == '(')
            left_par = p;
          else if (*p == '+')
            plus = p;
        }
        if (left_par && plus) {
          *plus = '\0';
          int status = 0;
          char* ret =
              abi::__cxa_demangle(left_par + 1, demangled, &len, &status);
          *plus = '+';
          if (status == 0) {
            demangled = ret;
            stack.append(strings[i], left_par + 1);
            stack.append(demangled);
            stack.append(plus);
            stack.push_back('\n');
            continue;
          }
        }
      }
      stack.append(strings[i]);
      stack.push_back('\n');
    }
    free(demangled);
    free(strings);
  }
  return stack;
}
```
其中涉及到了三个新的函数：`backtrace`， `backtrace_symbols` 以及 `abi::__cxa_demangle`。其中 `backtrace` 和 `backtrace_symbols` 用于获得线程的函数调用栈信息，其具体用法可以在 linux 下的 man 手册中查看，这里不多展开。而 `abi::__cxa_demangle` 是 libstdc++ 库中的函数，具体的用法可以在 GNU 网站的《abi Namespace Reference》<sub>[2]</sub>中找到

这里再额外补充点：
> * 某些编译器的优化选项对获取正确的函数调用栈有干扰
> * 内联函数的调用信息也不会出现在函数调用栈当中
> * 如果使用 `abi::__cxa_demangle` 函数，则在连接时需要加上选项 `-rdynamic`，　注意这是一个　linker option 而非 compiler option，如果你使用 cmake-3.13 或以上的版本，可以使用 `add_link_options(-rdynamic)`

#### Exception 的测试结果
tmuduo 当中有关于 Exception 的测试，代码路径为 test/Exception_test.cc。由于调用栈信息比较长，以下只贴一部分的执行结果，有需要的可以自己把代码下下来跑一下
经 demangle 处理的异常信息

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/395917c1-6cd7-4d9f-ac76-39a5243a6d5f.png" width=735 height=120/>
</div>
<div class="image-caption" align="center">Exception with demangle</div> 

未经 demangle 处理的异常信息
<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/78c97cd1-4761-4be2-8442-83c2b9d0ff0e.png"/>
</div>
<div class="image-caption" align="center">Exception without demangle</div> 

从上述结果可以看出，同一个函数 Bar::test 在经过 demangle 处理的异常信息中显示为 `Bar::test(...)`， 在没有经过 demangle 处理的异常信息中显示为 `_ZN3Bar4testEbSt6vectorINSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEESaIS6_EE+0x1c`


#### 参考资料
[1]. [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
[2]. [abi Namespace Reference](https://gcc.gnu.org/onlinedocs/libstdc++/libstdc++-html-USERS-4.3/a01696.html)
[3].[C++ Code Snippet - Print Stack Backtrace Programmatically with Demangled Function Names](https://panthema.net/2008/0901-stacktrace-demangled/)
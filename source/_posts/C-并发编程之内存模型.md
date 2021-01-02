---
title: C++并发编程之内存模型
top: false
cover: false
toc: true
mathjax: true
date: 2020-12-29 21:22:41
password:
summary:
tags: [C++ 并发编程, C++ 内存模型]
categories: C++沉思录
---

### 文前导读

本文旨在对 C++11 中引入的内存模型，特别是内存序的部分进行一些深入的探讨，主要包含了以下内容

> * 关于 C++11 的内存模型
>   * 内存模型的重要性
>   * 内存模型是什么
> * C++ 内存模型之内存布局
> * C++ 内存模型之内存序
>   * 操作间关系
>     * sequenced-before 关系
>     * synchronizes-with 关系
>     * happens-before 关系
>   * 修改顺序和 Visible side-effects
>   * C++ 内存序模型
>     * Relaxed Ordering
>     * Release-Acquire Ordering
>     * Sequential Consistent Ordering
>     * 总结
> * 内存栅栏(Memory Fence)
>   * Acquire Fence
>   * Release Fence
>   * Full Fence
> * Reference

<!-- more -->

### 关于 C++11 的内存模型
##### 内存模型的重要性
纵观计算机的发展史，多线程概念的提出要远慢于操作系统和编译器的实现。在多线程提出以前，编译器和操作系统认为程序当中仅有一个执行流，很多编译器以及处理器的优化都建立在这个假设之上。而在多线程程序大行其道的今天，这种假设已经不再成立。编译器以及处理器的优化可能会对多线程程序的运行造成影响，主要体现在以下几方面：
> * 因编译器的优化而导致的指令重排：
> * CPU 的乱序执行
> * 多核 CPU 的 Cache 不一致问题
一些编译器，如 gcc 在编译代码时可以通过 `-O` 参数来指定优化级别，而其中就包含了指令重排的优化。例如 gcc 可以将程序中彼此没有依赖关系的访问内存操作放在一起，并为其生成相应的汇编代码。

对于 CPU 而言，存在乱序执行的一个直观例子就是计算机组成原理中，在讲解流水线时所提到的数据冒险。

而对于多核系统而言，往往多个核心之间配备有不同的 L1 Cache 和 L2 Cache，如图示：

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/a878211f-a4ec-4a8b-8459-6e945e1e51af.png"/>
</div>
<div class="image-caption" align="center">多核 CPU 结构</div>

如果 cache 采用写回法的缓存策略，每次只在将 cache 块替换出去时才将 cache 块中的数据写回内存。那么当 core1 和 core2 访问并修改相同数据时，就会面临数据不一致的问题。

由于存在上述的优化操作，开发人员编写的多线程程序在最终运行时往往表现出和他们一开始预期的不同，这就导致了 bug 的产生。为了解决上述问题，C++ 11 引入了新的内存模型。**事实上，有研究<sup>[0]</sup>表明，只有在语言层面上提出内存模型，才可以使得编译器对多线程程序进行优化变成可能**。

除了对于 C++11 中引入的标准线程库具有重大意义以外，对 C++ 内存模型，特别是其中的内存序的了解，也有助于让开发者在程序中引入 Lock-Free 策略，更好地提高多线程程序的并发性<sup>[1]</sup>。

##### 内存模型是什么？
C++11 中的内存模型**本质上是一套行为规范**，它包括了两个层面的内容，一个是结构层面的内容 —— 内存布局，描述了在语言视角下数据是如何存放于内存当中，另一个是并发层面的内容 —— 内存序， 描述**一个线程对某个共享对象的修改何时能够被其他线程看到**。而编译器在对 C++ 11 下的多线程程序进行编译及优化时，必须遵守这一套行为规范，特别是其中关于内存序的要求。

### C++ 内存模型之内存布局

在 C++ 中，所有的数据都是对象，而所谓的对象就是一片有属性的内存区域，其中属性包括了对象的类型以及生命周期能。不论对象的类型是什么，一个对象都必然占据一个或多个内存位置(Memory Location)。

每一个内存位置上的必定是以下两种情况之一：

> * 标量类型，包括数字类型、指针类型、枚举类型以及 `nullptr_t`
> * 相邻位域的最大序列

例如以下代码：

```C++
struct my_data{
	int var_1;
    double var_2;
    unsigned var_3:10;
    int var_4:25;
    int:0;
    int var_6;
    char* var_7;
    std::string var_8;
};
```

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/1027fc3a-f3f1-4663-a2d3-ee39a9ef04c5.png"/>
</div>
<div class="image-caption" align="center">Memory Layout</div>


### C++ 内存模型之内存序

##### 操作间的关系

###### sequenced-before 关系

sequenced-before 是一种非对称，可传递的成对关系，它描述了在同一线程下，表达值求值的顺序。有以下几种可能：

* A sequenced-before B：A 的求值将会在 B 的求值开始之前完成
* A 与 B 之间不存在任何方向的 sequenced-before 关系：
  * A 与 B 的求值之间不仅是无序的，且彼此之间可以重叠(CPU 交替执行 A 与 B 的求值指令)。
  * A 与 B 的求值之间是无序的，但彼此之间不可重叠。

通常而言，同一线程下的顺序语句在语义上总是符合 sequenced-before 关系的。例如 A 语句位于 B 语句之前，那么语义上满足 A sequenced-before B，但由于编译器的优化，实际执行中可能是 B 语句先执行。例如：

```C++
// 语义上符合 ① sequenced-before ②
// 但实际上，最终执行结果可能是先执行 ②; 再执行 ①
int a = 10;		// ①
int b = 100;	// ②
// 语义上符合 ③ sequenced-before ④，且由于存在依赖关系，因此 ③ 会在 ④ 之前执行
int var = 1024;	// ③
++var;			// ④
```

###### synchronizes-with 关系

synchronizes-with 描述的是一种线程间的同步关系，**如果 A synchronizes-with B， 那么可以确认 A 状态在 B 执行之前一定是可见的**。像原子操作的 acquire-release、mutex 的 lock 与 unlock，thread 的 create 与 join 之间都可以构成 synchronizes-with 关系。使用 synchronizes-with 可以用来构成 inter-threads happen-before 关系

###### happens-before 关系

A、B 是两个在多核 CPU 上执行的操作。**如果 A happens-before B，那么 A 所产生的内存变化会在 B 执行之前被看到（visible）**。happens-before 主要包含了两种情况：

* 如果 A 和 B 位于同一线程内，当 A sequenced-before B 时，可以说 A happens-before B
* 如果 A 和 B 位于不同的线程内，当 A inter-threads happens-before B 时，可以说 A happens-before B

inter-threads happens-before 关系，顾名思义，就是发生在线程间的 happens-before 关系。它包含但又不限于前面所提到的 synchronizes-with 关系。例如

```C++
// write thread
void writer_thread(){
 data.push_back(42); 	// ①
 data_ready=true; 		// ②
}
// read thread
void read_thread(){
 while(!data_ready.load())	// ③
 	std::this_thread::sleep(std::milliseconds(1));
 std::cout<<”The answer=”<<data[0]<<”\n”; 	//④
}
```

从上述代码中，① sequenced-before ②、③ sequenced-before ④、② synchronizes-with ④，因此我们有 ① happens-before ④， ② happens-before ④。



##### 修改顺序 和 Visible side-effects

C++ 标准中对 side-effects 的定义如下：

> \[intro.execution\]:
>
> Reading an object designated by a volatile glvalue (3.10), modifying an object, calling a library I/O function, or calling a function that does any of those operations are all side effects, which are changes in the state of the execution environment.

**简单地说，所谓的 side-effect 就是指当执行完一个对表达式的求值操作后，导致了被操作对象状态的改变**。例如 执行完 `int var = 10; int i = var++;`后，表达式 `var++`的结果是 10，但 var 的状态发生了改变(从10 变成11)，那么 `var++`就产生了一个 side-effect.

在 C++ 中对于某个特定的原子变量的所有访问操作都存在一个修改序列。对于程序的每次运行而言，这一修改序列可能不同，但针对具体的某一次运行，都必然有一个确定的顺序。而**所谓的修改顺序的一致性，就是指所有访问该原子变量的线程都只会看到同一个修改序列**。换句话讲，**一旦这个原子变量的状态发生改变，那么这个改变后的状态应当同步给所有的线程，这样就避免了在访问同个原子变量的多个线程之间出现数据不一致的问题**。



##### C++ 内存序模型

内存序的定义：

```C++
typedef enum memory_order {
    memory_order_relaxed,
    memory_order_consume,
    memory_order_acquire,
    memory_order_release,
    memory_order_acq_rel,
    memory_order_seq_cst
} memory_order;
```

注意：由于 C++ 17 中重新修订了 release-consume ordering 的规范，C++ 并不提倡使用 `memory_order_consume`，因此在接下来的讨论中，我们将不会涉及 `memory_order_consume`的讨论。

C++ 中提供了三种不同的内存序，按照约束条件强弱划分，由弱到强可分为：Relaxed Ordering、Release-Acquire Ordering 以及 Sequential Consistent Ordering。它们和 C++ 中的 memory_order 之间的关系如下：

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/d394ee7b-3ec5-4db9-8b4b-63928766d5f9.png" width=800 height=400/>
</div>
<div class="image-caption" align="center">原子操作、内存序模型与memory_order</div>

为了更清晰的说明问题，我将 memory_order 和相关的内存序模型放在一起讨论。

###### Relaxed Ordering

Relaxed Ordering: 这一模型是 C++ 内存序模型中约束条件最宽松的。对于指定了 `memory_order_relaxed`的原子操作，C++ 语言做出了以下两个承诺(注意黑体字)：

> * 操作是原子性的
> * 对于**某个特定的原子变量**，保证其在**线程内的**修改一致性，对于**跨线程的**修改一致性不做任何保证。

我们可以看一下下面这段代码：

```C++
#include <atomic>
#include <thread>
#include <assert.h>
std::atomic<bool> x,y;
std::atomic<int> z;
void write_x_then_y(){
	x.store(true,std::memory_order_relaxed);	// ①
	y.store(true,std::memory_order_relaxed); 	// ②
}
void read_y_then_x(){
	while(!y.load(std::memory_order_relaxed)); 	// ③
	if(x.load(std::memory_order_relaxed)) 		// ④
		++z;
}
int main(){
	x=false;
	y=false;
	z=0;
	std::thread a(write_x_then_y);
	std::thread b(read_y_then_x);
	a.join();
	b.join();
	assert(z.load()!=0); 
}
```

在下面这段代码中，`assert(z.load()!=0);` 是有可能会被触发的。这是因为对于线程 a 而言，交换 ① 和 ② 并不违背 Relaxed Ordering 所作出的承诺：① 和 ② 分别操作的是 x 和 y 两个原子变量，且 x 和 y 各自的修改序列中就只包含了一个操作，因此**C++ 允许编译器或 CPU 对 ① 或 ② 进行指令重排**。一旦 ① 和 ② 的指令顺序对调，且线程 a 与 b 的执行序列变成 ② => ③ => ④ => ①，那么 `assert(z.load()!=0);` 就会发生。



###### Release-Acquire Ordering

在讨论 Release-Acquire ordering 之前，我们需要先知道以下两个概念以及一个例子：

> * Acquire Operation: 被标记为 memory_order_acquire 或 memory_order_acq_rel 的 load 操作，或者 mutex 的 lock 操作，相当于读操作
>
> * Release Operation: 被标记为 memory_order_release 或 memory_order_acq_rel 的 store 操作，或者 mutex 的 unlock 操作，相当于写操作

假设现有两个线程 A 和 B，以及一个全局的原子变量 var，如果 A 先对 var 执行了 release 操作(store操作)，而 B 后对 var 执行了 acquire 操作(load操作)，那么当 B 执行完 load 操作后，它将能够看到 A 在 store 操作之前所执行的所有写操作(不管是对非原子变量还是其他的原子变量)，如下图：

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/2e45bb10-2727-4684-9fad-ed598f4d0af4.png" width=800 height=400/>
</div>
<div class="image-caption" align="center">Acquire-Release Ordering Demo</div>

从上面的例子可以看出，Release-Acquire Ordering 为不同线程中对同一原子变量的一对 acquire 操作和 release 操作之间引入了 synchronizes-with 关系，它做出如下承诺：

> * 任何发生在 release 操作之前的读写操作都不会被重排到 release 操作之后
> * 任何发生在 acquire 操作之后的读写操作都不会被重排到 acquire 操作之前

另外，这层关系**只存在于**执行 acquire 操作和 release 操作的两个线程 A 和 B 之间。换言之，如果有其他线程(比如 C)正在运行，那么 Release-Acquire Ordering 并不对 C 所看到的修改序列做任何保证。换句话讲，C 看到的 A 对变量的修改序列既有可能和 B 看到 A 的修改序列相同，也有可能不同，语言不做保证，具体的情况由编译器或 CPU 的优化结果决定。

对于前面 Relaxed Ordering 中的例子而言，我们只需要指定 y 的 load 和 store 操作序列为 Release_Acquire Ordering 就可以确保`assert(z.load()!=0);`不会被触发。

```C++
void write_x_then_y(){
	x.store(true,std::memory_order_relaxed);	// ①
	y.store(true,std::memory_order_release); 	// ②
}
void read_y_then_x(){
	while(!y.load(std::memory_order_acquire)); 	// ③
	if(x.load(std::memory_order_relaxed)) 		// ④
		++z;
}
```

另外，在一些重视指令执行序列(strongly-ordered)的硬件平台，如 x86 等，大多数操作都是符合 Release-Acquire Ordering 的，编译器在编译程序时无需生成额外的指令来保证 Release-Acquire Ordering。而在一些弱序性(weakly-ordered)的硬件平台，如 ARM 上，则需要通过内存栅栏(memory fence)来实现。

总的来说，**Release-Acquire Ordering 相当于提供了两条"基线"，位于 release 线之前的相关指令不能越过这条线到这条线之后去执行，而位于 acquire 线之后的相关指令则不能越过 acquire 线跑到前面去执行**。另外，由于 std::mutex 本身的 lock 与 unlock 也属于 acquire 和 release 操作，因此也符合 Release-Acquire Ordering，其中临界区的概念就是被 acquire 线和 release 线所包围的那片区域。



###### Sequential Consistent Ordering

**Sequential Consistent Ordering 是C++ 所提供的内存序中约束性最强的一种，它将 Release-Acquire Ordering 中，对同一原子变量的修改序列从只在执行 release 操作的线程和执行 acquire 操作的线程之间保持同步，推广到了在全局所有线程中保持同步。**换句话讲，套用 Release-Acquire Ordering 的例子，在使用 Sequential Consistent Ordering 的情况下，C 线程看到的 A 线程中对变量的修改序列和 B 线程看到的 A 线程中对变量的修改序列是完全一致的。

任何指定为 memory_order_seq_cst 或采用默认内存序的原子操作都采用此内存序。

我们可以将前面的例子扩展到多个线程，如下：

```C++
#include <atomic>
#include <thread>
#include <assert.h>
std::atomic<bool> x,y;
std::atomic<int> z;
void write_x(){
    x.store(true, std::memory_order_seq_cst);
}
void write_y(){
    y.store(true, std::memory_order_seq_cst);
}
void read_x_then_y(){
    while (!x.load(std::memory_order_seq_cst));
    if (y.load(std::memory_order_seq_cst)) {
        ++z;
    }
}
void read_y_then_x(){
    while (!y.load(std::memory_order_seq_cst));
    if (x.load(std::memory_order_seq_cst)) {
        ++z;
    }
}
int main(){
    x=false;
	y=false;
	z=0;
    std::thread a(write_x);
    std::thread b(write_y);
    std::thread c(read_x_then_y);
    std::thread d(read_y_then_x);
    a.join(); b.join(); c.join(); d.join();
    assert(z.load() != 0);  // will never happen
}
```

在上述代码中，只有指定内存序为 Sequential Consistent Ordering 才能确保 `assert(z.load() != 0)`不会被触发。如果指定操作的内存序为 Release-Acquire Ordering，那么 c 和 d 就有可能会看到关于 x 和 y 的不同的修改序列。

在所有多核硬件系统平台上，Sequential Consistent Ordering 都需要使用 memory fence 指令来实现，因此它会将特定内存位置的访问同步到所有的核中，这可能会造成性能瓶颈。

最后，关于 Sequential Consistent Ordering 有两个值得仔细分辨的点：

> * 全局一致的修改顺序强调的是在程序的每次运行下，所有线程都能看到某个原子变量的统一的修改顺序。但是不同次运行中，同一个变量的修改顺序不一定是相同的。例如程序第一次运行所有线程所看到的某一个变量的修改顺序和程序第二次运行所有时所有线程看到的修改顺序不同。
> * 在很多情况下，对于同一个线程下**不同**原子变量的 memory_order_seq_cst 操作是可以被重排序的

###### 总结

C++ 并发编程困难不仅在于开发人员需要捋清不同线程之间复杂的协作关系，还需要**在一定程度上对抗编译器和 CPU 所做出的优化**。C++ 提供三种不同的内存序，旨在能够让开发者根据自身需要自行选择，以**最低限度**约束编译器及 CPU 的优化。

从上述三个内存序来看，**Relaxed Ordering 只保证了在同线程内的单个原子变量的修改一致性，而 Release-Acquire Ordering 则通过一对 release 和 acquire 操作，将这种修改一致性扩展到了相关的两个线程当中，而 Sequential Consistent Ordering 则将这种修改一致性扩展到了全局的所有线程当中**。



### 内存栅栏(Memory Fence)

在 C++ 中，除了使用原子操作指定内存序以外，还可以通过内存栅栏来指定内存序。根据指定都能内存序的不同，fence 可以分成以下三类：

> * acquire fence：指定内存序为 memory_order_acquire 的 std::atomic_thread_fence
> * release fene：指定内存序为 memory_order_release 的 std::atomic_thread_fence
> * full fence：指定内存序为 memory_order_seq_cst 或者 memory_order_acq_rel 的 std::atomic_thread_fence

对于编译器及处理器的乱序重排，一共可以分为四种情况：

> * Load-Load(读后读)
> * Load-Store(读后写)
> * Store-Load(写后读)
> * Store-Store(写后写)

指令重排的结果可以用下面这张图来表示(其中文字代表操作类型，箭头表示指令重排的移动方向)：

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/acd43408-e2fa-4c43-8d0f-7995f488b4b0.png"/>
</div>
<div class="image-caption" align="center">Instruction Reordering</div>

如果理解了前面的 Release-Acquire Ordering 中关于 acquire 基线和 release 基线的描述，就可以很容易了理解接下来的内容了。

##### Acquire Fence

如同前面所提到的 acquire 基线那样， Acquire Fence 可以防止 Fence 后的读写操作重排到 Fence 前的**任意** load 操作之前，即抑制 Load-Load 重排和 Load-Store 重排，如下图：

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/0d925d2a-3758-40a7-b940-bb4241d14855.png"/>
</div>
<div class="image-caption" align="center">Acquire Fence</div>

##### Release Fence

如同前面所提到的 release 基线那样， 防止 Fence 前的读写操作重排到 Fence 后的**任意** store 操作之后，即抑制 Load-Store 重排和 Store-Store 重排，如下图：

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/0388aa3e-d098-4770-9ae9-7114ba8a5d50.png"/>
</div>
<div class="image-caption" align="center">Release Fence</div>

注意：Acquire Fence 和 Release Fence 在同步关系上的约束性其实要比 Release-Acquire Ordering 中相应的"acquire 基线" 和 "release 基线"要强，我想可能是因为在 Release-Acquire Ordering 中 "acquire 基线"本身就是一个 acquire 操作，"release 基线"本身也是一个 release 操作。而 Acquire Fence 和 Release Fence 严格意义上并不算操作。

##### Full Fence

Full Fence 就相当于是前面 Acquire Fence 和 Release Fence 功能的并集，它能够抑制 Load-Load、Load-Store 以及 Store-Store 重排，但是它不能抑制 Store-Load 重排。

<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/d19c5829-c7e5-4a48-8f9e-322d0a268966.png"/>
</div>
<div class="image-caption" align="center">Full Fence</div>

回到前面 Relaxed Ordering 的例子中，我们也可以通过 Release Fence 来防止语句 ① 和语句 ② 之间的重排序，例如：

```C++
void write_x_then_y(){
	x.store(true,std::memory_order_relaxed);	// ①
    std::atomic_thread_fence(std::memory_order_release);
	y.store(true,std::memory_order_relaxed); 	// ②
}
void read_y_then_x(){
	while(!y.load(std::memory_order_relaxed)); 	// ③
    std::atomic_thread_fence(std::memory_order_acquire);
	if(x.load(std::memory_order_relaxed)) 		// ④
		++z;
}
```



### Reference

[0]. [《Threads Cannot be Implemented as a Library》—— Hans-J. Boehm](https://www.hpl.hp.com/techreports/2004/HPL-2004-209.pdf)

[1]. [An Introduction to Lock-Free Programming](https://preshing.com/20120612/an-introduction-to-lock-free-programming/)

[2]. [《C++ Concurrency In Action》 —— Anthony Williams](https://www.amazon.com/-/zh/dp/1933988770/ref=sr_1_2?__mk_zh_CN=%E4%BA%9A%E9%A9%AC%E9%80%8A%E7%BD%91%E7%AB%99&crid=EHHD3PRN1G0Y&dchild=1&keywords=c%2B%2B+concurrency+in+action&qid=1609319303&sprefix=C%2B%2B+Concurrency+%2Caps%2C365&sr=8-2)

[3]. [《C++11 Momery Model》 —— Scott B. Baden](https://cseweb.ucsd.edu/classes/fa13/cse160-a/Lectures/Lec07.pdf)

[4]. [cppreference.com:memory-order](https://en.cppreference.com/w/cpp/atomic/memory_order)

[5]. [cppreference.com:std::atomic_thread_fence](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence)


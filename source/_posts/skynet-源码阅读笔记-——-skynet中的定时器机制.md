---
title: skynet 源码阅读笔记 —— skynet中的定时器机制
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-04 17:22:37
password:
summary:
tags: [skynet框架, Timer, C语言]
categories: 源码剖析
---



#### 文前导读

skynet 是一个由云风所写的轻量级在线游戏服务器框架。本文为 skynet 框架源码剖析系列的第四篇文章，探讨了 skynet 的定时器机制，主要包含了以下内容：
> * 定时器的基本数据结构
> * starttime、current 以及 current_point 的意义，包括了 CLOCK_REALTIME 以及 CLOCK_MONOTONIC 等内容
> * time、near 数组以及 t 数组的意义
> * time、near 数组以及 t 数组三者之间的联系

<!-- more -->

#### 基本数据结构
要了解 skynet 的定时器机制，需要先了解 skynet 中的 `timer` 的数据结构及初始化代码(skynet 中所有 timer 相关的代码都存放于 skynet_timer.c 文件中)：
```C
#define TIME_NEAR_SHIFT 8
#define TIME_NEAR (1 << TIME_NEAR_SHIFT)
#define TIME_LEVEL_SHIFT 6
#define TIME_LEVEL (1 << TIME_LEVEL_SHIFT)
// TIME_NEAR_MASK = 0x11111111
#define TIME_NEAR_MASK (TIME_NEAR-1)
// TIME_LEVEL_MASK = 0x111111
#define TIME_LEVEL_MASK (TIME_LEVEL-1)
//超时事件
struct timer_event {
    uint32_t handle;    //标记该超时时间所对应的服务
    int session;        //超时事件发送消息所属的 handle
};

//定时器节点
struct timer_node {
    struct timer_node *next;
    uint32_t expire;    //超时事件
};

//定时器链表
struct link_list {
    struct timer_node head;
    struct timer_node *tail;
};

struct timer {
    struct link_list near[TIME_NEAR];
    struct link_list t[4][TIME_LEVEL];
    struct spinlock lock;
    uint32_t time;
    uint32_t starttime;     
    uint64_t current;       
    uint64_t current_point; 
};

static struct timer * TI = NULL;
```
从上述数据结构的定义中可以知道，skynet 采用 `timer_event` 来表示超时事件，其中 `handle` 代表了该超时事件属于哪个服务，而 `session` 则代表向对应服务所发送的超时消息的 session。skynet 采用了带头节点的单链表来存储多个定时器。

#### starttime、current 以及 current_point 的意义
要想了解 上述三个字段的具体意义，我们需要先了解 timer 是如何被初始化，以及节点是如何添加到 timer 当中的。在说明 `skynet_timer_init` 之前，需要花点时间说明 `clock_gettime` 中不同的时间类别，也就是所谓的 clock_id. `clock_gettime` 支持多种不同的 `clk_id`, 其中包括但不限于:`CLOCK_REALTIME`、`CLOCK_MONOTONIC`、`CLOCK_PROCESS_CPUTIMEID` 和 `CLOCK_THREAD_CPUTIME_ID`
>* CLOCK_REALTIME：墙上时间(wall time)，也就是我们现实生活中所用的时间，由变量xtime来记录。系统每次启动时将CMOS上的RTC时间读入xtime，这个值是"自1970-01-01起经历的秒数、本秒中经历的纳秒数"，每来一个timer interrupt，也需要去更新xtime。其值为从 1970-01-01:00:00:00 至今所流逝的时间。
>* CLOCK_MONOTONIC：单调时间(monotonic time)，代表的从系统启动至今所流逝的时间，由变量jiffies来记录。系统每次启动时jiffies初始化为0，每来一个timer interrupt，jiffies加1，也就是说它代表系统启动后流逝的tick数。jiffies一定是单调递增的
>* CLOCK_PROCESS_CPUTIMEID：进程专属的 CPU 时钟，代表从进程启动后至今所流逝的时间
>* CLOCK_THREAD_CPUTIME_ID：线程专属单 CPU 时钟，代表从线程启动后至今所流逝的时间

其中，`CLOCK_REALTIME` 和 `CLOCK_MONOTONIC` 的区别在于 `CLOCK_REALTIME` 的值可以受到系统时间跳变或 NTP 的影响， 而`CLOCK_MONOTONIC` 不会受到影响，因此常用 `CLOCK_MONOTONIC` 来计算系统启动后两个先后发生的事件之间的时间差

```C
//创建一个 timer 结构，并将其中 near 以及 t 链表数组清空
static struct timer* timer_create_timer() {
    struct timer *r=(struct timer *)skynet_malloc(sizeof(struct timer));
    memset(r,0,sizeof(*r));
    int i,j;
    for (i=0;i<TIME_NEAR;i++) {
        link_clear(&r->near[i]);
    }
    for (i=0;i<4;i++) {
        for (j=0;j<TIME_LEVEL;j++) {
            link_clear(&r->t[i][j]);
        }
    }
    SPIN_INIT(r)
    r->current = 0;
    return r;
}
void  skynet_timer_init(void) {
    TI = timer_create_timer();
    uint32_t current = 0;
    systime(&TI->starttime, &current);
    //TI->starttime 保存了当前墙上时间，精确到秒
    //TI->current 从 starttime 启动后到当前的时间，精确到 10 ms
    TI->current = current;
    //TI->current_point 代表是精确到 10 ms的单调时间，表示从系统启动到当前所流失的时间
    TI->current_point = gettime();
}

//获得当前系统的墙上时间，并将其中的整秒部分存入 sec 中，将纳秒部分转化为以 10 ms 为精度的 centisecond,并存入 cs 中
static void systime(uint32_t *sec, uint32_t *cs) {
#if !defined(__APPLE__) || defined(AVAILABLE_MAC_OS_X_VERSION_10_12_AND_LATER)
    struct timespec ti;
    clock_gettime(CLOCK_REALTIME, &ti);
    *sec = (uint32_t)ti.tv_sec;
    *cs = (uint32_t)(ti.tv_nsec / 10000000);
#else
    struct timeval tv;
    gettimeofday(&tv, NULL);
    *sec = tv.tv_sec;
    *cs = tv.tv_usec / 10000;
#endif
}
//获得当前系统的单调时间，并将其转换成为以 10ms 为精度的时间格式
static uint64_t gettime() {
    uint64_t t;
#if !defined(__APPLE__) || defined(AVAILABLE_MAC_OS_X_VERSION_10_12_AND_LATER)
    struct timespec ti;
    clock_gettime(CLOCK_MONOTONIC, &ti);
    t = (uint64_t)ti.tv_sec * 100;
    t += ti.tv_nsec / 10000000;
#else
    struct timeval tv;
    gettimeofday(&tv, NULL);
    t = (uint64_t)tv.tv_sec * 100;
    t += tv.tv_usec / 10000;
#endif
    return t;
}
```
从上述代码中，我们可以知道 `starttime` 代表的是 timer 初始化的墙上时间，精确到秒，而 `current` 则相当于 timer 启动后至今的时间差(也就是 timer 的运行时间)，精度为 10 ms，而 `current_point` 则相当于从系统开机至今经过的时间，精度同样为 10ms

#### time、near 数组以及 t 数组的意义
timer 一旦完成初始化后，就会交给 timer 线程去使用，为了了解上述三个字段的含义以及定时器背后的流程，我们需要先阅读 timer 线程的线程函数
```C
//skynet_timer.c
static void* thread_timer(void *p) {
    struct monitor * m = p;
    skynet_initthread(THREAD_TIMER);
    for (;;) {
        skynet_updatetime();
        skynet_socket_updatetime();
        CHECK_ABORT
        wakeup(m,m->count-1);
        usleep(2500);
        if (SIG) {
            signal_hup();
            SIG = 0;
        }
    }
    // wakeup socket thread
    skynet_socket_exit();
    // wakeup all worker thread
    pthread_mutex_lock(&m->mutex);
    m->quit = 1;
    pthread_cond_broadcast(&m->cond);
    pthread_mutex_unlock(&m->mutex);
    return NULL;
}
```
从上述代码可以看出，timer 的调用主要通过 `skynet_updatetime` 函数来实现(`skynet_socket_updatetime` 函数的部分会放到网络当中讲)。继续追踪相应的函数：
```C
static void add_node(struct timer *T,struct timer_node *node) {
    uint32_t time=node->expire;     //节点的超时时间
    uint32_t current_time=T->time;  
    //判断 time 和 current_time 之间的间隔是否小于 256 个 tick(2560ms)
    if ((time|TIME_NEAR_MASK)==(current_time|TIME_NEAR_MASK)) {
        link(&T->near[time&TIME_NEAR_MASK],node);
    } else {
        int i;
        uint32_t mask=TIME_NEAR << TIME_LEVEL_SHIFT;
        //找到合适的 level 添加节点
        for (i=0;i<3;i++) {
            if ((time|(mask-1))==(current_time|(mask-1))) {
                break;
            }
            mask <<= TIME_LEVEL_SHIFT;
        }

        link(&T->t[i][((time>>(TIME_NEAR_SHIFT + i*TIME_LEVEL_SHIFT)) & TIME_LEVEL_MASK)],node);    
    }
}

//将 t[level][idx] 中的链表取出，并将其中的节点插入 near 当中 
static void move_list(struct timer *T, int level, int idx) {
    struct timer_node *current = link_clear(&T->t[level][idx]);
    while (current) {
        struct timer_node *temp=current->next;
        add_node(T,current);
        current=temp;
    }
}

void skynet_updatetime(void) {
    //获得以 10 ms 为精度的单调时间
    uint64_t cp = gettime();
    if(cp < TI->current_point) {
        skynet_error(NULL, "time diff error: change from %lld to %lld", cp, TI->current_point);
        TI->current_point = cp;
    } else if (cp != TI->current_point) {
        //获得时间差
        uint32_t diff = (uint32_t)(cp - TI->current_point);
        TI->current_point = cp;
        //更新 timer 的运行时间
        TI->current += diff;
        int i;
        for (i=0;i<diff;i++) {
            timer_update(TI);
        }
    }
}
static void timer_update(struct timer *T) {
    SPIN_LOCK(T);
    // try to dispatch timeout 0 (rare condition)
    timer_execute(T);   //看看 near[T->time & TIME_NEAR_MASK] 中的链表是否为空
    // shift time first, and then dispatch timer message
    timer_shift(T); //移动链表，将 t 中链表移动值
    timer_execute(T);
    SPIN_UNLOCK(T);
}

static inline void timer_execute(struct timer *T) {
    //取出 time 的低 8 位，idx 代表了当前超时
    int idx = T->time & TIME_NEAR_MASK;
    
    while (T->near[idx].head.next) {
        struct timer_node *current = link_clear(&T->near[idx]);
        SPIN_UNLOCK(T);
        // dispatch_list don't need lock T
        dispatch_list(current);
        SPIN_LOCK(T);
    }
}
static void timer_shift(struct timer *T) {
    int mask = TIME_NEAR;
    uint32_t ct = ++T->time;
    //ct == 0 代表发生了溢出
    if (ct == 0) {
        //将 t[3][0] 中链表取出并依次添加
        move_list(T, 3, 0);
    } else {
        uint32_t time = ct >> TIME_NEAR_SHIFT;
        int i=0;

        while ((ct & (mask-1))==0) {
            int idx=time & TIME_LEVEL_MASK;
            if (idx!=0) {
                move_list(T, i, idx);
                break;              
            }
            mask <<= TIME_LEVEL_SHIFT;
            time >>= TIME_LEVEL_SHIFT;
            ++i;
        }
    }
}
```
在上述代码中，可以看到每调用一次 `timer_shift`， `time`就会自增 1，而 `skynet_updatetime` 中一共执行了 diff 次 `timer_shift`。因此 `time` 代表了**从 timer 启动后至今一共经历了多少次 tick(一次 tick 的长度为 10ms)**。而且从 `timer_shift` 函数我们可以看出`time`和`near`数组以及`t`数组关系：
<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/6f6e7aab-276f-40d8-9478-1b01b0a161f1.png"/>
</div>
<div class="image-caption" align="center">time 与 near 以及 t 的关系</div>


如上图所示，skynet 按照超时时间的紧迫程度为 timer 划分出 5 个槽，其中紧急程度为 near > level0 > level1 > level2 > level3。其中，`near` 中的定时器节点超时时间相差最大不超过 2^8 = 256 次 tick，而对于同一个 level 而言，t[level] 中的定时器超时时间间隔不超过 2^6 = 64 次 tick。 `time` 中不同的位域代表了不同的紧急程度。`timer_execute` 每次只对 `near` 中的定时器执行超时操作。

了解了上述内容，我们就能够明白 skynet 是怎么样运转定时器的：**skynet 的 timer 线程会不断触发 `skynet_update` 函数，在该函数中会不断执行 `timer_execute`对 `near` 中的定时器执行超时操作。执行完毕后，调用`timer_shift` 从 `t[0]`~`t[3]` 中选择合适的定时器节点加入到 `near` 中，这一过程就相当于提高了定时器节点的紧急程度(因为随着时间的流逝，定时器节点的紧急程度会越来越向 near 逼近)。**

讲完了 skynet 定时器的运转流程，最后来看看为什么在函数 `move_list` 中，当 `time` 发生回绕时，为什么直接将 `t[3]` 放到 `near` 当中？这主要是因为添加节点采用的是位运算的方式，因此当发生 time 发生回绕时，低位会全部变为0，因此 `t[0]` ～ `t[3]` 都会被接连移动到 `near` 当中, 所以出于效率的考虑，直接将 `t[3]` 移入 `near` 即可
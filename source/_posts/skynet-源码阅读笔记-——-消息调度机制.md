---
title: skynet 源码阅读笔记 —— 消息调度机制
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-04 16:35:38
password:
summary:
tags: [源码剖析, skynet框架, 消息队列, 多线程, C语言]
categories: skynet源码剖析
---

#### 文前导读

skynet 是一个由云风所写的轻量级在线游戏服务器框架。本文为 skynet 框架源码剖析系列的第一篇文章，主要探讨 skynet 的消息调度机制，包含了以下内容：

> * 基本数据结构：包括消息队列以及消息本身的数据结构
> * 消息调度的过程：主要包括了消息是如何生产，又如何消费的
> * 多线程模型：skynet 的线程池设计，如何为进程设置合理的优先级使其拥有较高的 CPU 利用率，如何做线程监控设计(monitor) 以及如何高效地实现线程安全。

<!-- more -->

#### 基本数据结构之消息队列

skynet 采用了**二级消息队列**模式，其中顶层消息队列为 `global_queue`，而底层的消息队列为 `message_queue`，它们的具体定义如下：

```C
//skynet_mq.c
struct message_queue {
    struct spinlock lock;   //自旋锁，避免多个线程同时向一个队列中 push 消息时导致的竞态问题
    uint32_t handle;    //服务句柄。表明该消息队列具体属于哪个服务
    int cap;            //消息队列的容量
    int head;
    int tail;
    int release;        //是否可以释放信息
    int in_global;      //是否位于全局队列当中
    int overload;       //是否过载
    int overload_threshold; //过载上限
    struct skynet_message *queue;   //指向消息队列中存放消息的一片内存区域
    struct message_queue *next;     //指向下个次级消息队列的指针
};

struct global_queue {
    struct message_queue *head;
    struct message_queue *tail;
    struct spinlock lock;
};
```

skynet 的消息队列形式如下：
<div style="text-align:center">
 <img src="http://qj9elheb0.hn-bkt.clouddn.com/%E4%BA%8C%E7%BA%A7%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97.png" width=800 height=505/>
</div>
<div class="image-caption" align="center">二级消息队列</div>

#### 基本数据结构之消息
skynet 中一共支持两种不同的消息，一种为本地消息`skynet_message` ，另一种则为远程消息 `remote_message`。其中，`skynet_message`和 `remote_message` 如下：
```C
//skynet_mq.h
struct skynet_message {
    uint32_t source;	//发送的源地址
    //session 用于将请求包和响应包匹配起来。当一个服务向另一个服务发起请求时，会产生一个 session
    //当响应端处理完毕后，会将 session 原样返回，这样请求端就可以根据 session 找到对应的结果
    int session;		
    void * data;
    size_t sz;
};
//skynet_harbor.h
#define GLOBALNAME_LENGTH 16
//remote_name 代表一个远程 skynet 节点。
struct remote_name {
	char name[GLOBALNAME_LENGTH];
	uint32_t handle;
};
struct remote_message {
	struct remote_name destination;
	const void * message;
	size_t sz;
	int type;
};
```
这里解释一下上述消息定义中的 `session` 和 `type` 字段。**`session` 主要用来匹配一对请求和响应**。当一个服务向另一个服务提起请求时，会生成一个 session，并跟随请求包一并发送出去。接收端接收到包并处理完毕后，再将同样的 session 返回。这样，编写服务的人只需要在服务的 callback 函数中记录下所有发送出去的 session 就可以在收到每个消息后调用正确的处理函数。而 **`type` 主要是用来区分不同的消息包类型**。type 的定义如下：
```C
#define PTYPE_TEXT 0		//文本类型
#define PTYPE_RESPONSE 1	//响应包
#define PTYPE_MULTICAST 2	//组播包
#define PTYPE_CLIENT 3		//客户端消息
#define PTYPE_SYSTEM 4		//系统消息
#define PTYPE_HARBOR 5		//集群内其他的 skynet 节点发来的消息
#define PTYPE_TAG_DONTCOPY 0x10000		//禁止拷贝
#define PTYPE_TAG_ALLOCSESSION 0x20000	//分配新的 session
```
#### 谁生产，谁消费？
在 skynet 中，每个服务都拥有自己的一个次级消息队列。一个服务给另一个服务发送消息的过程，本质上就是将一个 skynet_message 压入到目标服务的次级消息队列当中。当一个服务的次级消息队列非空时，skynet 会将其push 到全局消息队列当中。而消息的消费，则是由线程池中的 worker 线程来完成，其大致的框图如下：
<div style="text-align:center">
 <img src="http://qj9elheb0.hn-bkt.clouddn.com/%E7%94%9F%E4%BA%A7%E8%80%85%E6%B6%88%E8%B4%B9%E8%80%85%E7%AE%A1%E7%90%86%E8%80%85%E6%A8%A1%E5%9E%8B.png" width=800 height=366/>
</div>
<div class="image-caption" align="center">生产者消费者管理者模型</div>

###### 消息消费的过程

在 skynet 启动的时候，会根据配置文件的 `thread` 字段初始化线程池。其中线程池中的前三个线程是 `monitor`, `timer` 和 `socket` 线程。其中，monitor 线程主要负责检查每个服务是否陷入了死循环，socket 线程负责网络相关操作，timer 线程则负责定时器。对应代码如下：

```C
//skynet_start.c
static void* thread_worker(void *p) {
	struct worker_parm *wp = p;
	int id = wp->id;
	int weight = wp->weight;
	struct monitor *m = wp->m;
	struct skynet_monitor *sm = m->m[id];
	skynet_initthread(THREAD_WORKER);
	struct message_queue * q = NULL;
	while (!m->quit) {
		q = skynet_context_message_dispatch(sm, q, weight);
		if (q == NULL) {
			if (pthread_mutex_lock(&m->mutex) == 0) {
				++ m->sleep;
				// "spurious wakeup" is harmless,
				// because skynet_context_message_dispatch() can be call at any time.
				if (!m->quit)
					pthread_cond_wait(&m->cond, &m->mutex);
				-- m->sleep;
				if (pthread_mutex_unlock(&m->mutex)) {
					fprintf(stderr, "unlock mutex error");
					exit(1);
				}
			}
		}
	}
	return NULL;
}

static void start(int thread) {
	pthread_t pid[thread+3];
	struct monitor *m = skynet_malloc(sizeof(*m));
	memset(m, 0, sizeof(*m));
	m->count = thread;
	m->sleep = 0;
	m->m = skynet_malloc(thread * sizeof(struct skynet_monitor *));
	int i;
	for (i=0;i<thread;i++) {
		m->m[i] = skynet_monitor_new();
	}
	if (pthread_mutex_init(&m->mutex, NULL)) {
		fprintf(stderr, "Init mutex error");
		exit(1);
	}
	if (pthread_cond_init(&m->cond, NULL)) {
		fprintf(stderr, "Init cond error");
		exit(1);
	}
	//创建 monitor 线程负责监视所有的 worker 线程
	create_thread(&pid[0], thread_monitor, m);
	create_thread(&pid[1], thread_timer, m);
	create_thread(&pid[2], thread_socket, m);
	//worker 线程的权重值
	static int weight[] = { 
		-1, -1, -1, -1, 0, 0, 0, 0,
		1, 1, 1, 1, 1, 1, 1, 1, 
		2, 2, 2, 2, 2, 2, 2, 2, 
		3, 3, 3, 3, 3, 3, 3, 3, };
	struct worker_parm wp[thread];
	for (i=0;i<thread;i++) {
		wp[i].m = m;
		wp[i].id = i;
		if (i < sizeof(weight)/sizeof(weight[0])) {
			wp[i].weight= weight[i];
		} else {
			wp[i].weight = 0;
		}
		create_thread(&pid[i+3], thread_worker, &wp[i]);
	}
	for (i=0;i<thread+3;i++) {
		pthread_join(pid[i], NULL); 
	}
	free_monitor(m);
}
```

在上述代码中，我们可以看出 skynet 创建线程池的流程，先创建好 monitor、socket 和 timer 这三个线程，然后创建相应数量的 worker 线程，而每个 worker 线程最终会调用 `skynet_context_message_dispatch`函数从全局消息队列中获取消息。`skynet_context_message_dispatch` 的定义如下：

```C
// skynet_start.c
struct message_queue* skynet_context_message_dispatch(struct skynet_monitor *sm, struct message_queue *q, int weight) {
	//从全局消息队列中取出一个次级消息队列
	if (q == NULL) {
		q = skynet_globalmq_pop();
		if (q==NULL)
			return NULL;
	}
	//获得该次级消息队列所对应的服务的句柄
	uint32_t handle = skynet_mq_handle(q);
	//获取服务上下文
	struct skynet_context * ctx = skynet_handle_grab(handle);
    //若取出的服务没有上下文，则重取一个新的次级消息队列
	if (ctx == NULL) {
		struct drop_t d = { handle };
		skynet_mq_release(q, drop_message, &d);
		return skynet_globalmq_pop();
	}
	int i,n=1;
	struct skynet_message msg;
    //根据不同的权重从消息队列中获得不同数量的消息
	for (i=0;i<n;i++) {
		if (skynet_mq_pop(q,&msg)) {
			skynet_context_release(ctx);
			return skynet_globalmq_pop();
		} else if (i==0 && weight >= 0) {
			n = skynet_mq_length(q);
			n >>= weight;
		}
		int overload = skynet_mq_overload(q);
		if (overload) {
			skynet_error(ctx, "May overload, message queue length = %d", overload);
		}
		skynet_monitor_trigger(sm, msg.source , handle);
		if (ctx->cb == NULL) {
			skynet_free(msg.data);
		} else {
			dispatch_message(ctx, &msg);
		}

		skynet_monitor_trigger(sm, 0,0);
	}
	assert(q == ctx->queue);
	struct message_queue *nq = skynet_globalmq_pop();
	if (nq) {
		// If global mq is not empty , push q back, and return next queue (nq)
		// Else (global mq is empty or block, don't push q back, and return q again (for next dispatch)
		skynet_globalmq_push(q);
		q = nq;
	} 
	skynet_context_release(ctx);
	return q;
}
```

结合 `strat` 和 `skynet_context_message_dispatch`，我们可以知道 skynet 的消息调度机制的全貌：当 skynet 启动时会初始化线程池，其中线程池内总共包含 4 种线程：`monitor`、`timer`、`socket` 和 `worker`，其中`worker` 具有不同的权重值。**每个 `worker` 会不断从全局消息队列中取出某个服务的次级消息队列，并根据权重值的不同从消息队列中取出若干个消息，然后调用服务所属的 callback 函数消费消息。**权重与取出的消息个数的关系如下：

> -1 ：从次级消息队列取出一个消息进行处理
> 0  ：从次级消息队列中取出所有消息进行处理
> 1  ：从次级消息队列中取出一半的消息进行处理
> 2  ：从次级消息队列中取出四分之一的消息进行处理
> 3  ：从次级消息队列中取出八分之一的消息进行处理

**这种分配优先级的做法，使得 CPU 的运转效率尽可能的高**。当线程足够多时，如果每次都只处理一个消息，虽然可以避免一些服务饿死，但却可能会使得消息队列中出现大量消息堆积。如果每次都处理一整个消息队列中的消息，则可能会使一些服务中的消息长时间得不到相应，从而导致服务饿死。为线程配置权重的做法是一个非常好的折中方案

###### 消息生产的过程
skynet 中不同的服务运行在不同的上下文当中，彼此之间的交互只能通过消息队列进行转发。不同服务之间转发消息的接口为 `skynet_send` ，其定义如下：
```C
//skynet_server.c
int skynet_send(struct skynet_context * context, uint32_t source, uint32_t destination , int type, int session, void * data, size_t sz) {
	if ((sz & MESSAGE_TYPE_MASK) != sz) {
		skynet_error(context, "The message to %x is too large", destination);
		if (type & PTYPE_TAG_DONTCOPY) {
			skynet_free(data);
		}
		return -2;
	}
    //_filter_args:根据 type 中的 PTYPE_TAG_DONTCOPY 和 PTYPE_TAG_ALLOCSESSION 位域对参数进行一些相应的处理
    // PTYPE_TAG_DONTCOPY：表示不要拷贝 data 的副本，直接在 data 上进行处理
    // PTYPE_TAG_ALLOCSESSION: 表示为消息分配一个新的 session
	_filter_args(context, type, &session, (void **)&data, &sz);
	if (source == 0) {
		source = context->handle;
	}
	if (destination == 0) {
		if (data) {
			skynet_error(context, "Destination address can't be 0");
			skynet_free(data);
			return -1;
		}
		return session;
	}
	if (skynet_harbor_message_isremote(destination)) {
		struct remote_message * rmsg = skynet_malloc(sizeof(*rmsg));
		rmsg->destination.handle = destination;
		rmsg->message = data;
		rmsg->sz = sz & MESSAGE_TYPE_MASK;
		rmsg->type = sz >> MESSAGE_TYPE_SHIFT;
		skynet_harbor_send(rmsg, source, session);
	} else {
		struct skynet_message smsg;
		smsg.source = source;
		smsg.session = session;
		smsg.data = data;
		smsg.sz = sz;

		if (skynet_context_push(destination, &smsg)) {
			skynet_free(data);
			return -1;
		}
	}
	return session;
}
```
从上述代码中，`skynet_send` 使用了 `source` 和 `destination` 来标记消息的发送端和接收端，这两个参数的本质就是能够在全网范围内唯一标识一个服务的 handle。handle 为一个 32 位无符号整数，其中高 8 位为 harbor id，用来表示服务所属的 skynet 节点，而剩余的 24 位则用于表示该 skynet 内的唯一一个服务。不管最终调用的函数是 `skynet_harbor_send` 还是 `skynet_context_push`，最后都会回归到 `skynet_mq_push` 这个函数中。**因此，skynet 中发送消息的本质就是往目标服务的次级消息队列中压入消息。**

#### 监工机制 —— monitor 线程的工作
说完了 skynet 消息调度中消息的生产与消费，我们来稍微看一看 monitor 线程(监工) 是如何监管 worker 线程的工作的。在这之前我们先看看 monitor 的定义：
```C
//skynet_start.c
struct monitor {
    int count;		//monitor 所监视的 worker 线程的数量
    struct skynet_monitor ** m;	//存放所有的 skynet_monitor 的数组，worker 和 skynet_monitor 一一对应
    pthread_cond_t cond;
    pthread_mutex_t mutex;
    int sleep;	//休眠时间
    int quit;	//退出标志
};
//skynet_monitor.c
struct skynet_monitor {
	int version;			//版本号
	int check_version;		//前一个版本号
	uint32_t source;		//源地址
	uint32_t destination;	//目标地址
};
```

如前面所提到的，当 skynet 启动线程池时，第一个创建的线程便是 monitor 线程，它的运行函数如下：

```C
//skynet_start.c
static void *thread_monitor(void *p) {
	struct monitor * m = p;
	int i;
	int n = m->count;
	skynet_initthread(THREAD_MONITOR);
	for (;;) {
		//CHECK_ABORT : if (G_NODE.total == 0) break;
		CHECK_ABORT
		for (i=0;i<n;i++) {
			skynet_monitor_check(m->m[i]);
		}
		for (i=0;i<5;i++) {
			CHECK_ABORT
			sleep(1);
		}
	}
	return NULL;
}
//skynet_monitor.c
void skynet_monitor_check(struct skynet_monitor *sm) {
    //版本号相同时
	if (sm->version == sm->check_version) {
        //若目标地址不为 0，则 sm 所对应那个 worker 可能陷入了死循环
		if (sm->destination) {
			skynet_context_endless(sm->destination);
			skynet_error(NULL, "A message from [ :%08x ] to [ :%08x ] maybe in an endless loop (version = %d)", sm->source , sm->destination, sm->version);
		}
	} else {
        //版本号不同
		sm->check_version = sm->version;
	}
}
```
monitor 的监管逻辑非常简单，每隔 5 s 便为每个 worker 线程执行一次 `skynet_monitor_check` 函数。
我们再来看看 `skynet_monitor_trigger` 函数的实现：
```C
// skynet_start.c
struct message_queue* skynet_context_message_dispatch(struct skynet_monitor *sm, struct message_queue *q, int weight) {
...
		int overload = skynet_mq_overload(q);
		if (overload) {
			skynet_error(ctx, "May overload, message queue length = %d", overload);
		}
		skynet_monitor_trigger(sm, msg.source , handle);
		if (ctx->cb == NULL) {
			skynet_free(msg.data);
		} else {
			dispatch_message(ctx, &msg);
		}
    	skynet_monitor_trigger(sm, 0,0);
...
}
//skynet_monitor.c
void skynet_monitor_trigger(struct skynet_monitor *sm, uint32_t source, uint32_t destination) {
	sm->source = source;
	sm->destination = destination;
    //递增 version
	ATOM_INC(&sm->version);
}
```
从上述代码中，我们可以看出 monitor 线程的工作原理。我们来还原一下 monitor 的工作场景：
> 1. 当一个 worker 线程(记为w)从消息队列中取出一个次级消费队列进行消费。在执行 `dispatch_message(ctx, &msg);`之前会先调用 `skynet_monitor_trigger`函数，此时对应的 skynet_monitor(记为w_sm) 有 `w_sm->version = 1`， `w_sm->check_version = 0` 成立。随后 w 进入了消息消费过程。
> 2. 此时 monitor 刚好对 w_sm 执行了 `skynet_monitor_check`函数，使得有 `w_sm->version == w_sm->check_version == 1` 成立。
> 3. 当 w 在消费过程中陷入了死循环并超过第二步 5 s 的时间后，monitor 再一次对 w_sm 执行 `skynet_monitor_check`函数。这一次 monitor 发现条件  `w_sm->version == w_sm->check_version` 成立，于是向用户返回一条错误日志。
> 4. 若 w 在第二步 5 s 的时间内完成了消息消费的过程，则会将 `w_sm->source` 和 `w_sm->destination` 都设置为 0。 这样即使 monitor 即使检测到 `w_sm->version == w_sm->check_version` 也不会产生错误日志。

#### 如何实现线程安全
在 skynet 的消息调度机制中，可能涉及到竞态问题的地方主要有往全局消息队列中执行`push`和`pop`操作、往次级消息队列中执行 `push` 和 `pop` 操作以及消息的消费过程
```C
struct message_queue * skynet_globalmq_pop() {
	struct global_queue *q = Q;
	SPIN_LOCK(q)
	struct message_queue *mq = q->head;
	if(mq) {
		q->head = mq->next;
		if(q->head == NULL) {
			assert(mq == q->tail);
			q->tail = NULL;
		}
		mq->next = NULL;
	}
	SPIN_UNLOCK(q)
	return mq;
}
void skynet_globalmq_push(struct message_queue * queue) {
	struct global_queue *q= Q;

	SPIN_LOCK(q)
	assert(queue->next == NULL);
	if(q->tail) {
		q->tail->next = queue;
		q->tail = queue;
	} else {
		q->head = q->tail = queue;
	}
	SPIN_UNLOCK(q)
}
void skynet_mq_push(struct message_queue *q, struct skynet_message *message) {
	assert(message);
	SPIN_LOCK(q)
	q->queue[q->tail] = *message;
	if (++ q->tail >= q->cap) {
		q->tail = 0;
	}
	if (q->head == q->tail) {
		expand_queue(q);
	}
	if (q->in_global == 0) {
		q->in_global = MQ_IN_GLOBAL;
		skynet_globalmq_push(q);
	}
	SPIN_UNLOCK(q)
}
int skynet_mq_pop(struct message_queue *q, struct skynet_message *message) {
	int ret = 1;
	SPIN_LOCK(q)
	if (q->head != q->tail) {
		*message = q->queue[q->head++];
		ret = 0;
		int head = q->head;
		int tail = q->tail;
		int cap = q->cap;
		if (head >= cap) {
			q->head = head = 0;
		}
		int length = tail - head;
		if (length < 0) {
			length += cap;
		}
		while (length > q->overload_threshold) {
			q->overload = length;
			q->overload_threshold *= 2;
		}
	} else {
		// reset overload_threshold when queue is empty
		q->overload_threshold = MQ_OVERLOAD;
	}
	if (ret) {
		q->in_global = 0;
	}
	SPIN_UNLOCK(q)
	return ret;
}
```
skynet 的全局消息队列会被很多的线程访问，而且同一个服务可以同时接收多个不同服务所发送来的信息，因此这两个队列的访问频率都较高，而且对这两个队列的压入和弹出操作都非常快，使用自旋锁回避互斥锁更加经济。服务的 callback 不必是线程安全的，因为每次 worker 都会从全局消息队列中将整个次级消息队列取出，因此其他线程无法同时访问到同一个次级消息队列，自然也就不会面临竞态问题。

#### 参考资料
[1]. [Skynet 设计综述 —— 云风](https://blog.codingnow.com/2012/09/the_design_of_skynet.html)
[2].[skynet源码赏析](https://manistein.github.io/blog/post/server/skynet/skynet%E6%BA%90%E7%A0%81%E8%B5%8F%E6%9E%90/)
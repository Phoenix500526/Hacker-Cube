---
title: skynet 源码阅读笔记 —— skynet 的模块与服务
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-04 17:09:27
password:
summary:
tags: [skynet框架, 动态库, C语言]
categories: skynet源码剖析
---

#### 文前导读

skynet 是一个由云风所写的轻量级在线游戏服务器框架。本文为 skynet 框架源码剖析系列的第二篇文章，探讨了 skynet 的模块加载与服务的启动功能，主要包含了以下内容：

> * 基本概念：模块与服务是什么？
> * 模块的加载
> * 服务的启动
> * 流程的回顾

<!-- more -->

#### 1.基本概念：模块与服务
**模块(module)**：在skynet中，模块是指符合规范的 C 共享库文件。一个符合规范的 C 共享库应当具备 `*_create`、`*_signal`、`*_release` 以及 `*_init` 四个接口。其中 \* 代表模块名称。其中模块的接口及定义如下：

```C
//skynet_module.h
//每一个模块都应当提供 create、init、release 以及 signal 等四个接口
typedef void * (*skynet_dl_create)(void);
typedef int (*skynet_dl_init)(void * inst, struct skynet_context *, const char * parm);
typedef void (*skynet_dl_release)(void * inst);
typedef void (*skynet_dl_signal)(void * inst, int signal);

struct skynet_module {
    const char * name; //模块名称
    void * module;     //用于访问对应so库的句柄，由dlopen函数获得
    skynet_dl_create create;
    skynet_dl_init init;
    skynet_dl_release release;
    skynet_dl_signal signal;
};
//skynet_module.c
#define MAX_MODULE_TYPE 32

//modules 列表，用于存放全部用到的 module 
struct modules {
    int count;  //存放的 module 的数量
    struct spinlock lock;
    const char * path;  //path由配置文件中的module_path提供
    struct skynet_module m[MAX_MODULE_TYPE];    //存储module的数组
};
static struct modules * M = NULL;
```
**服务(service)**：相对于模块是静态的概念，服务则是动态的概念，指的是运行在独立上下文中的模块。
skynet 提供了这样的一种机制：用户可以将自定义的模块放置到 skynet 指定的目录下。当 skynet 使用到对应的服务时，会将该模块加载到 modules 当中，并为其创建一个独立的上下文环境(context)。这样不同的服务的运行环境相互透明，交互则通过消息队列来进行。
```c
//skynet_server.c:
struct skynet_context {
    void * instance;    //调用模块的 *_create 函数创建对应的服务实例
    struct skynet_module * mod; //指向对应的模块
    void * cb_ud;   //回调函数所需参数
    skynet_cb cb;   //回调函数
    struct message_queue *queue;    //服务所属的消息队列
    FILE * logfile;     //日志文件句柄
    uint64_t cpu_cost;  // in microsec
    uint64_t cpu_start; // in microsec
    char result[32];    //存放回调函数的执行结果
    uint32_t handle;    //位于该上下文环境中的一个服务的句柄
    int session_id;     //session_id 用来将请求和响应匹配起来
    int ref;            //引用计数，当 ref == 0 时回收内存
    int message_count;  //消息队列中消息的数量？
    bool init;          //是否完成了初始化
    bool endless;       //该服务是否是一个无限循环
    bool profile;       

    CHECKCALLING_DECL
};
```

#### 2.模块的加载
在 skynet 中，模块的加载主要通过 `skynet_module_query` 函数来完成。当 skynet 启动时会先执行 `skynet_module_init` 函数对全局模块列表 modules 进行初始化。当需要使用到某个服务时，skynet 会调用 `skynet_context_new` 函数为其创建上下文，这个过程当中会调用 `skynet_module_query(name)` 函数，该函数会根据 name 查找相应的模块。如果该模块尚未被加载，则将其加载到 modules 当中。具体代码如下
```C
//skynet_module.c
//根据模块名查找对应的模块，如果找不到且 modules 中尚有空间则将模块加载进来
struct skynet_module * skynet_module_query(const char * name) {
    struct skynet_module * result = _query(name);
    if (result)
        return result;

    SPIN_LOCK(M)
    //双重检测可以避免以下情形：两个不同的服务 A 和 B 同时调用了一个服务 C，在 A 查找 C 中的模块时，B 进入自旋等待状态。
    //当 A 调用结束后会将 C 模块插入 modules 中，此时如果 B 再执行插入则会导致重复插入
    result = _query(name); // double check

    if (result == NULL && M->count < MAX_MODULE_TYPE) {
        int index = M->count;
        //返回相应动态库的句柄
        void * dl = _try_open(M,name);
        if (dl) {
            M->m[index].name = name;
            M->m[index].module = dl;

            if (open_sym(&M->m[index]) == 0) {
                M->m[index].name = skynet_strdup(name);
                M->count ++;
                result = &M->m[index];
            }
        }
    }
    SPIN_UNLOCK(M)

    return result;
}
static int open_sym(struct skynet_module *mod) {
    mod->create = get_api(mod, "_create");
    mod->init = get_api(mod, "_init");
    mod->release = get_api(mod, "_release");
    mod->signal = get_api(mod, "_signal");

    return mod->init == NULL;
}
//从动态库中找到对应的 api 并将其函数地址返回
static void* get_api(struct skynet_module *mod, const char *api_name) {
    size_t name_size = strlen(mod->name);
    size_t api_size = strlen(api_name);
    char tmp[name_size + api_size + 1];
    memcpy(tmp, mod->name, name_size);
    memcpy(tmp+name_size, api_name, api_size+1);
    char *ptr = strrchr(tmp, '.');
    if (ptr == NULL) {
        ptr = tmp;
    } else {
        ptr = ptr + 1;
    }
    return dlsym(mod->module, ptr);
}
```
从上述代码中可以看出，加载模块需要先调用 `_try_open()` 函数去打开对应的 .so 文件, 并通过 `open_sym` 函数来将对应的 api 存放到 `module` 结构体中相应的函数指针处。.so 文件中的 api 命名统一按照 "module_function" 的格式命名。

#### 3.服务的启动
skynet 中服务的创建主要通过 `skynet_context_new` 来完成，其代码定义如下：
```C
//skynet_server.c
struct skynet_context* skynet_context_new(const char * name, const char *param) {
    struct skynet_module * mod = skynet_module_query(name);

    if (mod == NULL)
        return NULL;

    void *inst = skynet_module_instance_create(mod);
    if (inst == NULL)
        return NULL;
    struct skynet_context * ctx = skynet_malloc(sizeof(*ctx));
    CHECKCALLING_INIT(ctx)

    ctx->mod = mod;
    ctx->instance = inst;
    //此处将引用置为 2 的原因是因为在 skynet_handle_register 中会将 ctx 保存起来，增加一次引用。
    //之后再将 ctx 返回给对应的变量，增加了一次引用，因此 ref = 2
    ctx->ref = 2;
    ctx->cb = NULL;
    ctx->cb_ud = NULL;
    ctx->session_id = 0;
    ctx->logfile = NULL;

    ctx->init = false;
    ctx->endless = false;

    ctx->cpu_cost = 0;
    ctx->cpu_start = 0;
    ctx->message_count = 0;
    ctx->profile = G_NODE.profile;
    // Should set to 0 first to avoid skynet_handle_retireall get an uninitialized handle
    ctx->handle = 0;    
    ctx->handle = skynet_handle_register(ctx);
    struct message_queue * queue = ctx->queue = skynet_mq_create(ctx->handle);
    // init function maybe use ctx->handle, so it must init at last
    context_inc();

    CHECKCALLING_BEGIN(ctx)
    int r = skynet_module_instance_init(mod, inst, ctx, param);
    CHECKCALLING_END(ctx)
    if (r == 0) {
        //skynet_context_release 会在 ctx->ref == 0 时回收这个 context
        struct skynet_context * ret = skynet_context_release(ctx);
        if (ret) {
            ctx->init = true;
        }
        skynet_globalmq_push(queue);
        if (ret) {
            skynet_error(ret, "LAUNCH %s %s", name, param ? param : "");
        }
        return ret;
    } else {
        skynet_error(ctx, "FAILED launch %s", name);
        uint32_t handle = ctx->handle;
        skynet_context_release(ctx);
        skynet_handle_retire(handle);
        struct drop_t d = { handle };
        skynet_mq_release(queue, drop_message, &d);
        return NULL;
    }
}
```
从上述代码中我们可以看出 `skynet_context_new` 的主要工作为如下：
> 1. 在 modules 中查找对应的模块名称，如果存在则直接返回模块的句柄，不存在则将模块加载进内存，并保存在 modules 当中
> 2. 调用 module 的 create api 创建 module 的实例 inst
> 3. 分配 skynet_context 结构体并为其赋上相应的值
> 4. 调用 module 的 init api 为 inst 进行初始化
>       如果初始化成功，则将该 context 中的次级消息队列 queue 放入到全局消息队列当中，然后返回创建好的服务(context)
>       如果失败则释放分配的 skynet_context, 为服务分配的 handle 以及专属的次级消息队列, 然后返回 NULL。

上述代码中需要注意的，`ctx->ref`的初始值为 2。这是因为当 `skynet_context_new` 执行完毕后，会有两个地方引用了新创建好的 context。一个是 `skynet_context_new` 的调用者，它会保存返回的 context 指针; 另一个则是 `skynet_handle_register` 函数，该函数会将新创建的 context 保存在 `handle_storage` 的 `slot` 字段中
接下来，我们来看看 `skynet_context_new` 中的几个模块相关的函数：`skynet_module_instance_create`、 `skynet_module_instance_init`
```C
//skynet_module.c
void* skynet_module_instance_create(struct skynet_module *m) {
    if (m->create) {
        return m->create();
    } else {
        return (void *)(intptr_t)(~0);
    }
}

int skynet_module_instance_init(struct skynet_module *m, void * inst, struct skynet_context *ctx, const char * parm) {
    return m->init(inst, ctx, parm);
}

void skynet_module_instance_release(struct skynet_module *m, void *inst) {
    if (m->release) {
        m->release(inst);
    }
}

void skynet_module_instance_signal(struct skynet_module *m, void *inst, int signal) {
    if (m->signal) {
        m->signal(inst, signal);
    }
}
```
在上述代码中，`skynet_module_instance_create` 的返回值 `(void *)(intptr_t)(~0)` 引起了我的好奇。这个地址的值为 `0xffffffff`, 代表的是内存地址的最底端的地址。它主要的作用就是为了和 `NULL` 作区分。当 skynet 调用对应模块的 `_create` 函数时, 如果此时内存耗尽，无法创建模块对象，则会返回 `NULL`。如果用户在没有定义 `_create` 函数情况下也使用 `NULL` 做返回值，则无法区分这两种情况。

#### 4.总结
简单地来讲，skynet 的模块加载与服务创建的整体过程为：
当 skynet 启动时会先执行 `skynet_module_init` 进行 modules 的创建，随后调用 `skynet_context_new` 创建新的服务。在这个过程当中， skynet 先会自动根据配置文件中指定的模块路径进行 module 的加载。完成加载后的 module 将被保存在全局的 modules 当中。随后，分配 `skynet_context` 结构体并进行相应赋值。在赋值的过程中会调用到 module 的 `_create`, `_init` 等 api。如果分配成功则将 `context` 返回给调用者，失败返回 `NULL`。创建好的服务彼此透明，运行在不同的 `skynet_context` 下，不同的服务之间的交互必须通过消息队列进行转发
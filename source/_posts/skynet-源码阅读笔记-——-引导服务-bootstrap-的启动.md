---
title: skynet 源码阅读笔记 —— 引导服务 bootstrap 的启动
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-04 17:33:14
password:
summary:
tags: [skynet框架, lua, C语言, bootstrap服务]
categories: 源码剖析
---

#### 文前导读

skynet 是一个由云风所写的轻量级在线游戏服务器框架。本文为 skynet 框架源码剖析系列的第五篇文章，探讨了 skynet 的引导服务 bootstrap 的实现，主要包含了以下内容：

> * bootstrap 如何启动，属于何种服务
> * snlua 服务的加载及初始化
> * bootstrap 的主要作用

<!-- more -->

#### 引导服务 bootstrap 的启动
skynet 在启动的初期，在 `skynet_start` 函数中创建了两个服务 `logger` 和 `bootstrap`。其中 `bootstrap` 是一个 snlua 类型的服务，主要通过 `bootstrap` 函数来实现：
```C
//syknet_start.c
void skynet_start(struct skynet_config * config) {
    ...
    skynet_handle_namehandle(skynet_context_handle(ctx), "logger");
    //config->bootstrap = "snlua bootstrap"
    bootstrap(ctx, config->bootstrap);
    ...
}
static void bootstrap(struct skynet_context * logger, const char * cmdline) {
    int sz = strlen(cmdline);
    char name[sz+1];
    char args[sz+1];
    sscanf(cmdline, "%s %s", name, args);
    //name = snlua, args = "bootstrap"
    struct skynet_context *ctx = skynet_context_new(name, args);
    if (ctx == NULL) {
        skynet_error(NULL, "Bootstrap error : %s\n", cmdline);
        skynet_context_dispatchall(logger);
        exit(1);
    }
}
```
可以看到，在上述代码中 `bootstrap` 主要的工作便是调用 `skynet_context_new` 创建了一个名为 `snlua` 的服务。我们来看下 `skynet_context_new` 的代码实现：
```C
struct skynet_context*  skynet_context_new(const char * name, const char *param) {
    //查询模块名称，查询到则直接返回模块指针，否则将其加载到全局的模块列表中
    struct skynet_module * mod = skynet_module_query(name);

    if (mod == NULL)
        return NULL;

    void *inst = skynet_module_instance_create(mod);
    if (inst == NULL)
        return NULL;
    struct skynet_context * ctx = skynet_malloc(sizeof(*ctx));

    ...     //为避免粘帖过多代码，此处省略部分 ctx 的赋值操作

    struct message_queue * queue = ctx->queue = skynet_mq_create(ctx->handle);
    // init function maybe use ctx->handle, so it must init at last
    context_inc();

    CHECKCALLING_BEGIN(ctx)
    int r = skynet_module_instance_init(mod, inst, ctx, param);
    CHECKCALLING_END(ctx)
    if (r == 0) {
        //ctx 的引用计数减 1，skynet_context_release 会在 ctx->ref == 0 时回收这个 context
        struct skynet_context * ret = skynet_context_release(ctx);
        if (ret) {
            ctx->init = true;
        }
        //将次级消息队列放入全局消息队列中
        skynet_globalmq_push(queue);
        if (ret) {
            skynet_error(ret, "LAUNCH %s %s", name, param ? param : "");
        }
        return ret;
    } else {
        ... //错误处理，包括释放已分配的ctx、次级消息队列等
        return NULL;
    }
}
```


#### snlua 的加载及初始化
从前面 `skynet_context_new` 函数中，我们可以看出 `snlua` 服务的启动需要利用到 `skynet_module_instance_create`函数进行实例的创建，利用`skynet_module_instance_init`函数进行初始化，这两个函数最终会调用到对应模块中的 \*\_create 和 \*\_init 函数。对于 snlua 模块而言，其对应代码保存在 service-src/service_snlua.c 文件中，最终会编译成为 snlua.so 文件。由于在前面的文章 [skynet 源码阅读笔记 —— skynet 的模块与服务](https://www.jianshu.com/p/de2d10867aa6) 中已经说明了模块加载的详细方式，因此这里不多着笔墨说明。我们先来看看 snlua 的基本数据结构，然后直接看相关的模块函数
```C
//service-src/service_snlua.c
//内存阈值，当 snlua 占用的内存超过阈值则触发警报
#define MEMORY_WARNING_REPORT (1024 * 1024 * 32)
//snlua 的基本数据结构
struct snlua {
    lua_State * L;                  //每个 snlua 模块都配备了专属的 lua 环境
                                    //不同的 snlua 服务将不同的 lua 脚本运行在自己的 lua 环境中，彼此之间互不影响
    struct skynet_context * ctx;    //模块所属的服务
    size_t mem;
    size_t mem_report;              //内存阈值
    size_t mem_limit;
};

struct snlua* snlua_create(void) {
    struct snlua * l = skynet_malloc(sizeof(*l));
    memset(l,0,sizeof(*l));
    l->mem_report = MEMORY_WARNING_REPORT;
    l->mem_limit = 0;
    l->L = lua_newstate(lalloc, l);
    return l;
}

int snlua_init(struct snlua *l, struct skynet_context *ctx, const char * args) {
    int sz = strlen(args);
    char * tmp = skynet_malloc(sz);
    memcpy(tmp, args, sz);
    //将 launch_cb 设置为 snlua 服务的回调函数，参数为 l
    skynet_callback(ctx, l , launch_cb);
    const char * self = skynet_command(ctx, "REG", NULL);
    //self 的值为 :handle
    uint32_t handle_id = strtoul(self+1, NULL, 16);
    // it must be first message
    //向自己发送第一条消息，这条消息将由 launch_cb 进行处理，消息内容为 "bootstrap"
    skynet_send(ctx, 0, handle_id, PTYPE_TAG_DONTCOPY,0, tmp, sz);
    return 0;
}
```
从上述代码中可知，`snlua_create` 会负责初始化 `snlua` 结构体，并将其返回，而 `snlua_init` 函数则负责将创建好的 snlua 服务的回调函数设置为 `launch_cb` 函数，并对其发送一个注册命令，完成后向 snlua 服务的次级消息队列发送一条消息。
skynet 为每个模块都提供了一组相应的命令，其对应的数据类型为 `command_func`，skynet 为模块所提供的所有命令都存放在了 `cmd_funcs` 数组当中
```C
//skynet_service.c
struct command_func {
    const char *name;   //命令名称
    const char * (*func)(struct skynet_context * context, const char * param);  //命令对应的回调函数
};
static struct command_func cmd_funcs[] = {
    { "TIMEOUT", cmd_timeout },
    { "REG", cmd_reg },
    { "QUERY", cmd_query },
    { "NAME", cmd_name },
    { "EXIT", cmd_exit },
    { "KILL", cmd_kill },
    { "LAUNCH", cmd_launch },
    { "GETENV", cmd_getenv },
    { "SETENV", cmd_setenv },
    { "STARTTIME", cmd_starttime },
    { "ABORT", cmd_abort },
    { "MONITOR", cmd_monitor },
    { "STAT", cmd_stat },
    { "LOGON", cmd_logon },
    { "LOGOFF", cmd_logoff },
    { "SIGNAL", cmd_signal },
    { NULL, NULL },
};
```
在了解了 `command_func` 的定义后，我们来看看 `skynet_command` 函数的实现
```C
//查找相应的命令，并返回命令函数的执行结果
//snlua 对 skynet_command 的调用形式为 skynet_command(ctx, "REG", NULL)
const char* skynet_command(struct skynet_context * context, const char * cmd , const char * param) {
    struct command_func * method = &cmd_funcs[0];
    while(method->name) {
        if (strcmp(cmd, method->name) == 0) {
            return method->func(context, param);
        }
        ++method;
    }
    return NULL;
}
//cmd_reg(ctx, NULL)
static const char* cmd_reg(struct skynet_context * context, const char * param) {
    if (param == NULL || param[0] == '\0') {
        //将回调函数的执行结果和 handle 拼接在一起，并返回
        sprintf(context->result, ":%x", context->handle);
        //context->result 是用来存放 context->cb 的执行结果的
        return context->result;
    } else if (param[0] == '.') {
        return skynet_handle_namehandle(context->handle, param + 1);
    } else {
        skynet_error(context, "Can't register global name %s in C", param);
        return NULL;
    }
}
```
从上述代码中，我们可以简单地总结一下 snlua 的启动流程：
> 1. skynet 调用 bootstrap 函数创建了一个 snlua 服务
> 2. 在 bootstrap 创建服务的过程中，会先从全局的 modules 中查找 snlua 模块是否已加载，如果没有则加载到内存当中。
> 3. 加载完毕后，先调用 snlua_create 函数分配一个 snlua 结构体，该结构体中包含了一个独立的 lua 运行状态，用于执行相应的 lua 脚本
> 4. 创建好对应的 snlua 模块实例后，执行 snlua_init 函数为其进行初始化。初始化的过程中负责设置服务的回调函数，并向 snlua 服务发送一个注册命令，随后向 snlua 服务发送一条消息
> 5. 将 snlua 的消息队列压入全局的消息队列当中
> 完成上述的 5 个步骤后，一个 snlua 服务就算是启动起来了。

#### bootstrap 服务的主要工作
在前面的内容当中，我们看到了 snlua 模块在初始化的过程当中会向自己发送一条消息，这样做的目的是为了自身的服务启动起来。因为在 skynet 当中，服务要依靠消息来驱动。snlua 在初始化过程当中向自身发送了一条消息，当 snlua 服务创建完毕后，worker 线程便会消息队列当中取出消息并执行相应的回调函数 `launch_cb` 函数进行消费，这样就能够将 snlua 服务运转起来。我们来看一下 `launch_cb` 的实现：
```C
//msg 的值为 bootstrap
static int launch_cb(struct skynet_context * context, void *ud, int type, int session, uint32_t source , const void * msg, size_t sz) {
    assert(type == 0 && session == 0);
    struct snlua *l = ud;
    //重设回调函数
    skynet_callback(context, NULL, NULL);
    int err = init_cb(l, context, msg, sz);
    if (err) {
        skynet_command(context, "EXIT", NULL);
    }

    return 0;
}

static int init_cb(struct snlua *l, struct skynet_context *ctx, const char * args, size_t sz) {
    lua_State *L = l->L;
    l->ctx = ctx;
    //暂停 lua 的 GC 机制
    lua_gc(L, LUA_GCSTOP, 0);
    lua_pushboolean(L, 1);  /* signal for libraries to ignore env. vars. */
    lua_setfield(L, LUA_REGISTRYINDEX, "LUA_NOENV");
    luaL_openlibs(L);
    lua_pushlightuserdata(L, ctx);
    lua_setfield(L, LUA_REGISTRYINDEX, "skynet_context");
    //判断 skynet.codecache 是否为与 package.loaded 当中。如果不在则调用 codecache 进行加载
    luaL_requiref(L, "skynet.codecache", codecache , 0);
    lua_pop(L,1);
    //设置相关的全局变量
    const char *path = optstring(ctx, "lua_path","./lualib/?.lua;./lualib/?/init.lua");
    lua_pushstring(L, path);
    lua_setglobal(L, "LUA_PATH");
    const char *cpath = optstring(ctx, "lua_cpath","./luaclib/?.so");
    lua_pushstring(L, cpath);
    lua_setglobal(L, "LUA_CPATH");
    const char *service = optstring(ctx, "luaservice", "./service/?.lua");
    lua_pushstring(L, service);
    lua_setglobal(L, "LUA_SERVICE");
    const char *preload = skynet_command(ctx, "GETENV", "preload");
    lua_pushstring(L, preload);
    lua_setglobal(L, "LUA_PRELOAD");
    //traceback 将 L 栈的回溯信息压入栈
    lua_pushcfunction(L, traceback);
    assert(lua_gettop(L) == 1);
    //lua 服务的加载器为 loader.lua
    const char * loader = optstring(ctx, "lualoader", "./lualib/loader.lua");

    int r = luaL_loadfile(L,loader);
    if (r != LUA_OK) {
        skynet_error(ctx, "Can't load %s : %s", loader, lua_tostring(L, -1));
        report_launcher_error(ctx);
        return 1;
    }
    //args = bootstrap
    lua_pushlstring(L, args, sz);
    //利用 loader 将 bootstrap.lua 脚本执行起来。
    r = lua_pcall(L,1,0,1);
    if (r != LUA_OK) {
        skynet_error(ctx, "lua loader error : %s", lua_tostring(L, -1));
        report_launcher_error(ctx);
        return 1;
    }
    lua_settop(L,0);
    if (lua_getfield(L, LUA_REGISTRYINDEX, "memlimit") == LUA_TNUMBER) {
        size_t limit = lua_tointeger(L, -1);
        l->mem_limit = limit;
        skynet_error(ctx, "Set memory limit to %.2f M", (float)limit / (1024 * 1024));
        lua_pushnil(L);
        lua_setfield(L, LUA_REGISTRYINDEX, "memlimit");
    }
    lua_pop(L, 1);
    //重启 lua 的 GC 机制
    lua_gc(L, LUA_GCRESTART, 0);
    return 0;
}
static int codecache(lua_State *L) {
    luaL_Reg l[] = {
        { "clear", cleardummy },
        { "mode", cleardummy },
        { NULL, NULL },
    };
    luaL_newlib(L,l);
    lua_getglobal(L, "loadfile");
    lua_setfield(L, -2, "loadfile");
    return 1;
}
```
从上述代码中可以看出，bootstrap服务(即前面的 snlua 服务)在触发时，会调用 `init_cb` 来代替 `lauch_cb` 函数。简单地来说，`init_cb` 中最主要的部分便是设置相应的环境变量以及加载器loader。其中，环境变量的意义如下：
> LUA_PATH：Lua搜索路径，在config.lua_path指定。
> LUA_CPATH：C模块的搜索路径，在config.lua_cpath指定。
> LUA_SERVICE：Lua服务的搜索路径，在config.luaservice指定。
> LUA_PRELOAD：预加载脚本，这些脚本会在所有服务开始之前执行，可以用它来初始化一些全局的设置。

在设置好相应的环境变量后，`init_cb` 会执行 loader.lua，并将 bootstrap.lua 传进去。loader.lua 的主要作用是对环境变量以及传入的参数进行一些文本处理，然后找到对应的文件去执行，这里的参数主要是指 `bootstrap`，最终会执行 /service/bootstrap.lua 文件。其中 bootstrap.lua 的源码如下：
```lua
--将 skynet.lua 中定义的函数引用到当前文件
local skynet = require "skynet"
local harbor = require "skynet.harbor"
require "skynet.manager"    -- import skynet.launch, ...
skynet.start(function()
    local standalone = skynet.getenv "standalone"
    --利用 skynet.launch 启动一个 launcher
    local launcher = assert(skynet.launch("snlua","launcher"))
    skynet.name(".launcher", launcher)
    --确认当前的 skynet 节点是主节点还是从节点
    local harbor_id = tonumber(skynet.getenv "harbor" or 0)
    if harbor_id == 0 then
        assert(standalone ==  nil)
        standalone = true
        skynet.setenv("standalone", "true")

        local ok, slave = pcall(skynet.newservice, "cdummy")
        if not ok then
            skynet.abort()
        end
        skynet.name(".cslave", slave)

    else
        if standalone then
            if not pcall(skynet.newservice,"cmaster") then
                skynet.abort()
            end
        end

        local ok, slave = pcall(skynet.newservice, "cslave")
        if not ok then
            skynet.abort()
        end
        skynet.name(".cslave", slave)
    end

    if standalone then
        local datacenter = skynet.newservice "datacenterd"
        skynet.name("DATACENTER", datacenter)
    end
    skynet.newservice "service_mgr"
    pcall(skynet.newservice,skynet.getenv "start" or "main")
    skynet.exit()
end)
```
从上述 lua 代码中，我们可以看出 bootstrap.lua 的主要工作如下：
> 1. 启动`launcher`服务，这个服务是一个通用的服务启动器，如果我们需要在lua创建一个 C 服务就需要用到它
> 2. 启动`datacenterd`服务
> 3. 启动`service_mgr`服务
> 4. 根据 config 中的 start 字段，指定相应的 lua 脚本，在 bootstrap 服务中启动的是 main.lua 脚本

到目前为止，bootstrap 服务的基本内容大概就说完了，而相关的一些其他一部分未说明清楚的部分(如main.lua, skynet.newservice, skynet_launch 等)则留在其他文章中讨论
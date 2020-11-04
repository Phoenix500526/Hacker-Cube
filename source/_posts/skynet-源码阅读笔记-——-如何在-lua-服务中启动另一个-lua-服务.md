---
title: skynet 源码阅读笔记 —— 如何在 lua 服务中启动另一个 lua 服务
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-04 20:11:37
password:
summary:
tags: [源码剖析, skynet框架, lua, C语言]
categories: skynet源码剖析
---

#### 文前导读

skynet 是一个由云风所写的轻量级在线游戏服务器框架。本文为 skynet 框架源码剖析系列的第六篇文章，探讨了 skynet 的 lua 服务之间彼此之间是如何启动的，主要包含了以下内容：

> * 如何 launch 一个 lua 服务
> * 如何在 lua 服务中创建另一个 lua 服务

<!-- more -->

在上一篇文章中[《skynet 源码阅读笔记 —— 引导服务 bootstrap 的启动》](http://hacker-cube.com/2020/11/04/skynet-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0-%E2%80%94%E2%80%94-%E5%BC%95%E5%AF%BC%E6%9C%8D%E5%8A%A1-bootstrap-%E7%9A%84%E5%90%AF%E5%8A%A8/)，我们探讨了 bootstrap 服务的启动细节，其中 bootstrap 服务的核心在于 bootstrap.lua 脚本的执行。而这篇博客会借助 bootstrap.lua 脚本中的部分内容来说明如何在一个 lua 服务内启动其他的 lua 服务
```lua
--引用 skynet.lua 中的接口
local skynet = require "skynet"
local harbor = require "skynet.harbor"
require "skynet.manager"    -- import skynet.launch, ...

skynet.start(function()
    local standalone = skynet.getenv "standalone"

    local launcher = assert(skynet.launch("snlua","launcher"))
    skynet.name(".launcher", launcher)
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
在上述代码中，我们可以看到 bootstrap.lua 在文件的最开始处，执行了 `local skynet = require "skynet"` 以及 `require "skynet.manager"`, 这都是为了要在 bootstrap.lua 文件中，引用 skynet 为 lua 服务所设计的 api，对应文件及 api 如下：
> lualib/skynet.lua:  `skynet.start` 和 `skynet.newservice`
> lualib/skynet/manager.lua: `skynet.launch` 和 `skynet.name`

#### skynet.launch 及 skynet.name 的作用

对于 `skynet.start`函数我们放到后面讨论，这里先分析 `skynet.launch` 以及 `skynet.name` 两个函数，这两个函数定义如下：
```lua
--manager.lua
--bootstrap 中是这样调用 skynet.launch 函数的：skynet.launch("snlua","launcher")
function skynet.launch(...)
    --相当于执行 c.comand("LAUNCH", "snlua laucher")，
    local addr = c.command("LAUNCH", table.concat({...}," "))
    if addr then
        return tonumber("0x" .. string.sub(addr , 2))
    end
end
```
这里简单地说明一下 c 的意义，c 是定义在 skynet.lua 中的一个变量，其中保存了一张表。这张表可以由函数`luaopen_skynet_core` 创建。在这张表中定义了一个命令接口 command，对应的实现如下：
```C
//lua-skynet.c
static int lcommand(lua_State *L) {
    struct skynet_context * context = lua_touserdata(L, lua_upvalueindex(1));
    const char * cmd = luaL_checkstring(L,1);
    const char * result;
    const char * parm = NULL;
    if (lua_gettop(L) == 2) {
        parm = luaL_checkstring(L,2);
    }
    result = skynet_command(context, cmd, parm);
    if (result) {
        lua_pushstring(L, result);
        return 1;
    }
    return 0;
}
//skynet_service.c
static const char * cmd_launch(struct skynet_context * context, const char * param) {
    size_t sz = strlen(param);
    char tmp[sz+1];
    strcpy(tmp,param);
    char * args = tmp;
    char * mod = strsep(&args, " \t\r\n");
    args = strsep(&args, "\r\n");
    struct skynet_context * inst = skynet_context_new(mod,args);
    if (inst == NULL) {
        return NULL;
    } else {
        id_to_hex(context->result, inst->handle);
        return context->result;
    }
}
```
`lcommand` 函数的主要工作便是将对应的命令和参数转发回 C 层的 `cmd_launch` 函数中，这个函数最终会创建一个新的 snlua 类型的 C 服务 inst。而在创建这个 snlua 服务的过程中也会对其进行初始化，这个过程可见前一篇文章中所提到的 bootstrap 服务的创建及初始化，这里就不再赘述。

```lua
function skynet.name(name, handle)
    if not globalname(name, handle) then
        c.command("NAME", name .. " " .. skynet.address(handle))
    end
end
```
`skynet.name` 函数也会调用 `c.command` 接口来向对应的服务发送命令，只不过这次发送的是 NAME 命令，并且最终会调用 `cmd_name`函数来为服务进行命名。



#### 如何在 lua 服务中创建一个新的 lua 服务

在说完上面两个 api 后，我们再来看看 `skynet.newservice` 的作用。skynet 在 lua 层一共有两种不同的创建服务的方式：一种是 `skynet.launch` 创建用 C 编写的服务，而另一种方式则是调用 `skynet.newservice` 创建 lua 服务。以上述的 bootstrap 服务和 service_mgr 服务为例，创建 lua 服务的流程大致如下：
> 1.在 bootstrap 的 `start_func` 中执行 `skynet.newservice "service_mgr"`,此时 bootstrap 服务陷入阻塞状态;
> 2.在 service_mgr 服务被创建出来以后，执行 service_mgr.lua 这个脚本，在这个脚本中会执行 `skynet.start` 函数，表示 service_mgr 服务正式启动，能够正常地接收消息;
> 3.service_mgr 的 `skynet.start` 返回，bootstrap 服务的`skynet.newservice`函数返回，并获得了 service_mgr 服务的句柄

了解了这个基本过程后，让我们来看看 `skynet.newservice` 是如何定义的：
```lua
function skynet.newservice(name, ...)
    return skynet.call(".launcher", "lua" , "LAUNCH", "snlua", name, ...)
end
```
在上述代码中，bootstrap服务的`skynet.newservice`向launcher服务发送了一条命令，并阻塞等待launcher的返回执行结果。这条命令会传递到 launcher.lua中，并最终调用command.LAUNCH，进而调用launch_service：
```lua
function command.LAUNCH(_, service, ...)
    launch_service(service, ...)
    return NORET
end
local function launch_service(service, ...)
    local param = table.concat({...}, " ")
    --创建一个 lua 服务并获得该服务的句柄
    local inst = skynet.launch(service, param)
    local session = skynet.context()
    --调用 skynet.response() 获得一个 response 闭包
    local response = skynet.response()
    if inst then
        --将服务句柄和服务的命令形式以键值对的形式保存
        services[inst] = service .. " " .. param
        --保存闭包，这个 response 闭包最终会等 skynet.start 返回后再调用
        instance[inst] = response
        launch_session[inst] = session
    else
        response(false)
        return
    end
    return inst
end
```
在上述代码中，`launch_service` 在创建 service_mgr 服务后会调用相应的 service_mgr.lua 脚本。在对应的脚本中有一个 skynet.start 函数，其对应实现如下：
```lua
--skynet.lua
function skynet.start(start_func)
    --将对应服务的回调函数设置为 skynet.dispatch_message
    c.callback(skynet.dispatch_message)
    --执行服务脚本中传入的 start_func 函数
    init_thread = skynet.timeout(0, function()
        skynet.init_service(start_func)
        init_thread = nil
    end)
end
function skynet.init_service(start)
    local ok, err = skynet.pcall(start)
    if not ok then
        skynet.error("init service failed: " .. tostring(err))
        skynet.send(".launcher","lua", "ERROR")
        skynet.exit()
    else
        skynet.send(".launcher","lua", "LAUNCHOK")
    end
end
```
在上一篇文章中，我们提到了 snlua 模块在调用 `launch_cb` 函数时会执行 `skynet_callback(context, NULL, NULL);` 将回调函数置为 NULL，而在 skynet.start 函数中才将对应服务的回调函数置为 `skynet.dispatch_message`,然后调用 `skynet.init_service(start_func)`对服务进行初始化。而 `skynet.init_service(start_func)` 则会调用 start_func 函数完成对服务真正意义上的初始化，并根据初始化的结果向 launcher 发送成功或失败的消息。以下分别讨论：
* 当初始化结果成功时，服务会向 launcher 发送 LAUNCHOK 的命令，这会触发 `comand.LAUNCHOK` 的执行,其中 `command.LAUNCHOK` 的定义如下：
```lua
--launcher.lua
function command.LAUNCHOK(address)
    -- init notice
    local response = instance[address]
    if response then
        response(true, address)
        instance[address] = nil
        launch_session[address] = nil
    end

    return NORET
end
```
从上述代码中可以看出，在执行初始化成功后，launcher会将之前调用 `launch_service` 时保存的闭包取出来执行，传入的第一个参数为 true 表示初始化成功。

* 当初始化结果失败时，服务会向 launcher 发送 ERROR 的命令。
```lua
--launcher.lua
function command.ERROR(address)
    -- see serivce-src/service_lua.c
    -- init failed
    local response = instance[address]
    if response then
        response(false)
        launch_session[address] = nil
        instance[address] = nil
    end
    services[address] = nil
    return NORET
end
```
与前面 `command.LAUNCHOK`类似，`command.ERROR`会取出对应的 response 闭包并执行，传入参数为 false 表示初始化失败。随后当`skynet.send`返回后，调用 `skynet.exit` 函数移除初始化失败的服务。

```lua
--skynet.lua
function skynet.exit()
    fork_queue = {} -- no fork coroutine can be execute after skynet.exit
    skynet.send(".launcher","lua","REMOVE",skynet.self(), false)
    -- report the sources that call me
    for co, session in pairs(session_coroutine_id) do
        local address = session_coroutine_address[co]
        if session~=0 and address then
            c.send(address, skynet.PTYPE_ERROR, session, "")
        end
    end
    for resp in pairs(unresponse) do
        resp(false)
    end
    -- report the sources I call but haven't return
    local tmp = {}
    for session, address in pairs(watching_session) do
        tmp[address] = true
    end
    for address in pairs(tmp) do
        c.send(address, skynet.PTYPE_ERROR, 0, "")
    end
    c.command("EXIT")
    -- 退出服务后让出处理机权限
    coroutine_yield "QUIT"
end
--launcher.lua
function command.REMOVE(_, handle, kill)
    services[handle] = nil
    local response = instance[handle]
    if response then
        -- instance is dead
        response(not kill)  -- return nil to caller of newservice, when kill == false
        instance[handle] = nil
        launch_session[handle] = nil
    end

    -- don't return (skynet.ret) because the handle may exit
    return NORET
end
```
在执行 `skynet.exit`的过程中，会向 launcher 发送 REMOVE 命令，而这个命令最终会调用 `command.REMOVE` 函数。`command.REMOVE`会取出相应闭包，并判断该闭包是否已经被执行过。这代表了两种情况：一种是因为初始化出错而导致触发了 `command.ERROR`,这个过程中执行了 response 闭包；另一种就是服务自己调用了 `skynet.exit()` 自行退出，此时 response 闭包还没有被执行过。

当 service_mgr 服务的`skynet.start` 函数返回后，bootstrap 服务也重新进入运行状态，继续启动其他的服务(比如 main 服务)，整体的过程与启动 service_mgr 是相同的。
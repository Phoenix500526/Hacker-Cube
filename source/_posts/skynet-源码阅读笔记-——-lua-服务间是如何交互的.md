---
title: skynet 源码阅读笔记 —— lua 服务间是如何交互的
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-04 20:21:24
password:
summary:
tags: [skynet框架, C语言]
categories: 源码剖析
---



#### 文前导读

skynet 是一个由云风所写的轻量级在线游戏服务器框架。本文为 skynet 框架源码剖析系列的第七篇文章，探讨了 skynet 框架下，同一 skynet 节点内不同的lua 服务之间是如何通过消息来进行交互，主要包含了以下内容：

> * lua 服务的消息协议
> * lua 服务如何注册自己的消息及对应的回调函数
> * lua 服务是如何接受消息的？
> * lua 服务是如何发送消息的？

<!-- more -->

#### lua 服务的消息协议
skynet 使用 proto 来描述不同的消息协议。在最开始的时候，proto 是一个空表，需要由 `skynet.register_protocol` 进行消息协议的注册。skynet 在启动 lua 服务的初期会默认注册 lua，response 以及 error 类型的消息协议，这个过程通常在 `require "skynet"`语句中执行。`skynet.register_protocol`函数如下：
```lua
function skynet.register_protocol(class)
    local name = class.name
    local id = class.id
    assert(proto[name] == nil and proto[id] == nil)
    assert(type(name) == "string" and type(id) == "number" and id >=0 and id <=255)
    proto[name] = class
    proto[id] = class
end
do
    local REG = skynet.register_protocol
    --注册不同的消息类型，有普通的 lua 消息，响应消息以及错误消息
    REG {
        name = "lua",
        id = skynet.PTYPE_LUA,
        pack = skynet.pack,
        unpack = skynet.unpack,
    }

    REG {
        name = "response",
        id = skynet.PTYPE_RESPONSE,
    }

    REG {
        name = "error",
        id = skynet.PTYPE_ERROR,
        unpack = function(...) return ... end,
        dispatch = _error_dispatch,
    }
end
```
从 skynet 默认注册的消息类型来推断，我们知道一个消息协议应当包含有以下的一些字段：
* name:表明了该消息协议的类型名称
* id:表明该消息协议的类型编号，包括了以下几种不同的类型
    ```lua
    local skynet = {
        -- read skynet.h
        PTYPE_TEXT = 0,     --文本类型
        PTYPE_RESPONSE = 1, --响应消息
        PTYPE_MULTICAST = 2,--组播消息
        PTYPE_CLIENT = 3,
        PTYPE_SYSTEM = 4,
        PTYPE_HARBOR = 5,
        PTYPE_SOCKET = 6,
        PTYPE_ERROR = 7,    --错误消息
        PTYPE_QUEUE = 8,    -- used in deprecated mqueue, use skynet.queue instead
        PTYPE_DEBUG = 9,
        PTYPE_LUA = 10,     --lua 服务类型的消息
        PTYPE_SNAX = 11,
        PTYPE_TRACE = 12,   -- use for debug trace
    }
    ```
* pack:发送消息时所用到的打包函数
* unpack:接收消息时调用的解包函数
* dispatch:由消息提供方指定对应类型消息的处理函数，如果没有指定，则最终会调用 `skynet.dispatch(typename, func)`
 函数来处理

说完基本的消息协议，我们来看看 skynet 定义的三种不同类型的消息都有什么作用：
> 1. lua 型消息：采用 skynet.pack 和 skynet.unpack 进行消息的打包和解包, 默认调用`skynet.dispatch(typename, func)`进行消息的派发
> 2. response 型消息：response 消息主要用于处理skynet.call调用和定时器的返回。当源服务向目的服务发送请求，会附带一个 session，目的服务在处理完请求后，会将 session 加入 response 消息中一起通过 `skynet.ret` 返回给源服务
> 3. error 型消息：当调用 `skynet.call` 发送错误消息时，源服务可以接收到一个 error 类型的消息

#### lua 服务如何注册自己的消息及对应的回调函数
讲完了 lua 服务的消息服务的定义，我们以 example/simplemonitor.lua 中的服务来说明一下，lua 服务之间是如何相互收发信息的。而在这之前，我们需要看看 simplemonitor.lua 定义：
```lua
local skynet = require "skynet"
-- It's a simple service exit monitor, you can do something more when a service exit.
local service_map = {}

skynet.register_protocol {
    name = "client",
    id = skynet.PTYPE_CLIENT,   -- PTYPE_CLIENT = 3
    unpack = function() end,
    dispatch = function(_, address)
        local w = service_map[address]
        if w then
            for watcher in pairs(w) do
                skynet.redirect(watcher, address, "error", 0, "")
            end
            service_map[address] = false
        end
    end
}

local function monitor(session, watcher, command, service)
    assert(command, "WATCH")
    local w = service_map[service]
    if not w then
        if w == false then
            skynet.ret(skynet.pack(false))
            return
        end
        w = {}
        service_map[service] = w
    end
    w[watcher] = true
    skynet.ret(skynet.pack(true))
end

skynet.start(function()
    skynet.dispatch("lua", monitor)
end)
```
如以往的文章所提到的那样，当使用 `skynet.newservice` 函数启动一个新的 lua 服务时，会执行相应的脚本来完成服务的初始化。在 simplemonitor.lua 脚本中，先执行了 `require "skynet"`，这不仅会将相应的函数导入到当前 lua 脚本当中，还会执行 `skynet.register_protocol`为 simplemonitor 注册三种默认消息协议。随后，simplemonitor.lua 又调用了 `skynet.register_protocol` 注册了一个 client 类型的 lua 消息协议，并指定了对应的 dispatch 函数。随后调用 `skynet.start` 来启动 simplemonitor 服务。在上一篇文章[《skynet 源码阅读笔记 —— 如何在 lua 服务中启动另一个 lua 服务》](https://www.jianshu.com/p/bc37152b6413) 中提到了 `skynet.start` 会将 simplemonitor 服务的消息回调函数设置为 `skynet.dispatch_message`,然后执行 `skynet.dipatch("lua", monitor)`进行服务的初始化。

#### lua 服务是如何接受消息的？
讨论完 lua 服务是如何注册自己的消息类型及定义消息对应的回调函数后，我们来看看 lua 服务是如何接受消息的。我们先来看看 `skynet.dispatch` 函数的实现：
```lua
--skynet.lua
--simplemonitor 的调用形式为 skynet.dispatch("lua", monitor)
function skynet.dispatch(typename, func)
    --取出 lua 型消息对应的协议
    local p = proto[typename]
    if func then
        local ret = p.dispatch
        --将对应的 dispatch 函数设置为 monitor
        p.dispatch = func
        --返回原来的 dispatch 函数
        return ret
    else
        return p and p.dispatch
    end
end
```
从上述代码可以看出，当 simplemonitor 服务启动完毕后，对应的 lua 消息协议的 dispatch 函数实际上就是 `monitor` 函数。接着，我们再来看看 `skynet.dispatch_message`
```lua
function skynet.dispatch_message(...)
    --调用 raw_dispatch_message 进行消息的转发
    local succ, err = pcall(raw_dispatch_message,...)
    while true do
        local co = tremove(fork_queue,1)
        if co == nil then
            break
        end
        local fork_succ, fork_err = pcall(suspend,co,coroutine_resume(co))
        if not fork_succ then
            if succ then
                succ = false
                err = tostring(fork_err)
            else
                err = tostring(err) .. "\n" .. tostring(fork_err)
            end
        end
    end
    assert(succ, tostring(err))
end
local function raw_dispatch_message(prototype, msg, sz, session, source)
    if prototype == 1 then
        ... --prototype == 1代表响应消息类型
    else
        --取出相应的消息协议
        local p = proto[prototype]
        if p == nil then
            ...    --若 p == nil 则调用 c.send 发送一个 ERROR 型消息
        end

        local f = p.dispatch
        if f then
            -- co_create 会从协程池中获取一个空的协程，如果没有则创建一个新的协程，并将 dispatch 函数交给这个协程去执行。
            local co = co_create(f)
            session_coroutine_id[co] = session
            session_coroutine_address[co] = source
            local traceflag = p.trace
            if traceflag == false then
                -- force off
                trace_source[source] = nil
                session_coroutine_tracetag[co] = false
            else
                local tag = trace_source[source]
                if tag then
                    trace_source[source] = nil
                    c.trace(tag, "request")
                    session_coroutine_tracetag[co] = tag
                elseif traceflag then
                    -- set running_thread for trace
                    running_thread = co
                    skynet.trace()
                end
            end
            --启动并执行协程，将协程执行的结果返回给suspend函数，suspend 会根据这个结果执行相应的操作
            suspend(co, coroutine_resume(co, session,source, p.unpack(msg,sz)))
        else
            trace_source[source] = nil
            if session ~= 0 then
                c.send(source, skynet.PTYPE_ERROR, session, "")
            else
                unknown_request(session, source, msg, sz, proto[prototype].name)
            end
        end
    end
end
```
结合上述带注释的代码，我们描述一下整体的过程：当服务 A 向 simplemonitor 发送一条消息时，会将这条消息放入到 simplemonitor 对应的 snlua 服务所属的次级消息队列当中(skynet当中有多个 snlua 类型的服务，分别对应不同的 lua 服务)。worker 线程会将其取出并消费，在消费的过程当中会调用该消息所指定的 callback 函数。而 `skynet.start` 已经通过 `c.callback(skynet.dispatch_message)` 将 simplemonitor 的消息的回调函数设置为 `skynet.dispatch_message`。此时，worker线程最终就会调用到 `raw_dispatch_message`函数。这个函数会获得一个新的空的协程来执行消息协议中指定的 dispatch 函数。对应协程一旦执行起来完毕，会调用 `coroutine_yield` 函数将自身挂起，并返回挂起的原因。`suspend`会根据这个原因做不同的处理

#### lua 服务是如何发送消息的？
讲完了当 simplemonitor 收到消息的行为，我们再来看看发送消息的行为。假设现在有一个服务 A 需要向另一个服务 B 发送一条消息，那么他需要调用 `skynet.send` 函数。我们来看看 `skynet.send` 函数的定义：
```lua
function skynet.send(addr, typename, ...)
    local p = proto[typename]
    return c.send(addr, p.id, 0 , p.pack(...))
end
```
`skynet.send`会调用 `c.send(addr, p.id, 0 , p.pack(...))` 函数来发送消息，其中 `c.send` 函数的参数从左至右分别是目标地址，消息协议类型，session ID，自定义参数列表。
我们再来看看 `c.send` 所对应的函数 `lsend` 是如何实现的：
```C
//lua-skynet.c
static int lsend(lua_State *L) {
    return send_message(L, 0, 2);
}
static int send_message(lua_State *L, int source, int idx_type) {
    struct skynet_context * context = lua_touserdata(L, lua_upvalueindex(1));
    //获得目的地址 addr
    uint32_t dest = (uint32_t)lua_tointeger(L, 1);
    const char * dest_string = NULL;
    if (dest == 0) {
        if (lua_type(L,1) == LUA_TNUMBER) {
            return luaL_error(L, "Invalid service address 0");
        }
        dest_string = get_dest_string(L, 1);
    }
    int type = luaL_checkinteger(L, idx_type+0);
    int session = 0;
    //如果没有设置 session，则最后分配一个 ssession
    if (lua_isnil(L,idx_type+1)) {
        type |= PTYPE_TAG_ALLOCSESSION;
    } else {
        session = luaL_checkinteger(L,idx_type+1);
    }

    int mtype = lua_type(L,idx_type+2);
    switch (mtype) {
    case LUA_TSTRING: {
        size_t len = 0;
        void * msg = (void *)lua_tolstring(L,idx_type+2,&len);
        if (len == 0) {
            msg = NULL;
        }
        //调用 skynet_send 将对应的消息发送到指定服务的次级消息队列当中。
        if (dest_string) {
            session = skynet_sendname(context, source, dest_string, type, session , msg, len);
        } else {
            session = skynet_send(context, source, dest, type, session , msg, len);
        }
        break;
    }
    case LUA_TLIGHTUSERDATA: {
        void * msg = lua_touserdata(L,idx_type+2);
        int size = luaL_checkinteger(L,idx_type+3);
        if (dest_string) {
            session = skynet_sendname(context, source, dest_string, type | PTYPE_TAG_DONTCOPY, session, msg, size);
        } else {
            session = skynet_send(context, source, dest, type | PTYPE_TAG_DONTCOPY, session, msg, size);
        }
        break;
    }
    default:
        luaL_error(L, "invalid param %s", lua_typename(L, lua_type(L,idx_type+2)));
    }
    if (session < 0) {
        if (session == -2) {
            // package is too large
            lua_pushboolean(L, 0);
            return 1;
        }
        // send to invalid address
        // todo: maybe throw an error would be better
        return 0;
    }
    lua_pushinteger(L,session);
    return 1;
}
```
结合上述代码及注释，当一个 lua 服务向另一个 lua 服务发送消息时，会调用`skynet.send` 函数，这个函数最终会调用 C 层的 `send_message`函数，通过对调用参数的解析，为消息添加上 type 和 session 字段，并最终调用 `skynet_send` 函数，这个函数在之前的[skynet 源码阅读笔记 —— 消息调度机制](https://www.jianshu.com/p/6aa32e53856a)说明了它的作用，这里就不多做说明。`skynet_send`函数将消息压入到指定服务的次级消息队列中，发送的过程就结束了。接下来只需要等待 worker 线程从全局消息队列中取出对应的次级消息队列，并消费相应的消息即可。
---
title: skynet 源码阅读笔记 —— 配置文件的加载
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-04 17:15:48
password:
summary:
tags: [源码剖析, skynet框架, lua语言, C语言, 配置文件]
categories: skynet源码剖析
---

#### 文前导读

skynet 是一个由云风所写的轻量级在线游戏服务器框架。本文为 skynet 框架源码剖析系列的第三篇文章，主要探讨 skynet 的配置文件加载机制，包含了以下内容：

> * main 函数是如何启动配置文件加载的？
> * 让代码描述自身：使用 lua 作为配置语言，并利用 lua 的解释器来解析配置文件
> * main 函数是如何解析配置文件的？

<!-- more -->

#### skynet 中 main 函数的流程
skynet 的 main 函数位于 skynet_main.c 文件当中，其定义如下：
```C
int main(int argc, char *argv[]) {
    const char * config_file = NULL ;
    if (argc > 1) {
        config_file = argv[1];
    } else {
        fprintf(stderr, "Need a config file. Please read skynet wiki : https://github.com/cloudwu/skynet/wiki/Config\n"
            "usage: skynet configfilename\n");
        return 1;
    }
    skynet_globalinit();
    skynet_env_init();
    sigign();
    struct skynet_config config;

#ifdef LUA_CACHELIB
    // init the lock of code cache
    luaL_initcodecache();
#endif
    //创建一个新的 lua 环境，并将 lua 库加载进去, 该函数使用默认分配函数来进行创建
    struct lua_State *L = luaL_newstate();
    luaL_openlibs(L);   // link lua lib
    //将 load_config 加载为 lua 代码块，"t" 表示代码块的类型是文本类型，代码块的名称为 “=[skynet config]”
    int err =  luaL_loadbufferx(L, load_config, strlen(load_config), "=[skynet config]", "t");
    assert(err == LUA_OK);
    //向 lua 的虚拟栈中压入 config_file,作为 load_config 的参数
    lua_pushstring(L, config_file);
    //lua_pcall 的第二个参数表示传递的参数个数，第三个参数表示期望的结果数量，第四个参数表示错误处理函数
    err = lua_pcall(L, 1, 1, 0);
    if (err) {
        //如果出错，则返回存放在虚拟栈栈顶的错误信息
        fprintf(stderr,"%s\n",lua_tostring(L,-1));
        lua_close(L);
        return 1;
    }
    _init_env(L);
    config.thread =  optint("thread",8);
    config.module_path = optstring("cpath","./cservice/?.so");
    config.harbor = optint("harbor", 1);
    config.bootstrap = optstring("bootstrap","snlua bootstrap");
    config.daemon = optstring("daemon", NULL);
    config.logger = optstring("logger", NULL);
    config.logservice = optstring("logservice", "logger");
    config.profile = optboolean("profile", 1);

    lua_close(L);
    skynet_start(&config);
    skynet_globalexit();

    return 0;
}
```
使用过 skynet 的人都知道，skynet 在启动时相应的配置文件作为参数传递给 skynet 进程，例如 `skynet example/config`。从代码上可以看出，skynet 的 `main` 函数主要流程可以分为 3 个部分：
> 1. 初始化运行环境，并通过 C API 调用相应的 lua 脚本解析配置文件，然后结果保存在 config 结构体中
> 2. 以 config 为参数启动 skynet 进入事件循环
> 3. 执行 skynet_globalexit() 完成 skynet 的退出准备


#### 使用 lua 语言描述配置文件
在了解 skynet 是如何加载配置文件前，我们先来看看配置文件究竟长什么样？skynet 在 example 目录下提供了示范的 `config.path` 文件以及 `config` 文件
```lua
--config.path 文件
root = "./"
luaservice = root.."service/?.lua;"..root.."test/?.lua;"..root.."examples/?.lua;"..root.."test/?/init.lua"
lualoader = root .. "lualib/loader.lua"
lua_path = root.."lualib/?.lua;"..root.."lualib/?/init.lua"
lua_cpath = root .. "luaclib/?.so"
snax = root.."examples/?.lua;"..root.."test/?.lua"

--config 文件
--include 为 load_config 中定义的脚本调用接口
include "config.path"

-- preload = "./examples/preload.lua"   -- run preload.lua before every lua service run
thread = 8
logger = nil
logpath = "."
harbor = 1
address = "127.0.0.1:2526"
master = "127.0.0.1:2013"
start = "main"  -- main script
bootstrap = "snlua bootstrap"   -- The service for bootstrap
standalone = "0.0.0.0:2013"
-- snax_interface_g = "snax_g"
cpath = root.."cservice/?.so"
-- daemon = "./skynet.pid"

```
skynet 的配置文件本身使用了 lua 语言来描述对应的选项，而 skynet 也是通过 C API 来调用对应的 lua 脚本对配置文件进行解析。使用 lua 语言来描述配置选项，相较于以普通的文本来描述配置文件有以下几个好处：
> 1. lua 作为一门脚本语言，本身提供了解析器及灵活丰富的语法，不仅表达能力强于文本语言，而且 C/C++ 都为其提供了良好的支持，简单易用
> 2. 使用 lua 语言描述配置文件，则配置文件本身也可以运行。你可以在配置文件中定义并调用函数，要求用户输入数据或者访问系统的环境变量等，这些都是文本语言所难以实现的。
> 3. lua 实现的配置文件可以扩展性强，当需要向配置文件中添加新的配置机制会更加方便。


#### 配置文件解析脚本
这个解析配置文件的脚本的内容则是以 C 字符串的形式保存在 `load_config` 变量当中。我们将其以 lua 代码的形式展示在下方：
```lua
-- load_config 的内容：
local result = {}
--获取相应的环境变量
local function getenv(name) return assert(os.getenv(name), [[os.getenv() failed: ]] .. name) end
--获取路径的分隔符，在 linux 下 sep = /
local sep = package.config:sub(1,1)
--将 . 和 / 合并得到了当前目录的相对路径 ./
local current_path = [[.]]..sep

--定义脚本调用接口 include
local function include(filename)
    local last_path = current_path
    --以最后一个/为分界将 filename 分割为路径 path 和文件名 name
    local path, name = filename:match([[(.*]]..sep..[[)(.*)$]])
    if path then
        --若 path 为绝对路径，则起始字符为 /
        if path:sub(1,1) == sep then    -- root
            current_path = path
        else
            --path 为相对路径的情况
            current_path = current_path .. path
        end
    else
        --若 path 为 nil，则说明 filename 不包含路径
        name = filename
    end
    --打开配置文件
    local f = assert(io.open(current_path .. name))
    --读取配置文件中的所有内容
    local code = assert(f:read [[*a]])
    --如果配置文件中存在形如 $(环境变量) 的字符串，则调用 getenv 将其替换成环境变量的值
    code = string.gsub(code, [[%$([%w_%d]+)]], getenv)
    f:close()
    --将 code 中的内容以文本的形式加载到 result,其中 t 代表文本模式。
    assert(load(code,[[@]]..filename,[[t]],result))()
    current_path = last_path
end
--设置 result 的元表，这样在调用 include 的过程中，如果 result 中没有对应的键则会自动调用 include 函数
setmetatable(result, { __index = { include = include } })
--config_name 是变长参数
local config_name = ...
--使用 include 调用 config_name 脚本
include(config_name)
--调用完 include 后将 result 的元表清除，避免遍历 result 时收到元表的影响
setmetatable(result, nil)
return result
```

#### main 函数中解析脚本的流程
在了解了配置文件的内容及 load_config 的解析流程后，我们就可以来分析一下 `main` 函数加载配置文件的详细过程了
```C
//skynet_env.c
struct skynet_env {
    struct spinlock lock;
    lua_State *L;
};
static struct skynet_env *E = NULL;
void skynet_env_init() {
    E = skynet_malloc(sizeof(*E));
    SPIN_INIT(E)
    E->L = luaL_newstate();
}
//skynet_main.c
static void _init_env(lua_State *L) {
    lua_pushnil(L);  /* first key */
    while (lua_next(L, -2) != 0) {
        int keyt = lua_type(L, -2);
        if (keyt != LUA_TSTRING) {
            fprintf(stderr, "Invalid config table\n");
            exit(1);
        }
        const char * key = lua_tostring(L,-2);
        if (lua_type(L,-1) == LUA_TBOOLEAN) {
            int b = lua_toboolean(L,-1);
            skynet_setenv(key,b ? "true" : "false" );
        } else {
            const char * value = lua_tostring(L,-1);
            if (value == NULL) {
                fprintf(stderr, "Invalid config table key = %s\n", key);
                exit(1);
            }
            skynet_setenv(key,value);
        }
        lua_pop(L,1);
    }
    lua_pop(L,1);
}
int main(int argc, char *argv[]){
    ...
    skynet_env_init();
    ...
    //创建一个新的 lua 环境，并将 lua 库加载进去, 该函数使用默认分配函数来进行创建
    struct lua_State *L = luaL_newstate();
    luaL_openlibs(L);   // link lua lib
    //将 load_config 加载为 lua 代码块，"t" 表示代码块的类型是文本类型，代码块的名称为 “=[skynet config]”
    int err =  luaL_loadbufferx(L, load_config, strlen(load_config), "=[skynet config]", "t");
    assert(err == LUA_OK);
    //向 lua 的虚拟栈中压入 config_file,作为 load_config 的参数
    lua_pushstring(L, config_file);
    //lua_pcall 的第二个参数表示传递的参数个数，第三个参数表示期望的结果数量，第四个参数表示错误处理函数
    err = lua_pcall(L, 1, 1, 0);
    if (err) {
        //如果出错，则返回存放在虚拟栈栈顶的错误信息
        fprintf(stderr,"%s\n",lua_tostring(L,-1));
        lua_close(L);
        return 1;
    }
    //将配置文件的内容添加到环境变量当中
    _init_env(L);
    //opt*(key, value) 函数会以 key 为键值访问环境变量，如果设置了该环境变量则返回对应的值，若没有设置该环境变量则将其设为 value
    config.thread =  optint("thread",8);
    config.module_path = optstring("cpath","./cservice/?.so");
    config.harbor = optint("harbor", 1);
    config.bootstrap = optstring("bootstrap","snlua bootstrap");
    config.daemon = optstring("daemon", NULL);
    config.logger = optstring("logger", NULL);
    config.logservice = optstring("logservice", "logger");
    config.profile = optboolean("profile", 1);
    lua_close(L);
    ...
}
```
从上述代码中可以看出，skynet 读取配置文件的大致流程为：先调用 `skynet_env_init` 函数初始化一个全局的 lua 环境，接着创建一个新的 lua 环境，并在该环境中使用 `luaL_loadbufferx`将 load_config 加载进来，然后使用 `lua_pushstring`将配置文件 config_file 压入 lua 的虚拟栈中。最后使用 `lua_pcall` 调用 load_config 脚本完成配置文件的解析。解析完毕后，调用 `_init_env` 将解析结果保存为环境变量。在需要时调用相关类型的 `opt` 函数读取相应的配置项。
最后我们来看看 `skynet_env` 的定义及相应函数的实现：
```C
//skynet_env.c
// skynet_env 维护了一个全局的 lua 环境
struct skynet_env {
    struct spinlock lock;
    lua_State *L;
};
static struct skynet_env *E = NULL;
//从全局的 lua 环境中查找全局变量 key
const char*  skynet_getenv(const char *key) {
    SPIN_LOCK(E)
    lua_State *L = E->L;
    lua_getglobal(L, key);
    const char * result = lua_tostring(L, -1);
    lua_pop(L, 1);
    SPIN_UNLOCK(E)
    return result;
}
//将 {key, value} 保存为全局的 lua 环境的全局变量
void skynet_setenv(const char *key, const char *value) {
    SPIN_LOCK(E)
    lua_State *L = E->L;
    lua_getglobal(L, key);
    assert(lua_isnil(L, -1));
    lua_pop(L,1);
    lua_pushstring(L,value);
    lua_setglobal(L,key);
    SPIN_UNLOCK(E)
}
//skynet_main.c
//optboolean 和 optstring 函数的实现逻辑与 optint 相似。
static int optint(const char *key, int opt) {
    const char * str = skynet_getenv(key);
    if (str == NULL) {
        char tmp[20];
        sprintf(tmp,"%d",opt);
        skynet_setenv(key, tmp);
        return opt;
    }
    return strtol(str, NULL, 10);
}
```
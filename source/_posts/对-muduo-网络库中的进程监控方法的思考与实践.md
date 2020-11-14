---
title: 对 muduo 网络库中的进程监控方法的思考与实践
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-14 22:11:23
password:
summary:
tags: [muduo网络库, 进程监控, Observer Pattern, REST]
categories: muduo源码剖析
---

#### 文前导读
本文是 muduo 网络库源码剖析系列文章的第九篇文章，主要涉及了 muduo 网络库中是如何获取进程信息，又是以什么样的方式展现给客户的，其中包含了以下内容：
> * 幕后功臣 ： FileUtil 的实现
> * 一种基于 REST 风格的进程监控
> * 一种基于 Observer Pattern 实现的进程监控
> * 最后一点碎碎念

注：由于这两种进程监控的方法都涉及到了与其他组件的配合，例如 HttpServer、hub 等组件，碍于篇幅原因，本文中不会粘贴过多的代码，主要以图表的形式阐述。本文假定读者了解 Http 协议， Observer Pattern 以及 linux 下文件系统等相关知识。另外，为了避免代码与我之前文章代码不一致，此处涉及代码引用自 tmuduo。在进程相关方面， tmuduo 只做了及其微小的修改，与 muduo 源码差距很小。在设计上以 muduo 的思路为主，但代码还是来自于 tmuduo，望知悉。
本着“纸上学来终觉浅，绝知此事要躬行”的想法，我将 muduo 网络库重新实现了一遍，并在上面验证了自己的不少猜想。项目的仓库地址为 git@github.com:Phoenix500526/Tmuduo.git, 欢迎各位fork + star，一起加入学习
<!-- more -->
#### 幕后功臣 ： FileUtil 的实现
linux 的设计哲学之一便是“一切皆文件”，连进程信息也不例外。在 linux 系统下，想要获取进程信息不是一件难事，只需要读取 procfs 下相应的文件即可。在 tmuduo 中，用来读取文件信息的类有两个，一个是用来获取文件信息的 ReadSmallFile，另一个则是用来向文件中追加信息的 AppendFile，它们均被定义在 tmuduo/base/FileUtil.{h,cc} 文件中，而本文要关注的是 ReadSmallFile
```C++
// read small file < 64KB
class ReadSmallFile : noncopyable {
 public:
  ReadSmallFile(StringArg filename): fd_(::open(filename.c_str(), O_RDONLY | O_CLOEXEC)), err_(0) {
    buf_[0] = '\0';
    if (fd_ < 0) {
      err_ = errno;
    }
  }
  ~ReadSmallFile() {
    if (fd_ >= 0) {
      ::close(fd_);
    }
  }
  // return errno
  template <typename String>
  int readToString(int maxSize, String* content, int64_t* fileSize,
                   int64_t* modifyTime, int64_t* createTime);

  // Read at maxium kBufferSize into buf_
  // return errno
  int readToBuffer(int* size);

  const char* buffer() const { return buf_; }
  static const int kBufferSize = 64 * 1024;

 private:
  int fd_;
  int err_;
  char buf_[kBufferSize];
};

// read the file content, returns errno if error happens.
template <typename String>
int readFile(StringArg filename, int maxSize, String* content,
             int64_t* fileSize = nullptr, int64_t* modifyTime = nullptr,
             int64_t* createTime = nullptr) {
  ReadSmallFile file(filename);
  return file.readToString(maxSize, content, fileSize, modifyTime, createTime);
}
```
muduo 中在 ProcessInfo.h 中提供了一系列的 API，提供给用户调用以获得进程信息，我们来看一个例子：
```C++
std::string ProcessInfo::procStatus() {
  std::string result;
  FileUtil::readFile("/proc/self/status", 65536, &result);
  return result;
}
```
从上例可以看到，由于 linux 系统本身的支持，使得获取进程信息这件事情变得非常简单。然而，procfs 虽然便利，但是也有两个不足：
* 只能暴露 system-wide 的数据，不能查看每个进程内部的数据
* procfs 是本地文件系统，必须登录到这台机器上才能查看，如果要管理多台机器，那么工作量就会大大增加。
对于第一个问题，我们可以通过让程序主动暴露出内部状态，而针对第二个问题，muduo 中提供了两种不同的做法，以下分别讨论。

#### 基于 REST 风格的进程监控
针对 procfs 的第二个问题，muduo 中所提供的第一种是一种基于 REST 风格的进程监控。所谓 REST 风格，简而言之便是以 “**URL 定位资源，用 HTTP 动词描述操作**”。在分布式系统当中，使用 HTTP 获取进程信息有以下好处：
> * 使用 HTTP 协议，无需使用特定的客户端程序，只需要使用普通的 Web 浏览器就可以实现远程访问，紧急情况下，也可以使用 curl/wget 甚至是 telnet 来访问。
> * 实现方便，而且也很容易编写一些脚本来进行自动化的状态收集与分析
> * 可以通过 URL 区分资源，很容易实现有选择地查看信息，而不是把进程信息一股脑全 dump 出来
> * HTTP 天生支持聚合，一个浏览器页面内可以内置多个 iframe，能一眼看清多个进程的状态。
> * 除了 GET 方法外，如果有必要，还可以实现 PUT/POST/DELETE 等方法，通过 HTTP 协议来控制并修改进程状态，让程序做到“能观能控”

其中，Inspector 的实现框图大致如下：
<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/43866203-b1b7-49b1-b803-6dbe6a337944.png"/>
</div>
<div class="image-caption" align="center">HttpInspector框图</div>


muduo 中相关代码位于 net/inspect/ 下
```C++
class Inspector : noncopyable {
 public:
  using ArgList = std::vector<std::string>;
  using Callback =
      std::function<std::string(HttpRequest::Method, const ArgList& arg)>;
  Inspector(EventLoop* loop, const InetAddress& httpAddr, const std::string& name)
    : server_(loop, httpAddr, "Inspector:" + name),
      processInspector_(new ProcessInspector),
      systemInspector_(new SystemInspector) {
    assert(CurrentThread::isMainThread());
    assert(g_globalInspector == 0);
    /* omitted some code */
    server_.setHttpCallback(std::bind(&Inspector::onRequest, this, _1, _2));
    processInspector_->registerCommands(this);
    systemInspector_->registerCommands(this);
#ifdef HAVE_TCMALLOC
    performanceInspector_.reset(new PerformanceInspector);
    performanceInspector_->registerCommands(this);
#endif
    loop->runAfter(0, std::bind(&Inspector::start, this));
  }
  // Add a Callback for handling the special uri : /module/command
  void add(const std::string& module, const std::string& command,
           const Callback& cb, const std::string& help){
    UniqueLock lock(mutex_);
    modules_[module][command] = cb;
    helps_[module][command] = help;
  }

 private:
  using CommandList = std::map<std::string, Callback>;
  using HelpList = std::map<std::string, std::string>;

  void onRequest(const HttpRequest& req, HttpResponse* resp) {
    if (req.path() == "/") {
      std::string result;
      UniqueLock lock(mutex_);
      for (std::map<std::string, HelpList>::const_iterator helpListI = helps_.begin();
          helpListI != helps_.end(); ++helpListI) {
        const HelpList& list = helpListI->second;
        for (const auto& it : list) {
          result += "/";
          result += helpListI->first;
          result += "/";
          result += it.first;
          size_t len = helpListI->first.size() + it.first.size();
          result += std::string(len >= 25 ? 1 : 25 - len, ' ');
          result += it.second;
          result += "\n";
        }
      }
      resp->setStatusCode(HttpResponse::HttpStatusCode::k200Ok);
      resp->setStatusMessage("OK");
      resp->setContentType("text/plain");
      resp->setBody(result);
    } else {
      std::vector<std::string> result = split(req.path());
      bool ok = false;
      /* omitted some code */
      std::string module = result[0];
      UniqueLock lock(mutex_);
      std::map<std::string, CommandList>::const_iterator commListI =
          modules_.begin();
      if (commListI != modules_.end()) {
        std::string command = result[1];
        const CommandList& commList = commListI->second;
        CommandList::const_iterator it = commList.find(command);
        if (it != commList.end()) {
          ArgList args(result.begin() + 2, result.end());
          if (it->second) {
            resp->setStatusCode(HttpResponse::HttpStatusCode::k200Ok);
            resp->setStatusMessage("OK");
            resp->setContentType("text/plain");
            const Callback& cb = it->second;
            resp->setBody(cb(req.method(), args));
            ok = true;
          }
      }
    }
      if (!ok) {
        resp->setStatusCode(HttpResponse::HttpStatusCode::k404NotFound);
        resp->setStatusMessage("Not Found");
      }
    }
  }

  HttpServer server_;
  std::unique_ptr<ProcessInspector> processInspector_;
  std::unique_ptr<PerformanceInspector> performanceInspector_;
  std::unique_ptr<SystemInspector> systemInspector_;
  Mutex mutex_;
  std::map<std::string, CommandList> modules_ GUARDED_BY(mutex_);
  std::map<std::string, HelpList> helps_ GUARDED_BY(mutex_);
};
```
其中，结合大致的框图和相应的代码，可以看出 Inspector 的大致工作流程如下：
> 1. 将 onRequest 注册为 server_ 的 HttpCallback 函数
> 2. 在 Inspector 构造初期调用 processInspector_ 等对象的 registerCommands 函数，将其中的命令都注册到 Inspector 的 modules_ 和 helps_ 当中。
> 3. 启动事件循环，等待客户端发送相应的 http 请求，然后根据 url 指向相应 module 的命令，并将执行结果打包成为 http 响应报文，发送给用户。

其中 `ProcessInspector`, `SystemInspector` 以及 `PerformanceInspector` 的实现是大体一致的，我们以 `ProcessInspector` 为例，看看其对应的实现如何：
```C++
class ProcessInspector : noncopyable {
 public:
  void registerCommands(Inspector* ins) {
    ins->add("proc", "overview", ProcessInspector::overview,
           "print basic overview");
    ins->add("proc", "pid", ProcessInspector::pid, "print pid");
    ins->add("proc", "status", ProcessInspector::procStatus,
           "print /proc/self/status");
    ins->add("proc", "opened_files", ProcessInspector::openedFiles,
           "count /proc/self/fd");
    ins->add("proc", "threads", ProcessInspector::threads,
           "list /proc/self/task");
  }

  static std::string ProcessInspector::pid(HttpRequest::Method,
                                  const Inspector::ArgList&) {
    char buf[32];
    snprintf(buf, sizeof buf, "%d", ProcessInfo::pid());
    return buf;
  }
  /* omitted */
};

```
其中 `registerCommands` 会调用 `Inspector` 的 `add` 函数将对应的命令名称、描述信息以及对应对调函数注册到 `Inspector` 中。

#### 基于 Observer Pattern 实现的进程监控
muduo 还提供了另外一种基于 Observer Pattern 的实现。这种方式有点像 redis 的集群机制，让不同的节点将自身的相关信息发布到对应的 topic 上，有需要的节点只需要订阅相关的 topic 就可以获得相应节点的进程信息。

在基于 Observer Pattern 模式实现的进程监控方式中，主要涉及了四个不同的概念，分别是话题`Topic`, 集线器`Hub`，发布者`Pub`以及订阅者`Sub`,它们之间的关系如下：
<div style="text-align:center">
 <img src="https://vkceyugu.cdn.bspapp.com/VKCEYUGU-imgbed/88b75116-5611-4d72-a8a4-b0abe7bcc837.png"/>
</div>
<div class="image-caption" align="center">基于 Observer Pattern 模式实现的进程监控</div>

muduo 当中的 Hub 示例采用了 "\r\n" 分界的文本协议，这样做可以简单地使用 telnet 测试 Hub，使用时采用命令的方式，如：
> * 订阅 topic: sub \<topic\>\r\n
> * 退订 topic: unsub \<topic\>\r\n
> * 发布 content: pub \<topic\>\r\n\<content\>\r\n

由于代码篇幅过长且 sub 和 pub 都可以用 telnet 来代替，因此这里只放了 hub 以及 Topic 的相关实现。，有需要的可以自行将代码下下来研究。先来看看 Topic 的实现：
```C++
class Topic : public tmuduo::copyable {
 public:
  Topic(const string& topic) : topic_(topic) {}

  void add(const TcpConnectionPtr& conn) {
    audiences_.insert(conn);
    if (lastPubTime_.valid()) {
      conn->send(makeMessage());
    }
  }
  /* omitted */
  void publish(const string& content, Timestamp time) {
    content_ = content;
    lastPubTime_ = time;
    string message = makeMessage();
    for (auto it = audiences_.begin(); it != audiences_.end(); ++it) {
      (*it)->send(message);
    }
  }

 private:
  string makeMessage() { return "pub " + topic_ + "\r\n" + content_ + "\r\n"; }
  string topic_;
  string content_;
  Timestamp lastPubTime_;
  std::set<TcpConnectionPtr> audiences_;
};
```
对于 Topic 而言，关键的接口为 `add` 和 `publish`，其中 `add` 会将订阅了该 topic 的连接保存到 `audiences_` 当中，而 `publish` 则负责向 `audiences_` 中的连接发送发布消息。从框图中也可以看到在 hub 当中保存了一个 `topics_` 的 map, 用于保存所有的 `topic`,其实现相关部分如下：
```C++
class PubSubServer : tmuduo::noncopyable {
 public:
  PubSubServer(EventLoop* loop, const InetAddress& serverAddr)
      : loop_(loop), server_(loop, serverAddr, "PubSubServer") {
    server_.setConnectionCallback(
        std::bind(&PubSubServer::onConnection, this, _1));
    server_.setMessageCallback(
        std::bind(&PubSubServer::onMessage, this, _1, _2, _3));
    loop_->runEvery(1.0, std::bind(&PubSubServer::timePublish, this));
  }

 private:
  /* omitted */
  void onMessage(const TcpConnectionPtr& conn, Buffer* buf,
                 Timestamp receiveTime) {
    ParseResult result = ParseResult::kSuccess;
    while (result == ParseResult::kSuccess) {
      string cmd;
      string topic;
      string content;
      result = parseMessage(buf, &cmd, &topic, &content);
      if (result == ParseResult::kSuccess) {
        if (cmd == "pub") {
          doPublish(conn->name(), topic, content, receiveTime);
        } else if (cmd == "sub") {
          LOG_INFO << conn->name() << " subscribes " << topic;
          doSubscribe(conn, topic);
        } else if (cmd == "unsub") {
          doUnsubscribe(conn, topic);
        } else {
          conn->shutdown();
          result = ParseResult::kError;
        }
      } else if (result == ParseResult::kError) {
        conn->shutdown();
      }
    }
  }

  void timePublish() {
    Timestamp now = Timestamp::now();
    doPublish("internal", "utc_time", now.toFormattedString(), now);
  }
private:
  EventLoop* loop_;
  TcpServer server_;
  std::map<string, Topic> topics_;
};
```
在 hub 的实现上，比较关键的两个函数是 `onMessage` 和 `timePublish`，当其他节点向 hub 发送消息时，会触发 `onMessage` 函数的执行，该函数会根据消息的内容执行相应的函数，而 `timePublish` 则由定时事件进行触发。这里 `timePublish` 保留了原有的实现，如果需要发布进程信息可以直接在此进行修改。

#### 最后一点想法
到了这里，基于 Observer Pattern 模式实现的进程监控也大体说完。这两种方式实现不难，只需要了解其中大体流程及 muduo 的相关用法，也可以很快地写出来。在基于 Observer Pattern 的进程监控方式中，一个比较大的问题是 hub 很容易成为 single point of failure. 关于这点，也许在 Redis 的源码中会有解决的方案。而这也是我写这篇文章的目的之一：如果未来有时间，我希望在解决这个问题后，可以直接从这篇文章中接续下去，而不必重新开始。
这篇文章其实并不好写，原因是由于 linux 的 procfs 的实现使得获取进程信息非常简单，如果单独写篇文章比较划不来。可如果将这两种模式总结进来，又很容易使得文章变得非常冗长。
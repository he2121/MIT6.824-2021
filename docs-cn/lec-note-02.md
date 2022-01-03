---
title: "lecture-note-02"
date: 2021-12-16T14:30:00+08:00
tags: ["6.824", "翻译", "note"]
---
# 6.824 2021 Lecture 2: Infrastructure: RPC and threads

[笔记源地址](https://pdos.csail.mit.edu/6.824/notes/l-rpc.txt)

## 内容

讲 Go 中的线程和RPC，着眼于实验室

### 为什么 lab 使用 Go

- 对线程的良好支持
- 方便的 RPC 包
- 类型安全
- 垃圾收集（释放问题后没有用）。
- 线程+GC是特别有吸引力的!
- 相对简单
- 教程结束后，使用 https://golang.org/doc/effective_go.html

## 多线程

线程是一个有用的结构化工具，但可能很棘手，Go 称其为goroutines，其他的人则称其为线程。

线程允许一个程序同时做很多事情

- 每个线程都是串行执行的（单核），就像一个普通的非线程程序一样
- 各线程共享内存
- 每个线程都包括一些每个线程的状态。程序计数器、寄存器、堆栈

### 为什么使用多线程

它们表示并发性，这是你在分布式系统中需要的

- I/O并发性
     - 客户端向许多服务器并行发送请求，并等待回复。
     -  服务器处理多个客户端请求；每个请求都可能被阻塞。
     -  在等待磁盘为客户 X 读取数据的时候。处理一个来自客户端Y的请求。
- 多核性能
      - 在几个核心上并行地执行代码。
- 方便性
     - 在后台，每秒钟一次，检查每个 worker 是否仍然活着。

### 是否存在多线程的替代

存在：在一个单线程中，编写明确交错活动的代码，通常称为 "事件驱动"。事件驱动保持一个关于每个活动的状态表，例如每个客户的请求。

一个 "事件 "循环，即

- 检查每个活动的新输入（例如，服务器回复的到来）。
- 为每个活动做下一个步骤。
- 更新状态

事件驱动让你获得 I/O 并发性并消除了线程切换成本（这可能是相当大的）。但不能获得多核加速，而且编程起来很痛苦。

### 多线程的挑战

- 共享数据， 例如

  - 两个线程同时更新 n = n + 1

  - 一个线程在读，另一个线程在加

  - 这就是竞争 -- 通常是一个 bug，解决方案：

    - 用锁（Go 的 sync.Mutex）

    - 避免共享可变数据

- 线程通信，例如
  - 一个线程生产数据，另一个线程消费它
  - 消费者如何等待（并释放CPU），生产者如何唤醒消费者？
  - 解决方案：
    - 使用 Go channel 或者 sync.Cond 或者 WaitGroup
- 死锁

下面让我们看看教程的 Web 爬虫作为一个多线程示例。

### 什么是网络爬虫

目标是获取所有的网页，例如，提供给一个索引器。网页和链接形成一个图，多个链接指向某些网页，图有环

### 爬虫挑战

- 利用I/O并发性
  		- 网络延迟比网络容量更具限制性
    - 为了增加每秒钟获取的URL数量
         - 需要线程的并发性
- 每个 URL 只取一次
  - 避免浪费网络带宽
  - 对远程服务友好
    - 需要记录访问过的 URL
- 知道什么时候结束

### 三种解决方案

[crawler.go](https://pdos.csail.mit.edu/6.824/notes/crawler.go) 

#### 串行爬虫

通过递归序列调用进行深度优先探索，“fetch” map 避免了重复和循环搜索。这个 map 通过引用传递，调用者能看到被调用者更新的数据。

但是：

- 每次只获取一个页面
- 我们能不能把 `go`放到 `Serial()`前面
- 试一试，看看会发生什么

#### 并发 Mutex 爬虫

为每个获取的页面创建一个线程，许多并发的抓取，更高的抓取率。`go func`创建一个`goroutine`并开始运行，func... 是一个匿名函数。这些线程共享 "fetch" map。因此，只有一个线程会获取任何特定的页面

为什么要使用 `Mutex`（`Lock()`和`Unlock()`）？

- 两个不同的网页指向了同一个 URL，两个线程同时抓取这两个网页。T1 读取 fetched[url]，T2 读取 fetched[url]。两个线程都看到 URL 还没有被取走。导致同一个网页，两次抓取。加锁会使检查和更新是原子性的，只会有一个线程看到  fetched[url] = false

- 并发更新/更新可能破坏内部不变性，同时进行的更新/读取可能会使读取崩溃
- 如果我把 `Lock()`/`Unlock()`注释掉呢？
     - `go run crawler.go`为什么它能工作？      
     - `go run -race crawler.go`即使输出是正确的，也会检测到 ‘race’

并发 Mutex 爬虫怎么知道结束了？

- `sync.WaitGroup`:`Wait()`等待所有的`Add()`, 被`Done()`平衡。

这个爬虫会创建多少线程？

#### 并发 channel 爬虫

##### Go channel

- channel 是一个对象 `ch := make(chan int)`
- 一个 channel 让一个线程向另一个线程发送一个对象(ps：消息)
- `ch <- x`： sender 线程等待，直到某个 goroutine 收到这个消息（ps: 无缓冲 channel）
- `y := <- ch`,`for y := range ch`, 接收者等待，直到某个goroutine 发送了消息。
- channel 既能通信又能同步
- 几个线程可以在一个通道上发送和接收信息，并发安全
- channel 便宜
- 记住：发送方阻塞，直到接收方收到为止。"同步性"。小心死锁

##### `ConcurrentChannel master()`

- master() 创建一个 worker goroutine 来获取每个页面。

- worker() 在一个 channel 上发送抓取到的页面 URL，多个 worker 在一个 channel 上发送

- 主线程在哪一行等待阻塞？当主线程阻塞等待时使用 CPU 么

- 不需要给 fetch map 加锁，因为它不是共享的

- master 怎么知道任务全完成了

  - 计数 worker 的数量 n
  

##### 为什么多个线程使用一个 channel 没有 ‘race’

##### 当 worker 线程向 channel 写入 slice，主线程读取该 slice，没有用锁，是否存在 ‘race’？

* worker 只在发送前写 slice 
* 主线程只在接收后读取 slice，所以他们不能同时使用该slice。

##### 什么时候使用锁，而不是 channel

大多数问题都可以用以上两种方式解决，实际使用取决于程序员认为什么最重要

- state（状态） -> lock
- 通信 -> channel
- 对于 6.824 实验，我建议用共享+锁来处理状态，和`sync.Cond`或 `channel`  或 `time.Sleep()` 用于等待/通知。

## 远程过程调用（RPC）

分布式系统机制的一个关键部分；所有实验都使用 RPC

目标：易于编程的客户端/服务器通信。隐藏网络协议的详细信息。将数据(字符串、数组、slice 等等)转换为[Wire格式](https://en.wikipedia.org/wiki/Wire_protocol)

RPC 消息示意图：

```bash
  Client             Server
    request--->
       			<---response
```

软件架构：

```bash
  client app        handler fns
   stub fns         dispatcher
   RPC lib           RPC lib
     net  ------------ net
```

### Go RPC 例子

[kv.go](https://pdos.csail.mit.edu/6.824/notes/kv.go)

一个玩具的 key/val 的存储服务器  -- Put(key,value), Get(key)-> value， 使用了 Go 的 RPC 库。

#### 代码说明

#### Common

为每个服务 hanlder 声明 Args 和 Reply 结构。

#### Client

- `connect()` 的 `Dial()` 创建一个与服务器相连的 TCP 连接。

- `get()`和`put()`是客户端的 stub
- `Call()`要求 RPC 库执行调用。
  - 你指定服务器函数的名称、参数、reply。
  - RPC 库的序列化 args，发送请求，并等待回复，然后反序列化 reply
  - `Call()` 的返回值表明它是否得到了回复
  - 通常你也会有一个 `reply.Err` 表示服务级别的失败

#### Server

Go 要求 server 声明一个带有方法的对象作为 RPC hanlder，然后服务器将该对象注册到 RPC 库中。Server 接受 TCP 请求，将其交给 RPC 库。RPC 库：

1. 读取每个 request

2. 为这个 request 创建一个新的 goroutine

3. 反序列 request

4. 查找指定的对象（在 Register() 创建的表中

5. 调用该对象的命名方法（派发）
6. 序列化 reply
7. 写 reply 到 TCP 连接

Server 的 Get() 和 Put( ) handler，必须使用 LockI()，因为 RPC 库为每个 request 新建一个 goroutine 处理，可能有并发问题

#### RPC ：怎么应对失败

如：丢失数据包，网络中断，服务器速度慢，服务器崩溃

##### 对 client 来说，失败是什么样子的？

- 客户端从未看到服务器的响应
- 客户端并不知道服务器是否看到了这个请求!
    - 也许服务器从未见过这个请求
    - 可能是服务器执行了，在发送回复前崩溃了
    - 可能是服务器执行了，但在发送回复之前网络就中断了。

#### 最简单的故障处理方案，"最大交付"

`Call()`等待响应一段时间，如果没有回复，则**重新发送请求**。多试几次，然后放弃并返回一个错误。

#### "最大交付 "对应用程序来说是否容易应对？

一个特别糟糕的情况。一个 client 先后执行了`Put("k", 10)` `Put("k", 20)`。都成功了，`Get("k") `会得到什么？

[图，超时，重发，原件晚到]

问：最大交付是否可以？

- 只读操作
- 如果重复操作，则不做任何事情，例如，DB检查记录是否已经被插入。

#### 更好的RPC行为："最多一次（at most once）"

想法：Server 检测重复的请求返回之前的回复，而不是重新运行处理程序。

问：如何检测重复的请求

答：客户端在每个请求中包括唯一的ID（XID）。再次发送时使用相同的XID

```go
  server:
    if seen[xid]:
      r = old[xid]
    else
      r = handler()
      old[xid] = r
      seen[xid] = true

```

#### 一些最多一次的复杂情况

这些情况会出现在 lab3

1. ##### 如果两个 client 使用了相同的 XID？

- 使用大随机数字
- 组合唯一的客户端 ID（IP）

2. ##### server 最终必须丢弃有关旧 RPC 的信息， 什么时候丢弃安全

- 每一个 client 都有着唯一的 ID（可能是大随机数），每个 client 的 RPC 序列号，客户端在每个 RPC 中包括 "看到的所有回复<=X"。
- 类似于 TCP 序列号和 acks
- 或者一次只允许 client 有一个未完成的 RPC，seq+1的到来允许服务器丢弃所有 <=seq 的内容

3. ##### 如何在原请求仍在执行的情况下处理重复的请求？

- server 还不知道 reply 是什么
- 挂起，等待或者忽略

4. ##### 如果一个最多一次的服务器崩溃并重新启动怎么办？

服务器会丢失之前在内存中数据并在重新启动后接受重复的请求

- 也许它应该把重复的信息写到磁盘上
- 也许复制服务器也应该复制重复的信息

#### Go RPC 是最多一次的简单形式

1. 打开 TCP 连接
2. 向 TCP 连接写入 request
3. Go RPC 从不重发 request，所以 Go Server 不会见到重复的请求
4. Go Client 当没有收到回复时，返回一个错误
   1. 可能 TCP 超时了
   2. 可能 server 没见到 request
   3. 可能 server 处理了 request 但是在返回前 server、网络 故障了

#### 那 "正好一次(exactly once) "呢？

无限制的重试加重复检测加容错服务

lab3
---
title: "lecture-note-01"
date: 2021-12-16T14:30:00+08:00
tags: ["6.824", "翻译", "note"]
---

# 6.824 2021 Lecture 1: Introduction

[笔记源地址](https://pdos.csail.mit.edu/6.824/notes/l01.txt)

[MapReduce 论文](http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)

## 6.824: 分布式系统

### 什么是分布式系统？

- 多台电脑合作
- 大型网站的存储、MapReduce、点对点共享，等等。
- 很多关键的基础设施是分布式的

### 为什么要构建分布式系统？

- 通过并行增加容量
- 通过复制来实现容错
- 将计算置于与外部实体相近的物理位置
- 通过隔离来实现安全性

### 分布式系统也带来一些问题

- 许多并发部分，复杂的交互
- 必须应对部分故障
- 充分发挥分布式系统性能潜力比较难

### 为什么要上这个课？

- 有趣 -- 困难的问题，强大的解决方案
- 被实际的系统所使用 -- 由大型网站所驱动
- 在研究领域很活跃 -- 重要并且尚未解决的问题
- 实践 -- 你将在 lab 中建立真实的系统

## 课程结构

[地址](http://pdos.csail.mit.edu/6.824)

### 授课人员

- Frans Kaashoek, 主讲人
- Lily Tsai, 助教
- Cel Skeggs, 助教
- David Morejon, 助教
- Jose Javier Gonzalez, 助教

### 课程构成

- 讲课
- 论文
- 2 次考试
- 实验
- 结业项目（可选）

#### 讲课

- 宏观想法，论文讨论和实验

- 会被录下来，可在网上查阅

#### 论文

- 研究论文，一些经典，一些新的
- 问题、想法、实现细节、评价
- 许多讲座都以论文为中心
- 请在课前阅读论文!
- 每篇论文都有一个简短的问题供你回答
- 并且要求你们发给我们一个关于这篇论文的问题
- 在课前提交问题和答案

#### 考试

- 期中考试在课堂上进行
- 期末考试在期末考试周进行
- 主要是关于论文和实验

#### 实验

- 目标：加深对一些重要技术的理解
- 目标：体验分布式编程
- 第一个实验是在星期五之后的一个星期内完成
- 此后的一段时间内，每周一个实验

```bash
Lab 1: MapReduce
Lab 2: replication for fault-tolerance using Raft
Lab 3: fault-tolerant key/value store
Lab 4: sharded key/value store
```

- 期末专题可选，两人或三人一组。可替代了实验室4。你想一个项目，然后和我们商量。最后一天做一个展示：demo，代码，短的文字说明
- lab 的成绩取决于你通过多少个测试案例，我们给你测试案例，所以你知道你是否会做得好
- 调试实验可能会很耗费时间
  - 早点开始
  - 找助教
  - 在Piazza上问问题

## 主要内容

这是一个关于应用程序基础设施的课程。
  * 存储。
  * 通信。
  * 计算。

大的目标是: 隐藏分布式复杂。在我们的研究中，有几个主题会反复出现。

### 主题：容错

1000 多台服务器，大网络 -> 总是出现一些故障，我们想从应用程序中隐藏这些故障。

我们经常希望：

- 可用性(Availability) 		  -- 尽管出现故障，应用程序仍能继续    
- 可恢复性(Recoverability)  -- 当故障被修复时，应用程序能够复原

想法：冗余服务，如果一个服务崩溃了，可以用另一个（或几个）继续进行。

实现比较困难：

 - 服务可能没有崩溃，但只是对某些服务来说无法到达
 - 但其仍在为 client 的请求提供服务
 - 实验室 1、2 和 3 与这个相关

### 主题：一致性

通用基础设施需要明确的行为，不能出现模糊的结果

例如：“Get(k) 得出最近一次 Put(k,v) 的值”。实现这个明确的行为是很难的， "服务器在“复制”的情形下很难保持一致。

### 主题：性能

目标：可扩展的吞吐量，N 个服务器 -> 对应着 N 个并行 CPU、磁盘、网络的总吞吐量。

随着 N 的增加，扩展变得更加困难:

- 负载不平衡，掉队，N 最慢的延迟。 
- 不可并行的代码: 初始化，交互。
- 来自共享资源的瓶颈，例如网络。

有些性能问题不容易通过扩展解决，例如，

- 单个用户请求的快速响应时间

- 所有用户都希望更新相同的数据

所以通常需要更好的设计，而不仅仅是加机器

lab 4 与这个相关

### 主题: 容错、一致性和性能是互斥的

强容错能力需要通信

- 例如，发送数据到备份

强一致性需要通信，

- Get() 必须检查最近的 Put()。

许多设计只提供微弱的一致性，以获得性能。

- Get() 得到的结果不是最新的 Put()

这对程序员来说很痛苦，但可能是一种很好的权衡。

在一致性/性能中权衡中有许多可能的设计点

### 主题：实现

RPC、线程、并发控制

实验...

## 历史背景

1. 局域网和互联网应用程序(自20世纪80年代以来)

   - 10 - 100机器: AFS

   - 互联网规模的应用: DNS 和 email

2. 数据中心 (data center)  (90年代末 / 21世纪初)

   - 拥有许多用户(数百万)和大量数据的Web站点

   - 谷歌，雅虎，Facebook，亚马逊，微软等等。

   - 早期应用: 网络搜索、电子邮件、购物等。

   - 很酷很有趣的系统

   - 1000台机器

   - 系统主要用于内部使用，工程师们写关于它们的研究论文

3. 云计算 (cloud computing )

   - 用户将计算/存储外包给云提供商

   - 用户在云上运行自己的大型网站

   - 用户运行大量数据的大型计算(例如，机器学习)
   - 涌现出了许多新的面向用户的分布式系统基础设施

4. 现状

   - 学术界和产业界的研究和发展非常活跃，很难跟上!
   - 6.824 介绍论文中的一些系统已经过时，但概念仍然是相关的
   - 6.824 重在容错/存储，但也涉及到了通信和计算

## 案例研究：MapReduce

让我们把 MapReduce(MR) 作为一个案例来讨论吧

- 很好地说明了6.824 的主要议题
- 具有极大的影响力
- 第一个实验的内容

### MapReduce 概述

**背景**：在 TB 级数据集上进行小时级的计算

- 如：建立搜索索引，或排序，或分析网络结构
- 只有在有1000台计算机的情况下才实用
- 不是由分布式系统专家编写的应用程序

**总体目标**：对非专业（不熟悉分布式系统）的程序员来说很简单，程序员只需定义实现 Map 和 Reduce 函数，而这些通常是相当简单的顺序代码，MR 负责并隐藏了分布式的所有方面!

### MapReduce 工作的抽象视图

1. 输入被（已经）分割成 M 个文件

```bash
  Input1 -> Map -> a,1 b,1
  Input2 -> Map ->     b,1
  Input3 -> Map -> a,1     c,1
                    |   |   |
                    |   |   -> Reduce -> c,1
                    |   -----> Reduce -> b,2
                    ---------> Reduce -> a,2
```

2. MR为每个 input 文件调用 Map()，产生一组 k2,v2 的中间 (intermediate) 数据，每个Map()的调用都是一个 "task"。

3. MR 收集给定 k2 的所有中间(intermediate) v2。并将每个键+值传递给一个 Reduce 调用。最终输出是来自 Reduce() 的<k2,v3>对的集合。

例子：字符计数

```bash
input is thousands of text files

Map(k, v)
  split v into words
  for each word w
    emit(w, "1")
    
Reduce(k, v)
  emit(len(v))
```

MapReduce 的扩展性很好，N 台 work  计算机可以获得 N 倍的吞吐量。Maps() 可以并行运行，因为它们不相互影响。Reduce() 也一样。所以你可以通过购买更多的计算机来获得更多的吞吐量。

MapReduce 隐藏了很多细节。

- 向服务器发送应用程序代码
- 跟踪哪些任务已经完成
- 将数据从 Map 转移到 Reduce
- 平衡服务器上的负载
- 从故障中恢复

然而，MapReduce 限制了应用程序可以做什么。

- 没有互动或状态（除了通过中间输出）。
- 没有迭代，没有多阶段管道。
- 没有实时或流式处理。

输入和输出都存储在 GFS 集群文件系统上, MR 需要巨大的并行输入和输出的吞吐量。GFS将文件分割到许多服务器上，以 64MB 为一个大块

- Map 并行读取
- Reduce 并行方式写入
-  GFS 还将每个文件复制到 2或 3 个服务器上。拥有 GFS 对MapReduce 来说是一个巨大的胜利

#### 哪些方面可能会限制性能？

我们关心这个，因为那是要优化的东西。CPU、内存、磁盘、网络？

在 2004 年，当时的作者们受到了网络的限制

- Maps 从 GFS 读输入
- Reduces 读 Map 输出
-  Reduces 往 GFS 写 output

在 MR 的 all to all shuffle 中，有一半的流量通过根交换机。论文中的根交换机: 总共100 到 200gb/s

- 1800 台机器，每台机器 55M/s

- 55 M 是小的，远小于磁盘或 RAM 的速度。

今天: 网络和根交换机比 CPU/disk 快得多。

### 一些细节(论文图表 1)

需要一个协调者 coordinator，把任务分发给 workers 并记录进展。

1. coordinator 将 Map 任务分配给 workers，直到所有  Maps 完成, Maps 将写输出(中间数据)写到到本地磁盘，按哈希值将输出分割到每个 Reduce 任务的一个文件中
2. 在所有 Maps 完成后，协调器会分派 Reduce 任务，每个 Reduce任务从（所有）Map 工作者那里获取其中间输出，每个 Reduce 任务在 GFS上 写一个单独的输出文件

#### MR 如何尽量减少网络的使用？

1. coordinator 试图在存储其输入的 GFS 服务器上运行每个 Map 任务。

      - 所有的计算机都同时运行GFS和MR工作者

      - 因此，输入信息是从本地磁盘（通过GFS）读取的，而不是通过网络

2. 中间数据只经过一次网络。
   - Map worker 写到本地磁盘。
   - Reduce worker 直接从 Map work 那里读取，而不是通过GFS。
3. 中间数据被分割成持有许多 key 的文件。
      - R 要比 keys 数量小得多。
      - 大的网络传输更有效率。

####  MR 如何获得良好的负载平衡？

如果 N-1台服务器必须等待 1 台慢速服务器完成，则是浪费和缓慢的。但有些任务可能比其他任务花费更多时间

解决方案：task 比worker 多得多。
    - 协调员将新 task 交给先完成 task 的 worker。
    - 因此，没有一个 task 大到可以支配完成时间（希望如此）。
    - 因此，速度快的服务器比速度慢的服务器做更多的task，完成的时间基本相同。

#### 容错？

例如：worker 在进行 MR 工作时崩溃了怎么办

我们希望对程序员隐藏失败的情况！那么，MR 是否必须从头开始重新运行整个工作？为什么不呢？

MR 只重新运行失败的 Map() 或者 Reduce() 任务。

- 但假设MR运行一个Map两次，一个 Reduce 看到第一次运行的输出。另一个 Reduce 看到了第二次运行的输出？

- 正确性要求重新执行时产生完全相同的输出。

- 所以 Map 和 Reduce 必须是纯确定性的函数。
  -   它们只允许看它们的参数。
  -  没有状态，没有文件I/O，没有交互，没有外部通信。

- 如果你想允许非功能性的 Map 或 Reduce呢？
     - 工作者失败将需要重新执行整个工作。
     - 或者你需要创建同步的全局检查点。

#### Worker 崩溃恢复的细节

- Map worker 崩溃了
  - coodinator 注意到 worker 不再回复 pings
  - coordinator 知道这个 worker 上跑的是哪个 Map 任务 
    - 这些任务的中间输出现在已经丢失，必须重新创建
    - coodinator 告诉其他 worker 运行这些任务
  - 如果 Reduces 已经获取了中间数据，可以省略重新运行。

* Reduce worker 崩溃了
    - 已完成的任务是好的 -- 存储在GFS中，有副本。
    - 协调者在其他工作者上重新启动 worker 的未完成任务。

#### 其他失败问题

* 如果 coodinator 给两个 worker 布置了相同的 Map() 任务怎么办？
    - 也许 coodinator 会错误地认为有一个 worker 死了。
    - 它将只告诉 Reduce worker 其中一个的输出
* 如果 coodinator 给了两个工作者相同的 Reduce() 任务，会怎样？
    - 他们都会试图在 GFS 上写下同一个输出文件!
    - GFS的原子重命名可以防止混合；一个完整的文件将是可见的。

  * 如果一个工作者非常慢，怎么办？
    - 也许是由于软弱的硬件。
    - coodinator 对最后几个 task 的分发给其它服务跑。

  * 如果一个 worker 由于硬件或软件损坏而计算出不正确的输出，怎么办？
    - 太糟糕了! MR 假设 "fail-stop "的CPU和软件。

- coodinator 崩溃了怎么办

#### 当前 MR 状态

- 影响力巨大（Hadoop, Spark, &c）。
- 在谷歌可能已经不再使用了。
     - 被Flume/FlumeJava取代（见Chambers等人的论文）。
     - GFS 被 Colossus（没有好的描述）和 BigTable 取代。

### 总结

MapReduce 单枪匹马地使大集群计算流行起来。
  - 不是最有效或最灵活的。
  + 扩展性好

  + 易于编程 -- 失败和数据移动被隐藏。

      这些在实践中是很好的权衡。我们将在课程后期看到一些更高级的继承者。祝你在实验室里玩得开心!


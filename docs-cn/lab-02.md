---
title: "lab-02"
date: 2021-12-24T14:30:00+08:00
tags: ["6.824", "翻译", "lab"]
---

# 6.824 Lab 2: Raft

[lab 2 源文档地址](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html)

2A 截止日期：3月5  日（星期五）23:59
2B 截止日期：3月12日（星期五）23:59
2C 截止日期：3月19日（星期五）23:59
2D 截止日期：3月26日（星期五）23:59

[协作政策](https://pdos.csail.mit.edu/6.824/labs/collab.html)	//	[提交实验](https://pdos.csail.mit.edu/6.824/labs/submit.html)	//	[Go 设置](https://pdos.csail.mit.edu/6.824/labs/go.html)	//	[指南](https://pdos.csail.mit.edu/6.824/labs/guidance.html)	// 	[论坛](https://piazza.com/mit/spring2021/6824)

## 介绍

这是构建容错 key/value 存储系统的一系列实验中的开始。在这个实验中，您将实现 Raft，一个复制的状态机协议。在下一个实验中，你将在 Raf t之上建立一个 key/value 服务。然后，你将在多个复制的状态机上分片服务，以获得更高的性能。

复制服务通过在多个复制服务器上存储其状态(如数据)的完整副本来实现容错。复制允许服务继续运行，即使它的一些服务器出现故障(崩溃或损坏或不稳定的网络)。挑战在于，故障可能导致复制体持有不同的数据副本。

Raft将客户端的请求组成一个序列，称为日志，并确保所有复制服务器看到相同的日志。每个副本按照日志顺序执行客户端请求，并将它们应用到服务状态的本地副本中。因为所有的活动副本都看到相同的日志内容，所以它们都以相同的顺序执行相同的请求，因此继续具有相同的服务状态。如果服务器出现故障，但随后恢复，Raft 会及时更新其日志。Raft 将继续运行，只要至少大多数服务器是活着的，并可以相互通信。如果没有大多数存活，Raft 将不会有任何进展，但会在大多数能够再次沟通时重新开始服务。

在本实验中，你将把 Raft 实现为一个带有相关方法的 Go对象类型，旨在为一个更大的服务中的作为模块使用。一组 Raft 实例通过 RPC 相互通信来维护复制的日志。你的 Raft 接口将支持无限序列的有编号的命令，也称为日志条目（log entry）。日志条目用索引号（index）编号。具有给定索引的日志条目最终将被提交。此时，你的 Raft 应该将日志条目发送到更大的服务以供执行。

您应该遵循 [Raft 论文](https://raft.github.io/raft.pdf) 的设计，特别注意图 2。您将实现此论文中的大部分内容，包括保存持久状态以及在节点故障后重新启动后读取它。您将不会实现集群成员变更(第6节)。

您可能会发现[学生指南](https://thesquareplanet.com/blog/students-guide-to-raft/)以及关于[锁](https://pdos.csail.mit.edu/6.824/labs/raft-locking.txt)和[并发结构](https://pdos.csail.mit.edu/6.824/labs/raft-structure.txt)的建议很有用。从更广泛的视角，我们可以看一看 Paxos, Chubby, Paxos Made Live, Spanner; Zookeeper, Harp, Viewstamped Replication 和[Bolosky](https://static.usenix.org/event/nsdi11/tech/full_papers/Bolosky.pdf)等人(注:学生指南是几年前写的，特别是2D部分已经改变了。在盲目遵循某个特定的实现策略之前，请确保您理解了它的意义!)。

我们还提供了一个[Raft 交互图](https://pdos.csail.mit.edu/6.824/notes/raft_diagram.pdf)，可以帮助阐明 Raft 代码如何与其上面的层进行交互。

这个实验分四部分完成。你必须在相应的截止日期提交每一部分。

## 入门指南

如果您已经完成了 Lab 1，那么您已经有了实验源代码。如果没有，您可以在Lab 1的[说明](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)中找到通过 git 获取源代码的说明。

我们为你提供框架代码 `src/raft/raft.go`。我们还提供了一组测试，你应该使用它们来推动你的实现，我们将使用它们来对你提交的实验进行评分。测试在`src/raft/test_test.go` 文件中。要启动并运行，请执行以下命令。不要忘记使用 git 来获取最新的软件。

```bash
$ cd ~/6.824
$ git pull
...
$ cd src/raft
$ go test -race
Test (2A): initial election ...
--- FAIL: TestInitialElection2A (5.04s)
        config.go:326: expected one leader, got none
Test (2A): election after network failure ...
--- FAIL: TestReElection2A (5.03s)
        config.go:326: expected one leader, got none
...
$
```

### 代码说明

通过在 `raft/raft.go`中添加代码来实现 Raft。在该文件中，你将找到框架代码，以及如何发送和接收rpc的示例。

你的实现必须支持以下接口，测试和(最终)你的 key/value 服务器将使用该接口。你可以在 `raft.go` 的评论中找到更多细节。

```go
// create a new Raft server instance:
rf := Make(peers, me, persister, applyCh)

// start agreement on a new log entry:
rf.Start(command interface{}) (index, term, isleader)

// ask a Raft for its current term, and whether it thinks it is leader
rf.GetState() (term, isLeader)

// each time a new entry is committed to the log, each Raft peer
// should send an ApplyMsg to the service (or tester).
type ApplyMsg
```

服务调用 `Make(peers,me，…)` 来创建一个 Raft peer 实例。`peers` 参数是 Raft peer 网络标识符数组，用于 RPC 通信。`me`参数是该peer 实例在 peers 数组中的索引。`Start(command)`请求 Raft 启动处理，将该 command 追加到复制的日志中。`Start()` 应该立即返回，而不必等待日志追加完成。该服务希望你的实现能够为每个新提交的日志条目发送一个 `ApplyMsg` 到 `Make()`的`applyCh` channel参数中。
`raft.go` 包含发送一个RPC (`sendRequestVote()`)和处理传入 RPC ( `RequestVote()`)的示例代码。你的 Raft peer 实例们应该使用 `labrpc` Go 软件包（源代码在`src/labrpc`）替换 Go 自带的 RPC 包。测试文件可以告诉 `labrpc` 延迟 RPC请求，重新排序，并丢弃它们以模拟各种网络故障。虽然你可以临时修改 `labrp`，但要确保你的 Raft 能与原始的 `labrpc `一起使用，因为我们会用它来测试和评定你的实验。你的Raft实例之间必须只通过 RPC 通信；例如，它们不允许使用共享的 Go 变量或文件进行通信。

后面的实验建立在这个实验的基础上，所以给自己足够的时间来编写可靠的代码是很重要的。

## 2A: leader 选举（中等难度）

### 任务

实现 Raft 的 leader 选举和心跳机制（`AppendEntries` 没有日志条目的 RPC ），2A 部分目标是选出一个 leader，如果没有故障，leader 将继续担任 leader。如果旧的 leader 故障或者旧 leader 的数据包丢失，新的leader可以接管。运行 `go test -run 2A -race` 来测试你的 2A 部分的代码

### 提示

- 你不能轻易地直接运行你的 Raft 实现; 相反，你应该通过测试来运行它。如运行 `go test -run 2A -race`

- 按照论文的图 2 去实现。在这个部分，你关心的是发送和接收 `RequestVote` rpc，与选举相关的 server 规则，以及与 leader 选举相关的状态
- 在 `raft .go`中为 `Raft struct`添加图 2 中 leader 选举的状态, 你还需要定义一个结构来保存每个日志条目的信息。
- 填入`RequestVoteArgs` 和 `RequestVoteReply `结构。修改 `Make() `来创建一个后台 goroutine，当它有一段时间没有收到其他peer 的消息时，通过发送 `RequestVote rpc`来定期启动 leader 选举。这样，如果已经有了 leader，peer 就会知道谁是 leader，或者自己成为 leader。实现 `RequestVote()` RPC处理程序，这样服务器就可以互相投票。
- 要实现心跳，定义一个 `AppendEntries` RPC结构体(尽管你可能还不需要所有的参数)，并且 leader 需要定期发送这个 rpc。编写一个 `AppendEntries` RPC处理方法，重置选举超时时间，这样当一个服务已经当选时，其他服务器就不会站出来当 leader。
- 确保不同 peer 的选举超时不会总是在同一时间出现，否则所有 peer 将只为自己投票，没有人会成为领导者。
- 测试要求 leader 每秒发送心跳 RPC 的次数不超过10次
- 测试要求你的 Raft 在老 leader 失败后的 5 秒内选出一个新的 leader(如果大多数同伴仍然可以通信)。但是，请记住，如果出现分裂投票(如果分组丢失或 candidate 不幸选择了相同的随机回退时间，就会发生这种情况)，那么 leader 选举可能需要多轮。您必须选择足够短的选举超时(以及心跳间隔)，使选举很可能在不到 5 秒内完成，即使它需要多轮
- 论文的 5.2 节提到了150 到 300毫秒的选举超时。只有当 leader 的心跳频率远远高于 1次/150 ms，这样才有意义。因为测试将你的心跳限制在每秒 10 次，你必须使用一个大于论文中 150 到 300ms 的选举超时，但不能太大，因为这样你可能会在 5 秒内选举不出一个领袖。
- 你可能会发现 Go 的 [rand](https://pkg.go.dev/math/rand) 很有用。
- 你需要编写定期或延迟后采取行动的代码。要做到这一点，最简单的方法是创建一个 goroutine，它带有一个循环，调用 `time.Sleep()`;(请参阅 `Make()`为此目的创建的 `ticker()` goroutine)。不要使用 Go 的 `time.Timer` 或 `time.Ticker`，它们很难正确使用。
- [指导页](https://pdos.csail.mit.edu/6.824/labs/guidance.html)有一些关于如何开发和调试你的代码的提示.
- 如果你的代码难以通过测试，请再次阅读该论文的图 2；leader 选举的全部逻辑分布在图中的多个部分。
- 不要忘记实现`GetState()`。
- 测试在永久关闭一个实例时调用你的 Raft 的 `rf.Kill()`。你可以使用 `rf.killed()` 检查 `Kill()`是否被调用。你可能想在所有的循环中这样做，以避免死的 Raft 实例打印出混乱的信息。
- Go RPC 只发送名称以大写字母开头的结构字段。子结构也必须有大写的字段名（例如，数组中的日志记录字段）。labgob包会对此发出警告；不要忽视这些警告。

在提交第2A部分之前，请确保您通过了2A测试，这样您就可以看到如下内容:

```bash
$ go test -run 2A -race
Test (2A): initial election ...
  ... Passed --   4.0  3   32    9170    0
Test (2A): election after network failure ...
  ... Passed --   6.1  3   70   13895    0
PASS
ok      raft    10.187s
$
```

每个 Passed 行包含五个数字;这些分别是测试时间，Raft peers的数量(通常是3或5)，测试期间发送的RPC的数量，RPC消息的总字节数，以及Raft报告提交的日志条目的数量。你的数字会和这里显示的不一样。如果您愿意，可以忽略这些数字，但是它们可以帮助您健全地检查实现发送的rpc数量。对于所有实验 2、3和 4，如果所有测试(`go test`)花费的时间超过600秒，或者任何单个测试花费的时间超过120 秒，那么评分脚本将不通过您的解决方案

## 2B: 日志（难）

### 任务

实现 Leader 与 Follower 追加日志条目的代码，以便 `go test -run 2B -race` 通过

### 提示

- 运行 `git pull` 得到最新版本的软件
- 你的第一个目标应该是通过`TestBasicAgree2B()`。首先实现 `Start()`，然后编写代码通过 `AppendEntries` rpc 发送和接收新的日志条目，如论文中图 2 所示
- 你需要执行选举限制(论文第5.4.1节)。
- 在早期的 Lab 2B测试中无法达成协议的一个原因可能是，即使 leader 还活着，也要反复举行选举。1. 选举定时器管理中的错误，2. 或者在赢得选举后不立即发送心跳
- 你的代码可能有循环去重复检查某些事件。不要让这些循环在没有暂停的情况下连续执行，因为这将使你的实现慢到无法通过测试。使用 Go 的[条件变量](https://pkg.go.dev/sync#Cond)，或者在每个循环迭代中插入一个 `time.Sleep(10 * time.Millisecond)`
- 为你自己将来的实验帮个忙，写（或重写）代码时要干净利落。想了解更多信息，请重新访问我们的[指导页面](https://pdos.csail.mit.edu/6.824/labs/guidance.html)，其中有关于如何开发和调试代码的提示。
- 如果你的测试失败了，请查看 `config.go`和 `test_test.go`中的测试代码，以更好地了解测试的内容。`config.go` 还说明了测试如何使用 Raft API

如果你的代码运行得太慢，即将进行的实验的测试可能会失败。你可以用时间命令检查你的解决方案使用了多少实时时间和CPU时间。下面是典型的输出。

```
$ time go test -run 2B
Test (2B): basic agreement ...
  ... Passed --   1.6  3   18    5158    3
Test (2B): RPC byte count ...
  ... Passed --   3.3  3   50  115122   11
Test (2B): agreement despite follower disconnection ...
  ... Passed --   6.3  3   64   17489    7
Test (2B): no agreement if too many followers disconnect ...
  ... Passed --   4.9  5  116   27838    3
Test (2B): concurrent Start()s ...
  ... Passed --   2.1  3   16    4648    6
Test (2B): rejoin of partitioned leader ...
  ... Passed --   8.1  3  111   26996    4
Test (2B): leader backs up quickly over incorrect follower logs ...
  ... Passed --  28.6  5 1342  953354  102
Test (2B): RPC counts aren't too high ...
  ... Passed --   3.4  3   30    9050   12
PASS
ok      raft    58.142s

real    0m58.475s
user    0m2.477s
sys     0m1.406s
$
```

`ok raft 58.142s`意味着 Go 在2B 测试所花费的时间是 58.142 秒的实际（挂钟）时间。`user 0m2.477s` 表示代码消耗了 2.477秒 的CPU时间，即实际执行指令的时间（而不是等待或睡眠）, 如果你的解决方案在 2B 测试中使用了超过一分钟的实际时间，或者超过5秒的CPU时间，你可能会在以后遇到麻烦。寻找花费在 sleep 或等 待RPC 超时的时间，在没有睡眠或等待条件或通道消息的情况下运行的循环，或发送大量的RPC。

## 2C: 持久化（难）

如果基于 Raft 的服务重新启动，它应该在它停止工作的地方恢复服务。这要求 Raft 保持持久的状态，在重新启动后仍能存活。论文中的图 2 提及了哪些状态应该持久化。

真正实现会在 Raft 实例状态的每次更改时将其持久化写入磁盘，并在重启后从磁盘读取状态。你的实现不会使用磁盘；相反，它将从一个 `Persister` 对象（见 `persister.go`）保存和恢复持久化状态。任何调用`Raft.Make()` 的人会提供了一个 Persister 对象传进来，它最初持有 Raft 最近的持久化状态（如果有的话）。Raft 应该从那个 Persister 初始化它的状态，并且应该在每次状态改变时使用它来保存它的持久化状态。使用 Persister 的 `ReadRaftState()` 和 `SaveRaftState()`方法。

### 任务

1. 通过添加保存和恢复持久化状态的代码，完成 `raft.go` 中的 `persist()` 和 `readPersist()` 函数。你将需要把状态编码（或 "序列化"）为一个字节数组，以便将其传递给 Persister。使用 `labgob` 编码器；见 `persist()`和 `readPersist()` 的注释。`labgob` 类似于 Go 的 `gob` 编码器，但如果你试图对字段名为小写的结构进行编码，会打印出错误信息
2. 在你的实现改变持久化状态的地方插入对 `persist()` 的调用。一旦你完成了这些，你就应该通过其余的测试。

### 提示

- 运行 `git pull` 得到最新版本的软件
- 许多 2C 测试涉及服务器故障和网络丢失 RPC 请求或回复。这些事件是非确定性的，你可能会很幸运地通过测试，即使你的代码有bug。通常情况下，多运行几次测试就会暴露出这些 bug。
- 您可能需要一次通过多个条目备份 nextIndex 的优化。请看[Raft论文](https://raft.github.io/raft.pdf) 从第7页的底部到第8页的顶部(用灰色线条标记)。论文中的细节很模糊，你需要填补空白，或许可以借助 6.824 raft 的讲座视频。
- 虽然 2C 只要求你实现持久性和快速日志回溯，但 2C 的测试失败可能与你实现的前几个部分有关。即使你稳定地通过了 2A 和 2B 测试，你仍然可能有选举或日志错误，这些错误在2C测试中暴露。

你的代码应该通过所有 2C 测试（如下图所示），以及 2A 和 2B 测试。

```bash
$ go test -run 2C -race
Test (2C): basic persistence ...
  ... Passed --   7.2  3  206   42208    6
Test (2C): more persistence ...
  ... Passed --  23.2  5 1194  198270   16
Test (2C): partitioned leader and one follower crash, leader restarts ...
  ... Passed --   3.2  3   46   10638    4
Test (2C): Figure 8 ...
  ... Passed --  35.1  5 9395 1939183   25
Test (2C): unreliable agreement ...
  ... Passed --   4.2  5  244   85259  246
Test (2C): Figure 8 (unreliable) ...
  ... Passed --  36.3  5 1948 4175577  216
Test (2C): churn ...
  ... Passed --  16.6  5 4402 2220926 1766
Test (2C): unreliable churn ...
  ... Passed --  16.5  5  781  539084  221
PASS
ok      raft    142.357s
$ 
```

在提交之前，最好多次运行测试，并检查每一次运行都打印出PASS。

```bash
$ for i in {0..10}; do go test; done
```

## 2D: 日志压缩

按照你的代码现在的情况，一个重启服务会回放完整的Raft日志，以恢复其状态。然而，对于一个长期运行的服务来说，永远记住完整的Raft 日志是不现实的。相反，你将修改 Raft 以配合节省空间: 服务会不时地持久化存储其当前状态的快照，而 Raft 将丢弃快照之前的日志条目。当一个服务远远落后于领先服务并且必须迎头赶上时，该服务首先从一个快照恢复，然后重放创建快照之后的日志条目。论文中第 7 节概述了方案;你必须设计细节。

您可能会发现，参考[Raft交互图](https://pdos.csail.mit.edu/6.824/notes/raft_diagram.pdf)有助于理解复制的服务和Raft如何通信。

为了支持快照，我们需要在服务（上层应用）和 Raft 库之间建立一个接口。Raft 没有指定这个接口，有几种设计是可能的

- `Snapshot(index int, snapshot []byte)`
- `CondInstallSnapshot(lastIncludedTerm int, lastIncludedIndex int, snapshot []byte) bool`

服务调用 `Snapshot()` 将其状态的快照传递给 Raft。快照包括索引之前的所有信息。这意味着相应的 Raft 实例不需要保存这之前的 log 了。你的 Raft 实现应该尽可能地修剪其日志。你必须修改你的 Raft 代码，以便在操作时只存储日志的尾部。

正如在 Raft 论文中所讨论的，Raft leader 有时必须告诉落后的 Raft 同伴通过安装快照来更新其状态。当这种情况出现时，你需要实现	`InstallSnapshot` RPC发送和处理程序来安装快照。这与 AppendEntries 相反，后者发送日志条目，然后由服务逐一应用。

注意，`InstallSnapshot` rpc 是在 Raft 实例之间发送的，而提供的框架函数 `Snapshot/CondInstallSnapshot` 被用来上层应用与 Raft 通信。当一个 follower 接收并处理一个 `InstallSnapshot` RPC时，它必须使用 Raft 将快照交给服务。通过将快照放在 `ApplyMsg`中，`InstallSnapshot`处理程序可以使用 `applyCh` 将快照发送到服务。服务从`applyCh` 读取数据，并调用带有快照的`CondInstallSnapshot` 来告诉 Raft 服务正在切换到传入的快照状态，并且 Raft 应该同时更新它的日志(参见 config 中的 `applierSnap()`。请查看测试服务是如何做到这一点的)。

如果快照是旧的，`CondInstallSnapshot`应该拒绝安装快照(例如，如果 Raft 处理了快照的 `lastIncludedTerm/lastIncludedIndex`之后的条目)。这是因为 Raft 可能会处理其他 RPC，并在处理 `InstallSnapshot` RPC之后，在服务调用 `CondInstallSnapshot`之前，在applyCh 上发送消息。Raft 回到旧的快照是不允许的，所以旧的快照必须被拒绝。当你实现拒绝快照时，`CondInstallSnapshot` 应该只是返回 false，这样服务就知道它不应该切换到快照。

如果快照是最近的，那么 Raft 应该修剪它的日志，坚持新的状态，返回 true，并且服务应该在处理`applyCh` 上的下一条消息之前切换到快照。

`CondInstallSnapshot`是更新 Raft 和服务状态的一种方式;  服务和 raft 之间的其他接口实现也是可能的。这个特殊的设计允许您的实现检查快照是否必须安装在一个地方，并自动地将服务和Raft切换到快照。你可以用 `CondInstallSnapShot`总是返回true的方式来实现Raft; 如果您的实现通过了测试，您将获得满分。

### 任务

修改你的Raft代码以支持快照：实现 Snapshot、CondInstallSnapshot 和 InstallSnapshot RPC，以及对 Raft 的修改以支持这些（例如，对修剪后的日志进行操作）。当您的解决方案通过 2D 测试和所有 Lab 2 测试时，这个实验就结束了。(注意，实验 3 将比实验 2 更彻底地测试快照，因为实验 3 有一个真正的服务来使用 Raft 的快照。)

### 提示

1. 在单个 `InstallSnapshot` RPC中发送整个快照。不要实现图13中分割快照的偏移机制。
2. Raft 必须丢弃旧的日志条目，以允许 Go 垃圾收集器释放和重用内存;这要求对丢弃的日志项没有可达的引用(指针)。
3. raft 日志不能再使用日志条目的位置或日志的长度来确定日志条目的索引;您将需要使用独立于日志位置的索引方案。
4. 即使日志被修剪了，你的实现仍然需要在 `AppendEntries` RPC 的新条目之前正确发送条目的 term 和索引。这可能需要保存和引用最新快照的lastIncludedTerm/lastIncludedIndex（思考这是否应该被持续保存）。
5. Raft 必须使用 `SaveStateAndSnapshot()` 将每个快照存储在 persister 对象中。
6. lab2 的全套测试（2A+2B+2C+2D）的合理耗时是 8 分钟的真实时间和 1.5分钟的 CPU 时间。
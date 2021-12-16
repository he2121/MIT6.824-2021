---
title: "lab-01"
date: 2021-12-16T14:30:00+08:00
tags: ["6.824", "翻译", "lab"]
---

# 6.824 实验 1: MapReduce

[实验 1 原地址](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)

截止日期：2月26日（星期五）23:59 ET（麻省理工时间）。

[协作政策](https://pdos.csail.mit.edu/6.824/labs/collab.html)	//	[提交实验](https://pdos.csail.mit.edu/6.824/labs/submit.html)	//	[Go 设置](https://pdos.csail.mit.edu/6.824/labs/go.html)	//	[指南](https://pdos.csail.mit.edu/6.824/labs/guidance.html)	// 	[论坛](https://piazza.com/mit/spring2021/6824)

## 介绍

在这个实验室，你将建立一个 MapReduce 系统。你将实现一个调用 Map 和 Reduce 函数并处理读写文件的 worker 程序，以及一个向 worker 分配任务和应对失败的 coodinator 程序。你将构建类似于 [MapReduce 论文 ](http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf) 的东西(注意：实验使用 coordinator，而不是论文的 master)。

## 入门

你需要进行 [Go 设置](https://pdos.csail.mit.edu/6.824/labs/go.html)

你将用 [git](https://git-scm.com/)（一个版本控制系统）获取最初的实验软件。要了解更多关于 git 的信息，请看 [Pro git book](https://git-scm.com/book/en/v2) 或 [git 用户手册](https://mirrors.edge.kernel.org/pub/software/scm/git/docs/user-manual.html)。获取 6.824的实验软件:

```bash
$ git clone git://g.csail.mit.edu/6.824-golabs-2021 6.824
$ cd 6.824
$ ls
Makefile src
$
```

我们在 `src/main/mrsequential.go` 中为你提供了一个简单的顺序 MapReduce 实现。它以单进程程下在同一时间只运行一个 map 或者 reduce worker。我们还为你提供了几个 `MapReduce`应用程序：`mrapps/wc.go`中的单词计数，以及`mrapps/indexer.go`中的文本索引器。你可以按以下方式连续运行字数统计。

```bash
$ cd ~/6.824
$ cd src/main
$ go build -race -buildmode=plugin ../mrapps/wc.go
$ rm mr-out*
$ go run -race mrsequential.go wc.so pg*.txt
$ more mr-out-0
A 509
ABOUT 2
ACT 8
...
```

(注意：如果你没有用 -race 编译，不要使用 -race 运行)

`mrsequential.go` 将其输出在文件 `mr-out-0`中。输入是来自名为 `pg-xxx.txt` 的文本文件。

可以随意借用 `mrsequential.go`的代码。你也应该看看`mrapps/wc.go`，看看 MapReduce 应用代码是什么样子的。

## 你的工作（[中度/难度](https://pdos.csail.mit.edu/6.824/labs/guidance.html)）

你的工作就是实现一个分布式 MapReduce，由两个程序组成，即协调者（coordinator）和工作者（worker）。会有一个coordinator 进程以及一个或多个 worker 进程在并行执行。在真实系统中，worker 会分布在不同的机器上运行，但在这个实验中，你在一台机器上运行这些 worker 即可。这些 worker 会通过 RPC 与 coordinator 通信。

每个 worker 都会向 coordinator：

1. 请求一个任务
2. 从一个或多个文件中读取任务的输入，执行任务
3. 并将任务的输出写入一个或多个文件中。

coodinator 应该做到：如果一个 worker 在合理的时间内（对于这个实验，10秒）没有完成其任务，会将相同的任务交给不同的 worker。

我们已经给了你一点代码，让你开始工作。coordinator 和 worker 的 main 函数在 `main/mrcoordinator.go` 和`main/mrworker.go`中。不要改变这些文件。你应该把你的实现放在 `mr/coordinator.go`、`mr/worker.go`和`mr/rpc.go`中。

下面以 word-count 程序为例：如何运行你的代码。

1. 首先，确保 word-count 插件时新构建的

```bash
$ go build -race -buildmode=plugin ../mrapps/wc.go
```

2. 在 `main`目录下，运行 coordinator

```bas
$ rm mr-out*
$ go run -race mrcoordinator.go pg-*.txt
```

`pg-*.txt`参数是 `coordinator.go` 的输入文件，每个文件对应一个 "分割" 的数据块，是一个 Map 任务的输入。`-race` 标志在运行 go 时带有并发检测器。

3. 在一个或者多个 shel l窗口下，运行一些 worker：

```bash
$ go run -race mrworker.go wc.so
```

4. 当 worker 和 coordinator 都完成后，看看 `mr-out-*` 中的输出。当你完成实验后，输出文件的排序联合应该与顺序输出一致，像这样。

```bash
$ cat mr-out-* | sort | more
A 509
ABOUT 2
ACT 8
...
```

我们在 `main/test-mr.sh`中为你提供了一个测试脚本。该测试检查 wc 和 indexer MapReduce r任务在给定 `pg-xxx.txt`文件作为输入时是否产生正确的输出。

这些测试还可以检查你的实现是否并行地运行 Map 和 Reduce 任务，以及你的实现是否能容错（单个 worker 的崩溃不影响程序的正确运行）。

如果你现在运行测试脚本，它就会挂起，因为 coodinator 不会结束运行

```bash
$ cd ~/6.824/src/main
$ bash test-mr.sh
*** Starting wc test.
```

你可以在 `mr/coordinator.go`的 `Done`函数中把 `ret := false`改为`true`，这样 coodinator 就会立即退出。然后出现：

```bash
$ bash test-mr.sh
*** Starting wc test.
sort: No such file or directory
cmp: EOF on mr-wc-all
--- wc output is not the same as mr-correct-wc.txt
--- wc test: FAIL
$
```

测试脚本期望在名为`mr-out-X`的文件中看到输出，每个 reduce 任务都对应着一个 `mr-out-X` 文件。`mr/coordinator.go`和`mr/worker.go`的空实现并没有产生这些文件（或做很多其他事情），所以测试失败。

当你完成代码后，测试脚本的输出应该看起来像这样。

```bash
$ bash test-mr.sh
*** Starting wc test.
--- wc test: PASS
*** Starting indexer test.
--- indexer test: PASS
*** Starting map parallelism test.
--- map parallelism test: PASS
*** Starting reduce parallelism test.
--- reduce parallelism test: PASS
*** Starting crash test.
--- crash test: PASS
*** PASSED ALL TESTS
$
```

你还会看到来自Go RPC包的一些错误，看起来是

```
2019/12/16 13:27:09 rpc.Register: method "Done" has 1 input parameters; needs exactly three
```

忽略这些信息；把 coodinator 注册作为一个 [RPC server](https://go.dev/src/net/rpc/server.go) 会去检查它的所有方法是否符合 RPC 方法的规范（三个输入参数）。我们知道 `Done()` 这个方法不是一个 RPC 调用。

## 一些规则

- Map 阶段需要把中间文件分为 nReduce 个 bucket 存储，nReduce 对应着总共有 n 个 Reduce 任务。 nReduce 是 `main/mrcoordinator.go`传递给 `MakeCoodinator()` 方法的参数
- worker 的实现应该把第 X 个 reduce 任务的输出放在文件 `mr-out-X` 中

- 一个 `mr-out-X`文件应该包含 Reduce 函数输出的每一行。一行应该用Go“%v %v” 格式生成，用 key 和 value 调用。
- 你修改代码的地方在 `mr/worker.go`，`mr/coordinator.go`，和`mr/rpc.go`三个文件中。你也可以临时修改其他文件进行测试，但要确保你的代码能在原始版本中运行；我们会用原始版本进行测试。
- Worker 应该将 Map产生的中间输出放在当前目录下的文件中，你的 Worker 以后可以在那里读取它们作为 Reduce 任务的输入。
- `main/mrcoordinator.go `希望 `mr/coordinator.go` 实现一个`Done()`方法，当 MapReduce 作业完全完成时返回 true；这时，`mrcoordinator.go`将退出
  - 当工作完全完成时，worker 进程应该退出。实现这一点的一个简单方法是使用 call() 的返回值：如果 worker 未能联系到coordinator ，它可以认为 corrdinator 已经退出了，因为工作已经完成，所以 worker 也可以终止。这取决于你的设计，你可能也会发现有一个 "请退出 "的伪任务，coodinator 可以把它交给 worker。

## 提示

- 你可以从修改 `mr/worker` 的 `worker()` 方法入手：给 coodinator 发送一个获取任务执行的 RPC 请求。然后修改 coodinator，回复一个还没开始 map 任务的文件名。然后修改 `Worker`以读取该文件并调用应用程序 `Map`函数，如`mrsequential.go`中的内容
- 应用程序的 Map 和 Reduce 函数在运行时使用 Go 插件包加载，文件名以.so结尾。
- 如果你改变了`mr/`目录中的任何东西，你可能需要重新构建你使用的任何 MapReduce 插件，用类似 `go build -race -buildmode=plugin .../mrapps/wc.go` 这样的命令。
- 这个实验室依赖于 worker 共享一个文件系统。当所有 worker 都在同一台机器上运行时，这很简单，但如果 worker 在不同的机器上运行，就需要一个像 GFS 这样的全局文件系统。
- 一个合理的中间文件的命名惯例是 `mr-X-Y`，其中 X 是 Map 任务编号，Y 是 reduce 任务编号。

- worker 的 map 任务代码需要一种方法，将中间的 key/value 对以一种可以在 reduce 任务中正确读回的方式存储在文件中。一种可能性是使用 Go 的 `encoding /json` 包。将key/value 对写入 JSON 文件。

```go
  enc := json.NewEncoder(file)
  for _, kv := ... {
    err := enc.Encode(&kv)
```

从文件读：

```go
  dec := json.NewDecoder(file)
  for {
    var kv KeyValue
    if err := dec.Decode(&kv); err != nil {
      break
    }
    kva = append(kva, kv)
  }
```

- 你的 worker 的 map 部分可以使用 `ihash(key)` 函数（在worker.go中）来为给定的 key 归属在指定的 reduce 任务中。
- 你可以从 `mrsequential.go` 中窃取一些代码，例如：用于读取 Map 输入文件，在 Map 和 Reduce 之间排序中间文件的 key/value 对，以及将Reduce的输出存储在文件中。
- coodinator 作为 RPC server，将是并发的；别忘了锁定共享数据。
- 使用 Go 的 race 检测器，使用 `go build -race` 和 `go run -race`。`test-mr.sh` 默认使用竞赛检测器运行测试。
- worker 有时需要等待，例如，在最后一个 map 任务完成之前，reduce 任务不能开始。
  - 一种可能是 worker 定期向 coodinator 请求工作，在每次请求之间用 `time.Sleep()`睡觉。
  - 另一种可性是 coordinator 中的相关 RPC 处理程序有一个等待的循环，可以使用 `time.Sleep()` 或 `sync.Cond`。Go 对每一个 RPC 请求都使用单独的线程，因此一个 handler 在等待不会妨碍 coodinator 处理其它 RPC
- coodinator 无法可靠地区分崩溃、活着但由于某种原因停滞不前、以及正在执行但速度太慢而无法发挥作用的 worker。你能做的最好就是让 coodinator 等待一定的时间，然后放弃，把任务重新发给另一个 worker。在这个实验中，让 coodinator 等待10秒钟；之后 coodinator应该假定该 worker 已经死亡（当然，它可能没有死亡）。
- 如果你选择实现备份任务（第3.6节），请注意，当 worker 执行任务而不崩溃时，我们会测试你的代码不会安排无关的任务。备份任务应该只在某个相对较长的时间段（如10s）后安排。
- 为了测试崩溃恢复，你可以使用 `mrapps/crash.go` 应用程序插件。它在 Map 和 Reduce 函数中随机退出。
- 为了确保在崩溃的情况下没有人观察到部分写入的文件，MapReduce 论文中提到了使用临时文件的技巧，一旦完全写入就原子化地重命名。你可以使 `ioutil.TempFile`来创建一个临时文件，使用 `os.Rename` 来原子化地重命名它。
- `test-mr.sh`运行子目录 mr-tmp 中的所有进程，所以如果出了问题，你想看中间文件或输出文件，就看那里。你可以修改`test-mr.sh`，使其在测试失败后退出，这样脚本就不会继续测试（并覆盖输出文件）。
- `test-mr-many.sh` 提供了一个运行`test-mr.sh` 的基本脚本，并带有超时功能（这就是我们要测试你的代码的方式）。它把运行测试的次数作为一个参数。你不应该同时运行几个`test-mr.sh` 实例，因为协调器会重复使用同一个套接字，造成冲突。

## 无学分挑战练习

1. 实现你自己的 MapReduce 应用（见mrapps/*中的例子），例如，分布式 Grep（MapReduce论文第2.3节）。
2. 让你的 MapReduce coordinator 和 worker 在不同的机器上运行，就像他们在实践中一样。你需要将你的 RPC 设置为通过TCP/IP 而不是 Unix 套接字进行通信（见Coordinator.server()中的注释行），并使用共享文件系统读/写文件。例如，你可以通过 ssh 进入麻省理工学院的多个 Athena 集群机器，这些机器使用AFS来共享文件；或者你可以租用几个AWS实例，使用S3进行存储。


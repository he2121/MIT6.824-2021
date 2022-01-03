# 6.824 2020 Lecture 7: Raft (2)

## Raft Log(Lab 2B)

### 只要 Leader 继续维持

- client 只与 leader 交互
- client 看不到 follower/candidate 的状态或者日志

### Leader 更换时，事情会变得很有趣

例如，在老 leader 失败后，如何在不出现异常的情况下更换 leader？

- 分叉的副本、丢失的操作、重复的操作等等

### 我们想要确保什么?

如果服务器已将给定索引处的日志项应用到其状态机，则其他服务器将不会为同一索引应用不同的日志项（论文图 3 状态机安全性）

如果服务器在操作上有分歧，那么 leader 的改变可能会改变 client 的可见状态，这就违反了我们模仿单一服务器的目标。

例子

    S1: put(k1,v1) | put(k1,v2) 
    S2: put(k1,v1) | put(k2,x) 

不能让两者都执行他们的第 2 个日志条目!

### 崩溃后的日志怎么会不一致？

leader 在向所有人发送最后的 AppendEntries 之前崩溃了

    S1: 3
    S2: 3 3
    S3: 3 3

更糟糕的是:日志可能在同一个条目中有不同的命令!

在一系列的领导崩溃之后，例如。

```
        10 11 12 13  <- log entry #
    S1:  3
    S2:  3  3  4
    S3:  3  3  5
```

### Raft 通过让 Follower 采用 Leader 的日志来迫使达成一致

例如（还是上面例子）：

1. S3 被选为 term 6 的 Leader

2. S3 发送带有日志条目 13 的 `AppendEntries` 的 RPC

- `prevLogIndex=12`
- `prevLogTerm=5`

3. S2 回复为 false（AppendEntries步骤2）
4. S3 将 NextIndex[S2] 递减到12

5. S3 发送`AppendEntries` 带有 12+13, prevLogIndex=11, prevLogTerm=3

6. S2 删除其条目12

7. S1也有类似的情况，但是 S3 需要做进一步的后退

回滚的结果

- 每个活着的 Follower 都会删除与 leader 不同的日志的尾部
- 然后每个活着的 Follower 都接受 leader 在该点之后的条目
- 现在，Follower 的日志与 leader 的日志相同

### 提问：为什么可以丢失 S2 的 index=12 term=4 条目？

这个日志条目没有被提交（复制到大多数机器上）

新 leader 能否收回上届任期结束时的**已提交**日志条目？也就是说，新 leader 的日志中会不会缺少一个已提交的条目？

这将是一场灾难 -- 老 leader 可能已经对客户说 "yes "了

所以Raft 需要确保当选 leader 拥有所有已提交的日志条目

### 为什么不选举日志最长的服务器作为 leader？

```
  example:
    S1: 5 6 7
    S2: 5 8
    S3: 5 8
```

首先，这种情况会发生吗？ 如何发生？

- S1 在第 term 6 是 Leader；崩溃+重启；在第 term 又成为 leader；崩溃并下线。两次都是在只对自己的日志进行追加后就崩溃了。

- 问题：S1 在第 term 7 崩溃后，为什么 S2/S3 不会选择 6 作为下一个 term？
  - 下一届将是 8，因为 S2/S3 中至少有一个在投票时得知了 7
- S2 成为 term 8 的 leader， 只有 S2+S3 存活，然后都崩溃了

现在所有实例重启，现在谁会当选为 Leader？
  - S1的日志最长，但 term 8 的日志可能已经提交了
  - 所以新 leader 只能是 S2 或 S3 中的一个
  - 也就是说，该规则不能仅仅是 "最长的日志"。

论文 5.4.1 的结尾解释了 "选举的限制": `RequestVote` handler 只投票给"至少是最新的"的 candidate

- candidate 在最后一条日志中拥有更高的 term
- 或者同样的 term，但是有更长的日志

所以:
 
- S2 和 S3 不会给 S1 投票
- S2 和 S3 两者之间可以投票

所以只有 S2 或 S3 可以成为 leader，将迫使 S1放弃 6,7 的 log

这是好的行为，因为6,7 不在多数服务器上 --> 没有提交 --> 从未发送给 client 回复 --> client 将重新发送被丢弃的命令

"至少是最新的" 规则确保新 Leader 的日志包含了所有可能已提交的条目 所以新 Leader 不会回滚任何已提交的操作

### 问题：论文中图 7 leader 死了，谁会成为新 Leader

![image-20211231153526138](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20211231153526138-2021-12-31.png)

一些日志条目


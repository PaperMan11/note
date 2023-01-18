## 1 Raft 简介
[In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
[中文版](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)
### 1.1 Raft 角色
Raft 是一个用于管理日志一致性的协议。它将分布式一致性分解为多个子问题：Leader 选举（Leader election）、日志复制（Log replication）、安全性（Safety）、日志压缩（Log compaction）等。同时，Raft 算法使用了更强的假设来减少了需要考虑的状态，使之变的易于理解和实现。Raft 将系统中的角色分为领导者（Leader）、跟从者（Follower）和候选者（Candidate）：
- **Leader**：负责日志的同步管理，处理来自客户端的请求，与 Follower 保持 heartBeat 的联系；
- **Follower**：响应 Leader 的日志同步请求，响应 Candidate 的邀请投票请求，以及把客户端请求到 Follower 的事务转发给 Leader。
- **Candidate**：负责选举投票，集群刚启动或者 Leader 宕机时，状态 Follower 的节点将转化为 Candidate 并发起选举，胜出选举后（获得超过半数节点的投票），从 Candidate 转换为 Leader 状态。

Raft 算法角色状态转换如下：
![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/01_role_state.webp)

### 1.2 Term
Raft 算法将时间划分成为任意不同长度的任期（term）。任期用连续的数字进行表示。每一个任期的开始都是一次选举（election），一个或多个候选人会试图成为领导人。如果一个候选人赢得了选举，它就会在该任期的剩余时间担任领导人。在某些情况下，选票会被瓜分，有可能没有选出领导人，那么，将会开始另一个任期，并且立刻开始下一次选举。Raft 算法保证在给定的一个任期最多只有一个领导人。
![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/02_term.png)

### 1.3 RPC
Raft 算法中服务器节点之间通信使用 RPC，并且基本的一致性算法只需两种类型的 RPC，为了在服务器之间传输快照增加了第三种 RPC。
- **RequestVote RPC**：用于选举投票。
- **AppendEntries RPC**：Leader 的 heartbeat 和 日志复制。
- **InstallSnapshot RPC**：Leader 用来发送快照给落后太多的 Follower。

## 2 Leader 选举

### 2.1 Leader 选举过程
Raft 使用心跳（heartbeat）触发 Leader 选举。当服务器启动时，初始化为 Follower。Leader 向所有 Follower 周期性的发送 heartbeat。如果 Follower 在选举超时时间内没有收到 Leader 的 heartbeat，就会等待一段随机时间后发起 Leader 选举。

每一个 Follower 都有一个时钟，是一个随机的值，表示的是 Follower 等待成为 Leader 的时间，谁的时钟先跑完，则发起 Leader 选举。

Follower 将其**当前 term 加一**然后转换为 Candidate。它首先给自己投票并且给集群中的其他服务器发送 RequestVote RPC。结果有以下三种情况：
- 赢得多数的选票，成功选举为 Leader；
- 收到了 Leader 的消息，表示有其它服务器已经抢先当选了 Leader；
- 没有服务器赢得多数选票，Leader 选举失败，等待选举时间**超时后发起下一轮选举**。

![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/03_election.jpg)

### 2.2 Leader 选举的限制
在 Raft 协议中，所有的日志条目都只会从 Leader 节点 Follower 节点写入，且 Leader 节点上的日志只会增加，不会删除或覆盖。

这意味着 Leader 节点必须包含所有已经提交的日志，既能被选举为 Leader 节点一定需要包含所有的已经提交的日志（时间最新）。因为日志只会从 Leader 向 Follower 传输，所以如果被选举出的 Leader 缺少已经 Commit 的日志，那么这些已经提交的日志就会丢失，显然这是不符合要求的。

这就是 Leader 选举的限制：能被选举成为 Leader 的节点，**一定包含了所有已经提交的日志条目**（在 **4 安全性**详细说明）。

## 3 日志同步

### 3.1 日志复制过程
Leader 选出后，就开始接收客户端的请求。Leader 把请求作为日志条目（Log entries）加入到它的日志中，然后并行的向其他服务器发起 `AppendEntries RPC` 复制日志条目。当这条日志被复制到大多数服务器上，Leader 将这条日志应用到它的状态机并向客户端返回执行结果。
1. 客户端的每一个请求都**包含**被复制状态机执行的**指令**；
1. Leader 把这个指令作为一条新的日志添加到日志中，然后并行发起 RPC 给其他的服务器，让他们复制这条信息；
1. 假如这条日志被安全的复制，Leader 就应用这条日志到自己的状态机中（commit），并返回给客户端；
1. 如果 Follower 宕机或者运行缓慢或者丢包，Leader 会不断的重试，直到所有的 Follower 最终都复制了所有的日志条目。

![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/04_log_replication.jpg)

简而言之，Leader 选举的过程是：
1. 增加 term 号；
1. 给自己投票；
1. 重置选举超时计时器；
1. 发送请求投票的 RPC 给其它节点。

### 3.2 日志组成
日志由**有序编号（log index）**的日志条目组成。每个日志条目包含它被创建时的**任期号（term）**和用于状态机执行的**命令**。如果一个日志条目被复制到大多数服务器上，就被认为可以提交了（commit）。

![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/08_log_entry.png)

上图显示，共有 8 条日志，提交了 7 条。提交的日志都将通过状态机持久化到磁盘中，防止宕机。

### 3.3 日志的一致性

![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/05_log_replication.jpg)

**（1）日志复制的两条保证**

- 如果不同日志中的两个条目有着**相同的索引和任期号**，则它们所存储的命令时相同的（原因：Leader 在一个 term 内在给定的一个 log index 最多创建一条日志条目，同时该条目在日志中的位置也从来不会改变）。
- 如果不同日志中的两个条目有着**相同的索引和任期号**，则它们之前的所有条目都是完全一样的（原因：每次发送 `AppendEntries RPC` 时，Leader 会把新日志条目紧接着之前条目的 log index 和 term 都包含在里面。如果 Follower 发现和自己的日志不匹配，那么就拒绝接收这条日志，这个称之为一致性检查）。

**（2）日志不正常的情况**

一般情况下，Leader 和 Follower 的日志保持一致，因此 `AppendEntries RPC` 一致性检查通常不会失败。然而，Leader 崩溃可能会导致日志不一致：**旧的 Leader 可能没有完全复制完日志中的所有条目**。

![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/06_log_replication.jpg)

上图阐述了一些 Follower 可能和新的 Leader 日志不同的情况。一个 Follower 可能会丢失掉 Leader 上的一些条目，也可能包含一些 Leader 没有的条目，也有可能两者都会发生。丢失的或者多出来的条目可能会持续多个任期。

**（3）如何保证日志的正常复制**

Leader 通过强制 Follower 复制它的日志来处理日志的不一致，Follower 上的不一致的日志会被 Leader 日志覆盖。**Leader 为了使 Follower 的日志同自己的一致，Leader 需要找到 Follower 同它的日志一致的地方，然后覆盖 Follower 在该位置之后的条目。**

具体的操作是：Leader 会从后往前试，每次 `AppendEntries RPC` 失败后尝试前一个日志条目，直到成功找到每个 Follower 的日志一致位置点（基于上述的两条保证），然后向后逐条覆盖 Follower 在该位置之后的条目。

总结一下就是：当 Leader 和 Follower 日志冲突的时候，Leader 将校验 Follower 最后一条日志是否和 Leader 匹配，如果不匹配，将递减查询，直到匹配，匹配后，删除冲突的日志。这样就实现了主从日志的一致性。

## 4 安全性
Raft 增加了如下两条限制以保证安全性：
**（1）** 用有**最新的已提交的 log entry 的 Follower** 才有资格成为 Leader。

这个保证是在 `RequestVote RPC` 中做的，Candidate 在发送 `RequestVote RPC` 时，要带上自己最后一条日志的 term 和 log index，其他节点收到消息时，如果发现自己的日志比请求中携带的更新，则拒绝投票。日志比较的原则是：如果本地的最后一条 log entry 的 term 更大，则 term 大的更新；如果 term 一样大，则 log index 更大的更新。

**（2）** Leader 只能推进 commit index 来提交当前 term 的已经复制到大多数服务器上的日志，旧 term 日志的提交要等到提交当前 term 的日志来间接提交（log index 小于 commit index 的日志被间接提交）。

之所以要这样，是因为可能会出现已提交的日志又被覆盖的情况，如下所示：

![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/07_safety.jpg)

1. 在阶段 a，term 为 2，S1 是 Leader，且 S1 写入日志（term，index）为（2，2），并且日志被同步写入了 S2；
1. 在阶段 b，S1 离线，触发一次新的选主，此时 S5 被选为新的 Leader，此时系统 term 为 3，且写入了日志（term，index）为（3，2）；
1. S5 尚未将日志推送到 Follower 就离线了，进而触发了一次新的选主，而之前离线的 S1 经过重新上线后被选中变成 Leader，此时系统 term 为 4，此时 S1 会将自己的日志同步到 Follower，按照上图就是将日志（2，2）同步到了 S3，而此时由于该日志已经被同步到了多数节点（S1，S2，S3），因此，此时日志（2，2）可以被提交了（未提交宕机了）。
1. 在阶段 d，S1 又下线了，触发一次选主，而 S5 有可能被选为新的 Leader（这是因为S5可以满足作为主的一切条件：1. term = 5 > 4，2. 最新的日志为（3，2），比大多数节点（如S2/S3/S4的日志都新），然后 S5 会将自己的日志更新到 Follower，于是 S2、S3 中已经被提交的日志（2，2）被截断了。

增加上述限制后，即使日志（2，2）已经被大多数节点（S1、S2、S3）确认了，但是它不能被提交，因为它是来自之前 term（2）的日志，直到 S1 在当前 term（4）产生的日志（4， 4）被大多数 Follower 确认，S1 方可提交日志（4，4）这条日志，当然，根据 Raft 定义，（4，4）之前的所有日志也会被提交。此时即使 S1 再下线，重新选主时 S5 不可能成为 Leader，因为它没有包含大多数节点已经拥有的日志（4，4）。

## 5 日志压缩
在实际的系统中，不能让日志无限增长，否则系统重启时需要花很长的时间进行回放，从而影响可用性。Raft 采用对整个系统进行 snapshot 来解决，snapshot 之前的日志都可以丢弃（以前的数据已经落盘了）。

每个副本独立的对自己的系统状态进行 snapshot，并且只能对已经提交的日志记录进行 snapshot。

![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/09_snapshot.png)

**snapshot 包含以下内容：**
- 日志元数据，最后一条已提交的 log entry 的 log index 和 term。这两个值在 snapshot 之后的第一条 log entry 的 `AppendEntries RPC` 的完整性检查的时候会被用上。
- 系统当前状态。

当 Leader 要发给某个日志落后太多的 Follower 的 log entry 被丢弃，Leader 会将 snapshot 发给 Follower。或者当新加进一台机器时，也会发送 snapshot 给它。发送 snapshot 使用 `InstalledSnapshot RPC`。

做 snapshot 既不要做的太频繁，否则消耗磁盘带宽，也不要做的太不频繁，否则一旦节点重启需要回放大量日志，影响可用性。推荐当日志达到某个固定的大小做一次 snapshot。

做一次 snapshot 可能耗时过长，会影响正常日志同步。可以通过使用 copy-on-write 技术避免 snapshot 过程影响正常日志同步。

## 6 成员变更

[6 Cluster membership changes](https://raft.github.io/raft.pdf)

## 7 总结
Raft 算法各节点维护的状态：

![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/state.jpg)

Leader 选举：

![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/request_vote_rpc.jpg)

日志同步：
![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/appendentries_rpc.jpg)

Raft 状态机：
![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/rule_for_servers.jpg)

snapshot rpc：
![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/install_snapshot_rpc.jpg)

## 8 一些面试题
1、Raft分为哪几个部分？
: 主要是分为leader选举、日志复制、日志压缩、成员变更等。

2、Raft中任何节点都可以发起选举吗？
: Raft发起选举的情况有如下几种：
    1. 刚启动时，所有节点都是follower，这个时候发起选举，选出一个leader；
    1. 当leader挂掉后，时钟最先跑完的follower发起重新选举操作，选出一个新的leader；
    1. 成员变更的时候会发起选举操作。

3、Raft中选举中给候选人投票的前提？
: Raft确保新当选的Leader包含所有已提交（集群中大多数成员中已提交）的日志条目。这个保证是在RequestVoteRPC阶段做的，candidate在发送RequestVoteRPC时，会带上自己的last log entry的term_id和index，follower在接收到RequestVoteRPC消息时，如果发现自己的日志比RPC中的更新，就拒绝投票。日志比较的原则是，如果本地的最后一条log entry的term id更大，则更新，如果term id一样大，则日志更多的更大(index更大)。

4、Raft网络分区下的数据一致性怎么解决？
: 发生了网络分区或者网络通信故障，使得Leader不能访问大多数Follwer了，那么Leader只能正常更新它能访问的那些Follower，而大多数的Follower因为没有了Leader，他们重新选出一个Leader，然后这个Leader来接受客户端的请求，如果客户端要求其添加新的日志，这个新的Leader会通知大多数Follower。如果这时网络故障修复 了，那么原先的Leader就变成Follower，在失联阶段这个老Leader的任何更新都不能算commit，都回滚，接受新的Leader的新的更新（递减查询匹配日志）。

5、Raft数据一致性如何实现？
: 主要是通过日志复制实现数据一致性，leader将请求指令作为一条新的日志条目添加到日志中，然后发起RPC 给所有的follower，进行日志复制，进而同步数据。

6、Raft的日志有什么特点？
: 日志由有序编号（log index）的日志条目组成，每个日志条目包含它被创建时的任期号（term）和用于状态机执行的命令。

7、Raft和Paxos的区别和优缺点？
: Raft的leader有限制，拥有最新日志的节点才能成为leader，multi-paxos中对成为Leader的限制比较低，任何节点都可以成为leader。
: Raft中Leader在每一个任期都有Term号。

8、Raft prevote机制？
: Prevote（预投票）是一个类似于两阶段提交的协议，第一阶段先征求其他节点是否同意选举，如果同意选举则发起真正的选举操作，否则降为Follower角色。这样就避免了网络分区节点重新加入集群，触发不必要的选举操作。

: ![img](./%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E6%9C%AC%E7%90%86%E8%AE%BA.assets/prevote.png)

[更多参考](https://cloud.tencent.com/developer/article/2168468)
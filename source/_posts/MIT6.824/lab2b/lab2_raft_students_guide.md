---
title: MIT6.824之lab2_raft_students_guide
categories:
- 分布式
- MIT6.824
---

- [二.Students' Guide to Raf文档](#二students-guide-to-raf文档)
  - [背景](#背景)
  - [实现raft](#实现raft)
    - [重要的细节](#重要的细节)
  - [Debugging Raft](#debugging-raft)
    - [活锁](#活锁)
    - [不正确的RPC](#不正确的rpc)
    - [没有按照论文的理论实现raft](#没有按照论文的理论实现raft)
    - [term混乱(term不稳定)](#term混乱term不稳定)
    - [优化](#优化)
  - [Applications on top of Raft](#applications-on-top-of-raft)
  - [AppendIndex](#appendindex)


# 二.Students' Guide to Raf文档

> https://thesquareplanet.com/blog/students-guide-to-raft/ 原文链接

在过去几个月里，我成为了MIT6.824的一名助教.以往这门课程的实验都是基于paxos实现一致性算法,然后今年(2016),我们决定使用raft,raft的设计理念是"理解门槛低"，并且我们也希望这项决定使同学们的生活更简单[在过去的几个月里，我一直是麻省理工学院6.824分布式系统课的教学助理。这门课传统上有一些建立在Paxos共识算法上的实验，但今年，我们决定改用Raft。Raft是 "设计成易于理解的"，我们希望这一改变能使学生的生活更轻松]

这篇实验指南,对应着"教师教学指南"，印证着我们实验室与raft的journey，也希望对学生更好的理解raft内部运行机制，实现raft分布式协议起到帮助,如果你正在寻找Raft和Paxos的对比，或者raft的教学(pedagogical)分析,请前去阅读《教师教学指南》.文章底部包含一些学生提问的关于raft的共性问题的问题列表。如果你遇到的问题不在那些问题之内，请前去检查Q&A系统，这篇文章非常的长,但是它提出的所有点，都是现实中学生(助教)碰到过的问题，这是一篇值得阅读的文章。


## 背景

开始深入研究raft之前,一些前情提要或许是有用的，6.824过去有一些基于用golang实现的Paxos实验，之所以选择go是因为它易于理解和学习的，它非常适合实现高并发，分布式应用(goroutine是特别便利的)，通过这门课程的四门实验课程，学生构建一个容错的，共享key-value存储系统，第一个实验让你构建一个基于log的一致性算法库,第二门实验在一致性算法库上添加了key-value的存储，第三们实验课是实验构建共享的key-value容错集群,使用共享master节点解决集群成员变更，同时第四们实验,学生不得不实现失败和恢复机制，包括磁盘完整和磁盘不完整的情况这个实现曾作为学生默认的最后实验

这些年我们决定使用raft重写所有mit6.824的实验，前三个实验跟以前是相同的,但是删除了第四个已经在raft中实现了关于失败与持久化的实验,这篇文章主要讨论了我们第一个实验的经验，一个直接跟raft相关的的实验，虽然我们也会接触到构建raft之上的应用层,(第二门实验)。

如果想了解raft的简单的运行逻辑，这个[web site](https://raft.github.io/)网站演示的raft协议是最好的文字材料.

raft是一种被设计为容易理解的一致性算法,在容错,性能方面等价于paxos算法,他们的不同点是raft将问题解耦为相对独立的问题,它也在实际的系统中清晰的解决了很多实际问题.我们也希望跟多的读者实现raft,以及构建更多的基于raft一致性系统

这个网站的可视化raft协议提供了很好的主要组件的概括,论文提供了更直观的说明为什么需要这些组件,如果你还没有阅读raft extend 论文,请有限阅读那篇论文然后再阅读这份<<raft指南>>,由于我假设你对raft有了个大致的了解(ps:这段要翻译成"你对raft有一定程度的熟悉").

正如其他的分布式一致性协议一样,协议细节有很多的"坑",在稳定的状态下,没有节点失效,raft协议就很好理解,可以很直观的理解,可以从上面的可视化网站了解其运行机制,假设没有节点失效,一个leader会最终被选出来,客户端所有发给raft的请求都会最终发送给follower,但是当网络出现通信延迟,网络分区,节点宕机等等假设发生,但是在特定情况下,我们看到有很多bugs一次次的重复出现,由于直接的错误理解和疏忽,当阅读论文的时候,这不是raft独有的,而是所有提出正确复杂分布式系统都存在的问题

## 实现raft

raft论文的figure2是最好的guide，figure2指定每个RPC的工作的方式，当你读完figure2就可以着手实现raft，但是问题也就接踵而至。实际上，figure2是极其严格的，论文中有些叙述是must而不是should，举个例子，你也会有理由重置election timer, 无论什么时候收到 Request vote and Append entry RPC,这两个RPC说明，其他的server其中一个是leader，另一个是candidate在尝试成为leader，这不允许人为干预或者推理。
> If election timeout elapses without receiving AppendEntries RPC from current leader or granting vote to candidate: convert to candidate.

他们之间是有很大区别的，之前的实现可能有某些特定场景下有严重的活性问题


### 重要的细节

raft会把没有entry的appendEntry的RPC视为hearbeat,很多同学假设AppendEntry RPC是某种特殊的RPC,尤其是很多人是指简单的reset计时器，然后返回success，而不进行任何的冲突检测，接受accept就承认follower承认没有冲突，leader根据返回的response来提交logEntry,另外一个问题就是，follower根据收到冲突的prevLogIndex，就会删掉那个点的log，然后吧entries直接进行复制。
> If an existing entry conflicts with a new one (same index but different terms), delete the existing entry and all that follow it.

这里的if很关键。如果跟随者拥有领导者发送的所有条目，跟随者必须不截断其日志。任何跟随领导者发送的条目的元素都必须被保留。这是因为我们可能从领导者那里收到了一个过时的AppendEntries RPC，而截断日志将意味着 "收回"我们可能已经告诉领导者我们的日志中的条目。

## Debugging Raft

第一版的实现肯定是问题百出,我们需要慢慢的迭代实现,问题大致有如下几点，通常有如下四种主要的bugs`活锁`，`不正确或者不完整的RPC`,`fail_follow_rule`,`term confusion`,还有`死锁`

### 活锁

当你的系统活锁时，你系统中的每个节点都在做一些事情，但你的节点集体处于这样一种状态，没有任何进展。这种情况在Raft中很容易发生，尤其是当你没有虔诚地遵循图2时。有一种活锁情况特别经常出现；没有选举领导人，或者一旦选举了领导人，其他节点又开始选举，迫使最近当选的领导人立即退位。  

出现这种情况的原因有很多，但有几个错误是我们看到无数学生犯的。

- 确保你在图2说的时候准确地重置你的选举计时器。具体来说，你只应该在以下情况下重启你的选举计时器：a）你从当前的领导者那里得到一个AppendEntries RPC（即，如果AppendEntries参数中的term已经过时，你不应该重启你的计时器）；b）你正在开始一个选举；或者c）你授予另一个candidate(requestVote RPC)一个投票。最后一种情况在不可靠的网络中尤其重要，因为在这种网络中，跟随者很可能有不同的日志；在这些情况下，你最终往往只有少数服务器，而大多数服务器都愿意为其投票。如果你每当有人要求你为他投票时就重置选举计时器，这就使得一个有过时日志的服务器和一个有较长日志的服务器同样有可能站出来。事实上，由于具有足够最新的日志的服务器太少，这些服务器很不可能在正常的情况下举行选举而当选。如果按照图2的规则，拥有较多最新日志的服务器不会被过时的服务器的选举打断，因此更有可能完成选举，成为领导者。
- 按照图2的指示，你应该何时开始选举。特别要注意的是，如果你是一个候选人（即，你目前正在进行选举），但选举计时器启动了，你应该开始另一次选举。这一点很重要，可以避免系统因RPC的延迟或放弃而停滞。
- 在处理传入的RPC之前，请确保你遵循 "服务器规则 "中的第二条规则。第二条规则指出。

> If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)  

例如，如果你已经在当前任期内投票，而传入的RequestVote RPC的任期比你高，你应该首先下台，采用他们的任期（从而重新设置 votedFor），然后处理RPC，这将导致你授予投票权

### 不正确的RPC

尽管图2清楚地说明了每个RPC处理程序应该做什么，但一些细微之处仍然容易被忽略。以下是我们反复看到的一些情况，你应该在你的实现中注意这些情况.  

- 发现不对逻辑的RPC就尽快返回false
- 发现follower比leader日志少了，follower就返回false，然后将最新index更新给leader的nextIndex
- leader即使没有发送entry，也要检查prevLogIndex
- AppendEntries的最后一步（#5）中的min是必要的，它需要用最后一个新条目的索引来计算。仅仅让应用lastApplied和commitIndex之间的日志内容的函数在到达日志的末尾时停止，是不够的。这因为你的日志中可能会有与领导者的日志不同的条目，在领导者发给你的条目之后（这些条目都与你的日志中的条目一致）。因为 #3 决定了你只在有冲突的条目时才截断你的日志，那些条目不会被除，如果 leaderCommit 超出了领导者发给你的条目，你可能会应用不正确的条目。发送高leader commit index和空的entry的RPC就会导致apply出现问题
- 要完全按照第5.4节中描述的方式实现 "最新日志"检查。老老实实的实现，不要只只检查长度!

### 没有按照论文的理论实现raft

虽然Raft论文对如何实现每个RPC处理程序非常明确，但它也没有对一些规则和不变因素的实现进行说明。这些都列在图2右侧的 "服务器规则 "部分。虽然其中一些规则是不言自明的，但也有一些需要非常仔细地设计你的应用程序，使其不违反规则.

- 在任何阶段`commitIndex>lastApplied`你都可以直接`apply`log到上层状态机,提交日志的时候不持有锁或者设置数据保护区，保证其他程序不apply日志
- 将`commitIndex>lastApplied`解耦,每次sentout心跳的时候检查`commitIndex`你必须要等`appendlog`动作完成
- `AppendEntries`RPC并不是因为log不一致被reject,应该立刻降级为follower,不要更新`nextIndex`，如果这个时候立刻选举你可能会面对数据竞争的问题
- `commitIndex`不能设置为旧的term，你一定要checklog[N].Term ==currentTerm.这是因为leader不知道follower是否在当前任期提交了日志,Figure 8会详细阐述这个问题

一个常见的混淆是`nextIndex`和`matchIndex`之间的区别。特别是，你可能会观察到`matchIndex` = `nextIndex` - 1，而干脆不实现`matchIndex`。这是不安全的。虽然nextIndex和matchIndex通常在同一时间被更新为类似的值（具体来说，nextIndex = matchIndex + 1），但两者的作用完全不同。它通常是乐观的（我们分享一切），并且只在消极的反应中向后移动。例如，当一个领导者刚刚当选时，`nextIndex`被设置为日志末尾的索引指数。在某种程度上，`nextIndex`是用于性能的--你只需要将这些东西发送给这个peer。  

`matchIndex`是用于安全的。`matchIndex`不能被设置为一个太高的值，因为这可能会导致`commitIndex`被向前移动得太远。这就是为什么`matchIndex`被初始化为-1（也就是说，我们不同意任何前缀），并且只在跟随者肯定地确认AppendEntries RPC时才更新。  

### term混乱(term不稳定)

因为网络导致的RPC过期问题，term混淆是指服务器被来自旧term的RPC所迷惑。一般来说，在收到RPC时，因为图2中的规则确切地说明了当你看到一个旧term时你应该做什么。然而，图2一般没有讨论当你收到旧的RPC回复时你应该做什么。根据经验，我们发现，到目前为止，最简单的做法是首先记录回复中的term(它可能比你当前的term高),然后将当前term与你在原始RPC中发送的term进行比较。如果两者不同，就放弃回复并返回。只有当这两个术term相同时，你才应该继续处理回复。也许你可以通过一些巧妙的协议推理在这里做进一步的优化，但这种方法似乎很有效。而不这样做会导致一条漫长而曲折的血汗、泪水和绝望的道路。  
NOTE:A节点无论是RV还是AE的RPC,在回复中都要进行与A节点的currenTerm进行比对,如果发现不对，就立即放弃回复并返回。

### 优化

Raft论文包括几个感兴趣的可选功能。在6.824中，我们要求学生实现其中的两个：日志压缩（第7节）和加速日志回溯（第8页的左上方）。前者对于避免日志无限制地增长是必要的，而后者对于使落后的追随者快速更新是有用的。  

这些功能不是 "核心Raft "的一部分，因此在论文中没有得到像主要共识协议那样的关注。日志压缩的内容相当全面（在图13中），但遗漏了一些设计细节，如果你太随意地阅读，可能会错过。  

- 当快照应用程序状态时，你需要确保应用程序状态与Raft日志中某个已知索引之后的状态相对应，这意味着应用程序要么需要向Raft传达快照所对应的索引，要么Raft需要推迟应用额外的日志条目，直到快照完成。
- 该文本没有讨论当服务器崩溃并重新启动时的恢复协议，因为现在涉及到快照。特别是，如果Raft状态和快照是分开提交的，服务器可能会在坚持快照和坚持更新的Raft状态之间崩溃。这是一个问题，因为图13中的第7步决定了快照所覆盖的Raft日志必须被丢弃。如果当服务器重新启动时，它读取的是更新的快照，而不是过时的日志，那么它最终可能会应用一些已经包含在快照中的日志条目。这种情况会发生，因为commitIndex和lastApplied没有被持久化，所以Raft不知道这些日志条目已经被应用。解决这个问题的方法是在Raft中引入一个持久化状态，记录Raft持久化日志中的第一个条目所对应的 "真实 "索引。然后，这可以与加载的快照的lastIncludedIndex进行比较，以确定在日志的头部有哪些元素需要丢弃.  
-  如果当服务器重新启动时，它读取的是更新的快照，而不是过时的日志，那么它最终可能会应用一些已经包含在快照中的日志条目。这种情况会发生，因为commitIndex和lastApplied没有被持久化，所以Raft不知道这些日志条目已经被应用。解决这个问题的方法是给Raft引入一个持久化状态，记录Raft持久化日志中的第一个条目对应的 "真实 "索引。然后，这可以与加载的快照的lastIncludedIndex进行比较，以确定要丢弃日志头部的哪些元素。  

加速日志回溯的优化是非常不明确的，可能是因为作者认为这对大多数部署来说是不必要的。文本中并没有明确说明领导者应该如何使用从客户端发回的冲突索引和术语来决定使用哪一个NextIndex。我们认为作者可能希望你遵循的协议是。

- 如果一个跟随者的日志中没有prevLogIndex，它应该以conflictIndex = len(log)和conflictTerm = None返回。
- 如果一个跟随者在其日志中确实有prevLogIndex，但是术语不匹配，它应该返回conflictTerm = log[prevLogIndex].Term，然后在其日志中搜索其条目中术语等于conflictTerm的第一个索引。
- 在收到冲突响应时，领导者应该首先搜索其日志中的conflictTerm。如果它在日志中找到一个具有该term的条目，它应该将nextIndex设置为其日志中该term的最后一个条目的索引之外的那个索引。
- 如果它没有找到该术语的条目，它应该设置 nextIndex = conflictIndex。一个半途而废的解决方案是只使用conflictIndex（而忽略conflictTerm），这简化了实现，但这样一来，领导者有时会向跟随者发送更多的日志条目，而不是严格意义上所需要的，以使他们达到最新状态。

## Applications on top of Raft

## AppendIndex

- 2022/08/21
    etcd的raft模块难得不只是一点点，看来不能直接看它和etcd的整合，尝试着理解那个example开始。一个玩具的raftkv.
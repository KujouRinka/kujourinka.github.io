---
title: 6.824 Lab2A Leader Election 可行方案
date: 2023-02-08 20:00
---

#### 一些碎碎念

书接上回，这是 6.824 的 [lab2A](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html)，实现 raft 协议中的 leader election。关于 raft 的更多详细内容，[raft paper](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf) 和网络上大多数文章一定会比我在这里介绍得详细，这里只给出一个链接，以动图的方式助于理解 raft 的工作原理：[Raft](http://thesecretlivesofdata.com/raft/)，https://raft.github.io/ 也提供了一个可交互的动画，大家可以去玩一玩。

个人的这个实现相较于网上的各种版本，代码量较大，但个人感觉逻辑更加清晰。

不保证 bug free，但使用课程中的 test 测试了近 1000 轮无一失败。

#### 思路

实现 raft 协议中的 leader election。由 paper 中可知集群中的所有节点会选出一个 leader 节点，选出 leader 后其余节点均为 follower。对集群的各种操作都需要经过 leader 之手，具体表现为 client 直接向 leader 进行请求，或向 follower 请求，随后该 follower 将请求重定向到 leader。

对于单个节点，有三种可能的状态：

- follower
- candidate
- leader

对于每个节点，有以下几种事件会导致状态间的转移：

- 超时事件
- **接收**到来自 RequestVote RPC 的请求
- **接收**到来自 AppendEntries RPC 的请求

同时由于处于 candidate 和 leader 状态下的节点分别会**发出** RequestVote 和 AppendEntries 请求，还应该在上面三个事件中加入：

- 来自 RequestVote RPC 请求的回应
- 来自 AppendEntries RPC 请求的回应

于是，整个 leader election 变成了一个填表游戏：

| 事件\行为\状态 | follower | candidate |leader|
|:-----:|:-----:|:-----:|:---:|
| timeout | a | d |h|
| RequestVote recv | b | e |i|
| AppendEntries recv | c | f |j|
| RequestVote callback | / | g |/|
| AppendEntries callback | / | / |k|

当表填完，整个流程就基本完成了。

#### 实现

##### 杂项

##### rpc.go

依照 paper 对 `rpc.go` 进行了一些修改：

``` go
// ---------- for RequestVote ----------
type RequestVoteArgs struct {
	// Your data here (2A, 2B).
	Term        int // candidate's curTerm
	CandidateId int // candidate requesting vote
}

type RequestVoteReply struct {
	// Your data here (2A).
	Term        int  // currentTerm, for candidate to update itself
	VoteGranted bool // true means candidate received vote
}

type voteParam struct {
	args   *RequestVoteArgs
	reply  *RequestVoteReply
	notify chan struct{}
}

func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	notify := make(chan struct{})
	rf.voteChan <- voteParam{args, reply, notify}
	<-notify
}

func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply) bool {
	ok := rf.peers[server].Call("Raft.RequestVote", args, reply)
	return ok
}

// ---------- for AppendEntries ----------
type AppendEntriesArgs struct {
	Term     int // leader's curTerm
	LeaderId int // so follower can redirect clients
}

type AppendEntriesReply struct {
	Term    int  // currentTerm, for leader to update itself
	Success bool // true if follower contained entry matching prevLogIndex and prevLogTerm
}

type appendEntryParam struct {
	args   *AppendEntriesArgs
	reply  *AppendEntriesReply
	notify chan struct{}
}

func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	notify := make(chan struct{})
	rf.entryChan <- appendEntryParam{args, reply, notify}
	<-notify
}

func (rf *Raft) sendAppendEntries(server int, args *AppendEntriesArgs, reply *AppendEntriesReply) bool {
	ok := rf.peers[server].Call("Raft.AppendEntries", args, reply)
	return ok
}
```

我并没有将处理的逻辑直接写到 RPC 处理函数中，而是将其封装到一个结构中，发送到一个专门用于接受这个参数的 channel 中，并传入一个空 channel 作同步作用。

##### raft.go

在 `type Raft struct` 中增加了一些 paper 中提到的本实现需要用到的字段，包括上面提到的接受 RPC 参数的 channel：

``` go

	// Your data here (2A, 2B, 2C).
	// Look at the paper's Figure 2 for a description of what
	// state a Raft server must maintain.
	curTerm  int    // current curTerm
	state    RState // current state
	votedFor int    // candidate id that received vote in current curTerm

	voteChan  chan voteParam        // channel for vote request
	entryChan chan appendEntryParam // channel for entry request
```

有关 `RState interface` 和具体的实现，定义如下：

``` go
type RState interface {
	Run(tf *Raft)
}

type Follower struct{}
type Candidate struct{}
type Leader struct{}
```

这里的设计参考了[设计模式：可复用面向对象软件的基础](https://book.douban.com/subject/34262305/)一书中的 State 模式，于是，便有了 `GetState` 函数的如下写法：

``` go
func (rf *Raft) GetState() (int, bool) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	return rf.curTerm, reflect.TypeOf(rf.state) == reflect.TypeOf(&Leader{})
}
```

`ticker` 函数也变得格外简单：

``` go
func (rf *Raft) ticker() {
	for !rf.killed() {
		rf.state.Run(rf)
	}
}
```

`Make` 函数只需要初始化我们添加的几个字段即可：

``` go
func Make(peers []*labrpc.ClientEnd, me int,
	persister *Persister, applyCh chan ApplyMsg) *Raft {
	rf := &Raft{}
	rf.peers = peers
	rf.persister = persister
	rf.me = me

	rf.curTerm = 0
	rf.state = &Follower{}
	rf.votedFor = -1

	rf.voteChan = make(chan voteParam)
	rf.entryChan = make(chan appendEntryParam)
	rf.readPersist(persister.ReadRaftState())

	// start ticker goroutine to start elections
	go rf.ticker()

	return rf
}
```

##### common.go

定义一些常用函数：

``` go
const (
	ReElectLower = 150
	ReElectUpper = 300

	HeartBeatTimeout = 100
)

func randBetween(lower, upper int) int {
	return rand.Intn(upper-lower) + lower
}

func electionTimeout() time.Duration {
	return time.Duration(randBetween(ReElectLower, ReElectUpper)) * time.Millisecond
}

func heartbeatTimeout() time.Duration {
	return time.Duration(HeartBeatTimeout) * time.Millisecond
}

func resetTimer(timer *time.Timer, timeout time.Duration) {
	if !timer.Stop() {
		<-timer.C
	}
	timer.Reset(timeout)
}
```

#### 重头戏

下面正片开始，我将不同状态的逻辑分开写，虽然代码看上去挺长的，但是我感觉逻辑挺清晰的

##### follower.go

follower 是最简单的一个实现对照我上面画的一个表，只需要处理三个事件，大致框架如下：

``` go
func (f *Follower) Run(rf *Raft) {
	timer := time.NewTimer(electionTimeout())
	for {
		select {
		case <-timer.C:
		// a 处理超时
		case vote := <-rf.voteChan:
		// b 处理 RequestVote 请求
		case entry := <-rf.entryChan:
		// c 处理 AppendEntries 请求
	}
}
```

当 follower 超时，自动变为 candidate，并为自己投票，因此，a 处填写：

``` go
rf.mu.Lock()
rf.state = &Candidate{}
rf.votedFor = rf.me
rf.mu.Unlock()
return
```

当 candidate 收到 RequestVote RPC 的请求，首先检查 `Term`，若小于 `curTerm` 则拒绝为其投票；若满足 `Term > curTerm` 或者 `Term == curTerm && vote.args.CandidateId`，同意为其投票，并重置超时计时器。b 处填写：

``` go
vote.reply.Term = vote.args.Term
vote.reply.VoteGranted = false

vote.args.CandidateId)
rf.mu.Lock()
if vote.args.Term < rf.curTerm {
	rf.mu.Unlock()
	vote.notify <- struct{}{}
	continue
}

// grant vote, update curTerm
if vote.args.Term > rf.curTerm ||
	(vote.args.Term == rf.curTerm && rf.votedFor == vote.args.CandidateId) {
	rf.curTerm = vote.args.Term
	rf.votedFor = vote.args.CandidateId
	vote.reply.Term = rf.curTerm
	vote.reply.VoteGranted = true
	// reset timer
	resetTimer(timer, electionTimeout())
}
rf.mu.Unlock()
vote.notify <- struct{}{}
```

这里提一嘴，`time.Timer` 和 `time.Ticker` 真的很容易用错，这里建议参考 golang 关于这两者的[文档](https://pkg.go.dev/time)。

当收到 AppendEntries RPC，同样依照 paper 中所说的检查任期等等逻辑如法炮制，c 处填写：

```go
entry.args.LeaderId)
rf.mu.Lock()
if entry.args.Term < rf.curTerm {
	// stale curTerm, do nothing
	entry.reply.Term = rf.curTerm
	entry.reply.Success = false
} else if entry.args.Term > rf.curTerm {
	// larger curTerm, may this server is out of date
	rf.curTerm = entry.args.Term
	rf.votedFor = entry.args.LeaderId
	entry.reply.Term = rf.curTerm
	entry.reply.Success = true
	resetTimer(timer, electionTimeout())
} else {
	// same curTerm, reset timer
	entry.reply.Term = rf.curTerm
	entry.reply.Success = true
	resetTimer(timer, electionTimeout())
}
rf.mu.Unlock()
entry.notify <- struct{}{}
```

 ##### candidate.go

先给出框架：

``` go
func (c *Candidate) Run(rf *Raft) {
	rf.mu.Lock()
	rf.curTerm++

    // TODO: send RequestVote RPC
	rf.mu.Unlock()

	timer := time.NewTimer(electionTimeout())
	for {
		select {
		case <-timer.C:
		// d 超时
		case reply := <-replyChan:
		// g RequestVote 回答
		case vote := <-rf.voteChan:
		// e RequestVote 请求
		case entry := <-rf.entryChan:
		// f AppendEntries 请求
		}
	}
}
```

首先自增 curTerm，发送 RequestVote 请求其他节点的选票，随后开启循环等待事件驱动，其中 d，e，f 遵随 paper 所描述即可，只需要注意加锁释放锁的时机，这里简单贴一下代码：

d：

``` go
return
```

e：

``` go
rf.mu.Lock()
if vote.args.Term > rf.curTerm {
	rf.curTerm = vote.args.Term
	rf.votedFor = vote.args.CandidateId
	rf.state = &Follower{}
	rf.mu.Unlock()
	vote.reply.VoteGranted = true
	vote.reply.Term = vote.args.Term
	vote.notify <- struct{}{}
	return
} else if vote.args.Term < rf.curTerm {
	vote.reply.VoteGranted = false
	vote.reply.Term = rf.curTerm
} else {
	vote.reply.VoteGranted = false
	vote.reply.Term = rf.curTerm
}
rf.mu.Unlock()
vote.notify <- struct{}{}
```

f：

``` go
rf.mu.Lock()
// a leader send AppendEntries, if args.term >= curTerm
// acknowledge the leader and become follower
// or reject the request then stay as candidate
if entry.args.Term >= rf.curTerm {
	rf.curTerm = entry.args.Term
	rf.votedFor = entry.args.LeaderId
	rf.state = &Follower{}
	rf.mu.Unlock()
	entry.reply.Term = entry.args.Term
	entry.reply.Success = true
	entry.notify <- struct{}{}
	return
} else {
	entry.reply.Term = rf.curTerm
	entry.reply.Success = false
}
rf.mu.Unlock()
entry.notify <- struct{}{}
```

而 candidate 相较于 follower 多出来的逻辑部分就全部在下面了，我们需要在进入循环前以类似异步的方式发送 RequestVote 请求，并在循环中通过添加了一个 replyChan 进行处理 RequestVote 请求的回答。这样将请求和事件分开，逻辑没有那么混乱。于是，发送请求的代码为：

``` go
// send RequestVote RPC
replyChan := make(chan RequestVoteReply, len(rf.peers)-1)
args := RequestVoteArgs{
	Term:        rf.curTerm,
	CandidateId: rf.me,
}
for i := range rf.peers {
	if i == rf.me {
		continue
	}
	go func(pees *labrpc.ClientEnd) {
		var reply RequestVoteReply
		if pees.Call("Raft.RequestVote", &args, &reply) {
			replyChan <- reply
		}
	}(rf.peers[i])
}

grantedCnt := 1
minVote := len(rf.peers)/2 + 1
```

而对应的事件处理，当收到的选票大于集群数量的一半时，转换为 leader，但同时仍要处理任期 `Term`，g：

``` go
rf.mu.Lock()
if reply.Term > rf.curTerm {
	// received larger term, become follower
	rf.curTerm = reply.Term
	rf.votedFor = -1
	rf.state = &Follower{}
	rf.mu.Unlock()
	return
} else if reply.Term == rf.curTerm && reply.VoteGranted {
	// received a grantVote
	grantedCnt++
	if grantedCnt >= minVote {
		rf.state = &Leader{}
		rf.mu.Unlock()
		return
	}
}
rf.mu.Unlock()
```

##### leader.go

框架：

``` go
func (l *Leader) Run(rf *Raft) {
    // TODO: send heartbeat
    // TODO: send AppendEntries RPCs to all other servers

	// make heartbeat timer
	timer := time.NewTimer(heartbeatTimeout())
	for {
		select {
		case <-timer.C:
		// h 超时
		case reply := <-replyChan:
		// k AppendEntries 回答
		case vote := <-rf.voteChan:
		// i RequestVote 请求
		case entry := <-rf.entryChan:
		// j AppendEntries 请求
		}
	}
}
```

h: 超时：

``` go
return
```

i：

``` go
rf.mu.Lock()
if vote.args.Term > rf.curTerm {
	rf.curTerm = vote.args.Term
	rf.votedFor = vote.args.CandidateId
	rf.state = &Follower{}
	vote.reply.VoteGranted = true
	vote.reply.Term = rf.curTerm
	rf.mu.Unlock()
	vote.notify <- struct{}{}
	return
} else if vote.args.Term <= rf.curTerm {
	vote.reply.VoteGranted = false
	vote.reply.Term = rf.curTerm
}
rf.mu.Unlock()
vote.notify <- struct{}{}
```

j：

``` go
rf.mu.Lock()
if entry.args.Term > rf.curTerm {
	rf.curTerm = entry.args.Term
	rf.votedFor = entry.args.LeaderId
	rf.state = &Follower{}
	entry.reply.Success = true
	entry.reply.Term = rf.curTerm
	rf.mu.Unlock()
	entry.notify <- struct{}{}
	return
} else if entry.args.Term <= rf.curTerm {
	entry.reply.Success = false
	entry.reply.Term = rf.curTerm
}
rf.mu.Unlock()
entry.notify <- struct{}{}
```

模仿 RequestVote 的发送方式，发送 AppendEntries 请求，这个实验中只需要发送 heartbeat，实现起来并不困难：

``` go
// send heartbeat
rf.mu.Lock()
replyChan := make(chan AppendEntriesReply, len(rf.peers))
args := AppendEntriesArgs{
	Term:     rf.curTerm,
	LeaderId: rf.me,
}
// send AppendEntries RPCs to all other servers
for i := range rf.peers {
	if i == rf.me {
		continue
	}
	go func(peer *labrpc.ClientEnd) {
		var reply AppendEntriesReply
		if peer.Call("Raft.AppendEntries", &args, &reply) {
			replyChan <- reply
		}
	}(rf.peers[i])
}
rf.mu.Unlock()
```

k：

``` go
rf.mu.Lock()
if vote.args.Term > rf.curTerm {
	rf.curTerm = vote.args.Term
	rf.votedFor = vote.args.CandidateId
	rf.state = &Follower{}
	vote.reply.VoteGranted = true
	vote.reply.Term = rf.curTerm
	rf.mu.Unlock()
	vote.notify <- struct{}{}
	return
} else if vote.args.Term <= rf.curTerm {
	vote.reply.VoteGranted = false
	vote.reply.Term = rf.curTerm
}
rf.mu.Unlock()
vote.notify <- struct{}{}
```

#### 最后

整个实验做下来的感觉：难，但是豁然开朗。明白了很多东西，很多细节需要处理，以及如何把握整个框架。写这么复杂的状态框架的原因是为了后续实验的更好拓展。

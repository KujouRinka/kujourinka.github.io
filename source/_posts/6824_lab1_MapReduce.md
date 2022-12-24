---
title: 6.824 Lab1 MapReduce 快速实现
date: 2022-12-23 22:00
---

这个是 MIT 6.824 分布式系统（Distributed Systems）2022 年的 lab1，[lab 链接](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)。字有点多，需要花些时间耐心读一下。

个人认为这个 Lab 在对 Golang 有一定了解的情况下不是很难实现，一定要把题意理解清楚，画一个大致框架再动手。我用了两天时间，第一天主要是读题（再次吐槽一下字有点多）和画大致框架，第二天花了一下午几个小时写了代码。Debug 没有难度，可能是我运气比较好，一把过2333

下面开始讲个人的实现，代码量不大，算上注释大致 200 行。

[rpc.go 的修改](https://github.com/KujouRinka/6.824-2022/commit/2f3c87aff43fc548f341853307b424d8ab37438e)

[coordinator.go 的修改](https://github.com/KujouRinka/6.824-2022/commit/9b3fc49785e57b2846145c9082a162769456a2c7)

[worker.go 的修改](https://github.com/KujouRinka/6.824-2022/commit/e6833214404b7fb567d85ba9edc62444781ad9e7)

#### 复述

实现 Google 曾经使用的 [MapReduce](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf) 的一个简单版本。具体的，修改`mr` 目录下的 `coordinator.go`，`rpc.go`，`worker.go` 三个文件实现 MapReduce

#### 思路

主要是修改 `coordinator.go` 和 `worker.go`，分别对于 pdf 文档中的 Master 和 Worker。前者负责调度任务，比如如何分配文件，如何规定超时任务的重新安排等等；后者负责实际执行 `map` 和 `reduce` 函数。

由于实际编写的是 coordinator 和 woker 的分布式版本，我们需要模拟这两个函数实际是在不同的机器上运行的，所以 coordinator 和 worker 两者的通信方式需要使用 RPC。于是，我们需要简单定义一下两者的通信方式：

- 我们需要知道，需要先启动单个 coordinator，随后再启动多个 worker，这个测试脚本为我们做了，我们只需要了解。
- worker 通过 **RPC** 向 coordinator 发送请求，表示“我现在空闲，请给我一个任务”。
- 由于 coordinator 已经注册了 RPC 服务（我们无需关注），其可以收到来自 worker 的 RPC 请求，我们从 coordinator 的任务队列中取出一个任务发送给 worker，并同时启动一个用于接收 worker 完成任务的方法和一个超时计时器。前者用于当 worker 完成任务时通知 coordinator；后者用于当 worker 没有按时完成任务时（paper 中解释的，可能 worker 崩溃了，或者网络阻塞等等），用于将任务重新安排给一个 worker。
- 当 worker 通过 RPC 获取到任务时，进行实际的工作。如果任务出错，直接终止等待下个任务即可，因为 coordinator 的计时器会帮我们重新安排这个任务；如果任务成功，再通过 RPC 告知 coordinator 任务已完成。若对 coordinator 的 RPC 调用失败，我们认为 coordinator 已经传输了所有的任务已退出，worker 可以终止。

简单的流程图如下：

![cw_rpc](https://raw.githubusercontent.com/KujouRinka/flowchart/master/cw_rpc.svg)

程序的流程是：

- 我们通过对每个原始文件的内容作为 map 函数的参数输入，得到一组 `KeyValue`，对于不同的 `Key` 通过 `ihash(key) % nReduce` 计算出不同的 `key` 的 `reduceId` 将这个结果保存在 `mr-{taskId}-{reduceId}` 这个中间文件（intermediate）中。
- 当所有原始文件全部生成完对应的 `mr-{taskId}-{reduceId}` 文件后，coordinator 开始分配 reduce 函数的任务，于是每个 worker 只需要查找全部对应的 `reduceId` 的文件（具体的：`mr-*-{reduceId}`）进行合并处理和  `reduce` 调用即可。对于每个 `reduceId` ，生成对应的 `mr-out-{reduceId}` 文件。

简单来说，就是：`pg*.txt` --> `mr-1-0`, `mr-1-1`, ..., `mr-2-0`, `mr-2-1`, ... --> `mr-out-0`, `mr-out-1`,... 这个流程。

#### 代码修改

由于没有删除的代码，下面只列出添加的代码：

##### rpc.go

首先添加一些 rpc 的定义：

``` go
type Task struct {
	Type     int		// 任务的类型，包括 Map 和 Reduce
	Filename string		// 当 Type 为 Map 时字段有效，表示需要作用 Map 函数的文件的文件名
	TaskId   int		// 用于标记这个 Task 的 Id，在对 Coordinator 回馈结果的时候作为参数返回
	ReduceId int		// Coordinator 要求这个 Worker 统计的 Reduce 的 Id
}

const (
	Map = iota
	Reduce
)

// 创建一个对应 Map 的 Task 结构
func NewMapTask(filename string, taskId int) Task {
	return Task{
		Type:     Map,
		Filename: filename,
		TaskId:   taskId,
	}
}

// 创建一个对应 Reduce 的 Task 的结构
func NewReduceTask(reduceId, taskId int) Task {
	return Task{
		Type:     Reduce,
		TaskId:   taskId,
		ReduceId: reduceId,
	}
}

// NeedWork RPC: 用于请求 Coordinator 分配新的任务
// NeedWork 的参数
type NeedWorkArgs struct {
}

// NeedWork 的返回值
type NeedWorkReply struct {
	T         Task
	ReduceCnt int
}

// FinishWork RPC: 用于告知 Coordinator 任务已完成
// FinishWork 的参数
type FinishWorkArgs struct {
	TaskId int
}

// FinishWork 的返回值
type FinishWorkReply struct {
}
```

##### worker.go

``` go
// 添加的 import
import (
	"encoding/json"
	"fmt"
	"io"
	"os"
	"sort"
	"strings"
)

// 引入的排序 interface，从 mrsequential.go copy 来
// for sorting by key.
type ByKey []KeyValue

// for sorting by key.
func (a ByKey) Len() int           { return len(a) }
func (a ByKey) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByKey) Less(i, j int) bool { return a[i].Key < a[j].Key }
```

下面是 `Worker` 函数的定义，全是感情，没有任何 Go 的技巧 :)

``` go
func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string) {
	for {
		needWordReply := NeedWorkReply{}
		ok := call("Coordinator.NeedWork", &NeedWorkArgs{}, &needWordReply)
		if !ok {
			// 当 RPC 调用失败，我们认为 Coordinator 任务完成退出，
			// 所以 Worker 理应退出终止
			// Coordinator finish its work
			break
		}
		if needWordReply.T.Type == Map {
			// 处理 Map 任务的逻辑
			filename := needWordReply.T.Filename
			file, err := os.Open(filename)
			if err != nil {
				log.Fatalf("cannot open %v", filename)
			}
			content, err := io.ReadAll(file)
			if err != nil {
				log.Fatalf("cannot read %v", filename)
			}
			file.Close()
			// 得到单个文件的 Key-Value
			kva := mapf(filename, string(content))

			// 生成 intermediate 文件的逻辑
			// 首先计算每个 Key 的 reduceId，保存在 intermediate 这个二维切片中
			intermediate := make([][]KeyValue, needWordReply.ReduceCnt)
			for _, kv := range kva {
				reduceTask := ihash(kv.Key) % needWordReply.ReduceCnt
				intermediate[reduceTask] = append(intermediate[reduceTask], kv)
			}
			// intermediate[i] 对应 mr-{taskId}-{i} 这个中间文件
			// 以 paper 中提示的使用 json 的方式写入
			for i := 0; i < needWordReply.ReduceCnt; i++ {
				ofilename := fmt.Sprintf("mr-%d-%d", needWordReply.T.TaskId, i)
				// ofile, _ := os.Create(ofilename)
				tf, _ := os.CreateTemp("./", ofilename)
				enc := json.NewEncoder(tf)
				for _, kv := range intermediate[i] {
					enc.Encode(&kv)
				}
				tf.Close()
				os.Rename(tf.Name(), ofilename)
			}
		} else if needWordReply.T.Type == Reduce {
			// 处理 Reduce 任务的逻辑
			// 找出目录下所有对应该 Reduce 的文件名
			// find all files corresponding to this reduce task
			var filenames []string
			files, err := os.ReadDir(".")
			if err != nil {
				log.Fatalf("cannot read current directory")
			}
			for _, file := range files {
				if file.IsDir() {
					continue
				}
				filename := file.Name()
				prefix := "mr-"
				suffix := fmt.Sprintf("-%d", needWordReply.T.ReduceId)
				if strings.HasPrefix(filename, prefix) && strings.HasSuffix(filename, suffix) {
					filenames = append(filenames, filename)
				}
			}

			// 对所有已找到的文件进行读取
			// do reduce job
			var kva []KeyValue
			for _, filename := range filenames {
				file, err := os.Open(filename)
				if err != nil {
					log.Fatalf("cannot open %v", filename)
				}
				dec := json.NewDecoder(file)
				for {
					var kv KeyValue
					if err := dec.Decode(&kv); err != nil {
						break
					}
					kva = append(kva, kv)
				}
			}

			// 对该任务进行 Reduce 调用，逻辑和 mrsequential.go 完全一致，代码直接 copy
			// copy from mrsequential.go
			sort.Sort(ByKey(kva))
			oname := fmt.Sprintf("mr-out-%d", needWordReply.T.ReduceId)
			ofile, _ := os.Create(oname)
			i := 0
			for i < len(kva) {
				j := i + 1
				for j < len(kva) && kva[j].Key == kva[i].Key {
					j++
				}
				values := []string{}
				for k := i; k < j; k++ {
					values = append(values, kva[k].Value)
				}
				output := reducef(kva[i].Key, values)
				fmt.Fprintf(ofile, "%v %v\n", kva[i].Key, output)
				i = j
			}
			ofile.Close()
		} else {
			// unknown task type
			log.Fatalf("unknown task type: %v", needWordReply.T.Type)
		}

		// 运行到此代表该任务成功完成，通过 RPC 告知 Coordinator 任务完成
		// make FinishWork RPC call
		call("Coordinator.FinishWork", &FinishWorkArgs{TaskId: needWordReply.T.TaskId}, &FinishWorkReply{})
	}
	// log.Printf("Worker: %v exit", os.Getpid())
}
```

##### coordinator.go

个人认为 `coordinator.go` 的实现颇具技巧，网络上很多实现有大量评论代码“很不 Go”，没有 Go 的风格。

我虽说不是 Go 的高手，但也有一段时间具体学习了一下 Go 这门语言，参加了一些开源项目，对自己的代码风格还有有些信心，在这里自卖自夸一下2333，话不多说，先是 import：

``` go
import (
	"fmt"
	"log"
	"sync"
	"time"
)
```

对 `Coordinator` 结构体的修改，添加了一些字段用于操作：

``` go
type Coordinator struct {
	// Your definitions here.
	// 保存一下 nReduce
	nReduce int

	// 定义任务队列和任务的计数器，
	// 用于任务的发送，判断 Map 和 Reduce 任务的交界以及何时关闭 Coordinator
	// task sending definition
	taskQueue chan Task
	taskWg    sync.WaitGroup

	// 任务 Id 计数器，自增
	// task id counter
	taskIdCounter int

	// 保存用于通知任务完成的 chan 的字段，
	// 若 Worker 完成并调用 RPC，处理函数会向对应的 channel 发送一个信号
	// task notification record
	taskNotifyMap  map[int]chan struct{}
	taskNotifyLock sync.Mutex

	// 用于标记所有任务是否完成
	// done flag
	done     bool
	doneLock sync.Mutex
}

// 获得一个新的 taskId
func (c *Coordinator) NewTaskId() int {
	c.taskIdCounter++
	return c.taskIdCounter
}

// Done 函数，返回所有任务是否已经完成
func (c *Coordinator) Done() bool {
	c.doneLock.Lock()
	ret := c.done
	c.doneLock.Unlock()

	// Your code here.
	return ret
}
```

`MakeCoordinator` 函数的设计：

``` go
func MakeCoordinator(files []string, nReduce int) *Coordinator {
	// 初始化一个 Coordinator
	c := Coordinator{
		nReduce:       nReduce,
		taskQueue:     make(chan Task, 100),
		taskWg:        sync.WaitGroup{},
		taskIdCounter: 0,
		taskNotifyMap: make(map[int]chan struct{}),
		done:          false,
	}
	// log.Printf("%d file found\n", len(files))

	// 由于 Reduce 任务必须在所有 Map 任务完成后才能开始
	// 添加 Map 任务计数到 WaitGroup，用于标识所有的 Map 任务是否完成
	c.taskWg.Add(len(files))

	// Your code here.

	// 开启一个新的 goroutine 向 c.taskQueue channel 中发送任务
	// make send task goroutine
	go func(files []string) {
		// 发送 Map 任务
		// send map task
		for _, filename := range files {
			c.taskQueue <- NewMapTask(filename, c.NewTaskId())
		}
		// log.Println("all task sent, waiting for next reduce task")
		// 等待所有 Map 任务完成，这时才能发送 Reduce 任务
		// wait for all map task done
		c.taskWg.Wait()
		// log.Println("start sending reduce task")

		// 添加 Reduce 任务的计数
		// send reduce task
		c.taskWg.Add(nReduce)
        
		// 下面这个 goroutine 的作用是
		// 等待 WaitGroup 值为 0 时，此时表示所有的 Reduce 任务也完成了
		// 所以可以将 c.done 置为 true，表示 Coordinator 可以结束了
		// make Done() check goroutine
		// log.Println("waiting exit goroutine created")
		go func() {
			c.taskWg.Wait()
			c.doneLock.Lock()
			c.done = true
			c.doneLock.Unlock()
		}()
		// log.Println("sending reduce task")

		// 发送 Reduce 任务
		for i := 0; i < nReduce; i++ {
			c.taskQueue <- NewReduceTask(i, c.NewTaskId())
		}
	}(files)

	c.server()
	return &c
}
```

`NeedWork` RPC 函数：

``` go
func (c *Coordinator) NeedWork(args *NeedWorkArgs, reply *NeedWorkReply) error {
	// 从任务队列 c.taskQueue 中取出一个任务作为 RPC 返回值
	task, ok := <-c.taskQueue
	if !ok {
		return fmt.Errorf("cannot get task from task queue")
	}
	reply.T = task
	reply.ReduceCnt = c.nReduce

	// 使用 taskId 作为唯一任务标识，创建一个 struct{} channel
	// 用于 FinishWork RPC 函数通知任务已被 Worker 完成
	// 注意锁的临界区，否则会导致 condition race
	// make task notification channel
	c.taskNotifyLock.Lock()
	c.taskNotifyMap[task.TaskId] = make(chan struct{})
	// 启动 goroutine 定时
	// set timer for task
	go func(taskChan chan struct{}) {
		select {
		case <-taskChan:
			// 任务完成
			// task done
			return
		case <-time.After(10 * time.Second):
			// 任务超时，重新将任务放回任务队列中
			c.taskQueue <- task
		}
	}(c.taskNotifyMap[task.TaskId])
	c.taskNotifyLock.Unlock()

	return nil
}
```

`FinishWork` RPC 函数：

``` go
func (c *Coordinator) FinishWork(args *FinishWorkArgs, reply *FinishWorkReply) error {
	taskId := args.TaskId
	c.taskNotifyLock.Lock()
	// 思考这里为什么要检查 taskId 所对应的值是否有效
	// 若考虑网络延迟之类的不确定因素，可能导致同一个 taskId 调用两次 FinishWork 函数
	notifyChan, ok := c.taskNotifyMap[taskId]
	if !ok {
		c.taskNotifyLock.Unlock()
		return nil
	}
	// 向对应 channel 发送信号表示任务完成，同时任务计数 -1
	// notify task done
	notifyChan <- struct{}{}
	c.taskWg.Done()
	// log.Printf("task %d done\n", taskId)

	// 删除对应的 channel
	// delete task notification channel
	delete(c.taskNotifyMap, taskId)
	c.taskNotifyLock.Unlock()
	return nil
}
```

#### 总结

通过这个实验我大致搞明白了一个简单的分布式系统由哪些部分组成。至此，分布式系统的 Lab1 完成，我认为难点不在代码的实现，而是题目的理解。更干净的代码实现见本人的 [github commit history](https://github.com/KujouRinka/6.824-2022/commits/master)

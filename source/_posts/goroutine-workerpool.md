---
title: '在 Go 语言中如何实现协程池'
date: 2025-05-09 23:00:42
tags:
- Go
categories:
- Go
---

如果你熟悉 Java、Python 等编程语言，那么你一定听说或者使用过进程池或线程池。因为进程和线程不是越多越好，过多的进程或线程可能造成资源浪费和性能下降。所以池化技术在这些主流编程语言中非常流行，可以有效控制并发场景下资源使用量。

而 Go 语言则没有提供多进程和多线程的支持，仅提供了协程（goroutine）的概念。在 Go 中开启 goroutine 的成本非常低，以至于我们在绝大多数情况下开启 goroutine 时根本无需考虑协程数量，所以也就很少有人提及 Go 的协程池化技术。不过使用场景少，不代表完全没用。通过协程池我们可以来掌控资源使用量，降低协程泄漏风险。

<!-- more -->

[gammazero/workerpool](https://github.com/gammazero/workerpool) 就是用来实现协程池的 Go 包，本文我们一起来学习一下其使用方法，并深入其源码来探究下如何实现一个 Go 协程池。

### 使用示例

workerpool 直译过来是工作池，在 Go 中就是指代协程池。workerpool 的用法非常简单，示例代码如下：

```go
package main

import (
	"fmt"
	"time"

	"github.com/gammazero/workerpool"
)

func main() {
	wp := workerpool.New(2)
	requests := []string{"alpha", "beta", "gamma", "delta", "epsilon"}

	for _, r := range requests {
		wp.Submit(func() {
			fmt.Printf("%s: Handling request: %s\n", time.Now().Format(time.RFC3339), r)
			time.Sleep(1 * time.Second)
		})
	}

	wp.StopWait()
}
```

`workerpool.New(2)` 表示我们创建了一个容量为 2 的协程池，即同一时刻最多只会有 2 个 goroutine 正在执行。`wp.Submit()` 用来提交一个任务，任务类型为无参数和返回值的函数 `func()`，这里我们在 `for` 循环中提交了 5 个任务。调用 `wp.StopWait()` 可以等待所有已提交的任务执行完成。

执行示例代码，得到输出如下：

```bash
$ go run main.go
2025-05-08T23:40:16+08:00: Handling request: alpha
2025-05-08T23:40:16+08:00: Handling request: beta
2025-05-08T23:40:17+08:00: Handling request: gamma
2025-05-08T23:40:17+08:00: Handling request: delta
2025-05-08T23:40:18+08:00: Handling request: epsilon
```

不过这里的输出内容并不是一下子全部输出完成的，而是两行两行的输出。

根据打印的时间可以发现，是先输出：

```bash
2025-05-08T23:40:16+08:00: Handling request: alpha
2025-05-08T23:40:16+08:00: Handling request: beta
```

接着等待 1s 再输出：

```bash
2025-05-08T23:40:17+08:00: Handling request: gamma
2025-05-08T23:40:17+08:00: Handling request: delta
```

再次等待 1s 最后输出：

```bash
2025-05-08T23:40:18+08:00: Handling request: epsilon
```

这个输出结果符合预期，也就是说同一时刻最多只会有 2 个 goroutine 正在执行。

### 源码解读

workerpool 用法非常简单，接下来我们一起看看其实现原理。

下图是 workerpool 源码中实现的全部功能：

![workerpool](workerpool.png)

`WorkerPool` 是一个结构体，源码中围绕这个结构体定义了很多函数或方法。这些函数或方法你不必死记硬背，先有一个宏观上的认知，接下来我将带你深入学习其中的核心方法。

`WorkerPool` 结构体完整定义如下：

```go
// WorkerPool 是 Go 协程的集合池，用于确保同时处理请求的协程数量严格受控于预设的上限值
type WorkerPool struct {
	maxWorkers   int                 // 最大工作协程数
	taskQueue    chan func()         // 任务提交队列
	workerQueue  chan func()         // 工作协程消费队列
	stoppedChan  chan struct{}       // 停止完成通知通道
	stopSignal   chan struct{}       // 停止信号通道
	waitingQueue deque.Deque[func()] // 等待队列（双端队列）
	stopLock     sync.Mutex          // 停止操作互斥锁
	stopOnce     sync.Once           // 控制只停止一次
	stopped      bool                // 是否已经停止
	waiting      int32               // 等待队列中任务计数
	wait         bool                // 协程池退出时是否等待已入队任务执行完成
}
```

这里属性很多，其中有 3 个属性是需要我们重点关注的，`taskQueue`、`workerQueue` 以及 `waitingQueue`，这三者分别代表任务提交队列、工作队列和等待队列。稍后我将通过一个流程图来讲解任务在这 3 个队列中的传递流程，现在我们一起来看一下 `WorkerPool` 的构造函数：

```go
// New 创建并启动协程池
// maxWorkers 参数指定可以并发执行任务的最大工作协程数。
func New(maxWorkers int) *WorkerPool {
	// 至少有一个 worker
	if maxWorkers < 1 {
		maxWorkers = 1
	}

	// 实例化协程池对象
	pool := &WorkerPool{
		maxWorkers:  maxWorkers,
		taskQueue:   make(chan func()),
		workerQueue: make(chan func()),
		stopSignal:  make(chan struct{}),
		stoppedChan: make(chan struct{}),
	}

	// 启动任务调度器
	go pool.dispatch()

	return pool
}
```

`New` 函数创建一个指定容量的协程池对象 `WorkerPool`，我们已经在使用示例中见过其用法了。这里逻辑还是比较简单的，仅接收一个参数，并初始化了几个必要的属性。

值得注意的是，这里通过开启新的 goroutine 的方式启动了 `dispatch()` 方法，这个方法是协程池最核心的逻辑，用来实现任务的调度执行。

为此，我画了一张流程图，来分析 `WorkerPool` 最核心的任务派发流程：

![任务派发流程](dispatch.png)

图中涉及两个方法，其中 `Submit()` 方法用于提交一个任务到协程池，`dispatch` 方法则用于派发任务到 goroutine 中去执行。`dispatch` 方法内部有一个无限循环，实现任务实时派发执行。这个 `for` 无限循环中控制着任务在 3 个队列中的流转和工作协程数量。

只要通过 `Submit()` 方法提交任务，就一定会进入任务提交队列 `taskQueue` 中，而 `taskQueue` 是一个通过 `make(chan func())` 初始化的无缓冲的 channel，所以任务不会在里面停留，要么通过链路 ② 下发到等待队列 `waitingQueue` 中，要么通过链路 ④ 下发到工作队列 `workerQueue` 中。最终具体会下发到哪里，是 `dispatch` 方法中的 `for` 循环逻辑来决定的。

`dispatch` 的 `for` 循环中会处理任务分发，核心逻辑有两个部分，包含两种处理模式：

+ 队列优先模式：在 `for` 循环中，会优先判断等待队列 `waitingQueue` 是否为空，如果不为空，则进入队列优先模式。
    1. 此时会优先从等待队列对头取出任务，然后交给工作队列 `workerQueue`，协程池中的工作协程（worker）就会不停的从 `workerQueue` 中拿到任务并执行。
    2. 如果此时刚好还有新的任务被提交，则新提交的任务自动进入等待队列尾部。
    3. 任务从提交到执行的流程是 ① ② ③。
+ 直通模式：等待队列完全清空后，程序自动切换到直通模式。
    1. 此时等待队列 `workerQueue` 已经清空，如果有新任务提交进来，可以直接交给工作队列 `workerQueue`，让工作协程（worker）来执行。
    2. 如果此时工作协程数量达到了协程池的上限，则将任务提交到等待队列 `waitingQueue` 中。
    3. 任务从提交到执行的流程是 ① ④。

以上就是协程池 `dispatch` 方法的核心调度流程。

接下来，我将对 `WorkerPool` 核心代码进行一一解读，以此来从微观上更加细致的理解协程池的设计。

`Submit` 方法实现如下：

```go
// Submit 将任务函数提交到工作池队列等待执行，不会等待任务执行完成
func (p *WorkerPool) Submit(task func()) {
	if task != nil {
		p.taskQueue <- task
	}
}
```

这个方法非常简单，就是将任务提交到 `taskQueue` 队列中。

接下来我们看下是最核心也是最复杂的方法 `dispatch` 是如何实现的：

```go
// 任务派发，循环的将下一个排队中的任务发送给可用的工作协程（worker）执行
func (p *WorkerPool) dispatch() {
	defer close(p.stoppedChan)            // 保证调度器退出时关闭停止通知通道
	timeout := time.NewTimer(idleTimeout) // 创建 2 秒周期的空闲检测定时器
	var workerCount int                   // 当前活跃 worker 计数器
	var idle bool                         // 空闲状态标识
	var wg sync.WaitGroup                 // 用于等待所有 worker 完成

Loop:
	for { // 主循环处理任务分发

		// 队列优先模式：优先检测等待队列
		if p.waitingQueue.Len() != 0 {
			if !p.processWaitingQueue() {
				break Loop // 等待队列为空，退出循环
			}
			continue
		}

		// 直通模式：开始处理提交上来的新任务
		select {
		case task, ok := <-p.taskQueue: // 接收到新任务
			if !ok { // 协程池停止时会关闭任务通道，如果 !ok 说明协程池已停止，退出循环
				break Loop
			}

			select {
			case p.workerQueue <- task: // 尝试派发任务
			default: // 没有空闲的 worker，无法立即派发任务
				if workerCount < p.maxWorkers { // 如果协程池中的活跃协程数量小于最大值，那么创建一个新的协程（worker）来执行任务
					wg.Add(1)
					go worker(task, p.workerQueue, &wg) // 创建新的 worker 执行任务
					workerCount++                       // worker 记数加 1
				} else { // 已达协程池容量上限
					p.waitingQueue.PushBack(task)                              // 将任务提交到等待队列
					atomic.StoreInt32(&p.waiting, int32(p.waitingQueue.Len())) // 原子更新等待计数
				}
			}
			idle = false // 标记为非空闲
		case <-timeout.C: // 空闲超时处理
			// 在一个空闲超时周期内，存在空闲的 workers，那么停止一个 worker
			if idle && workerCount > 0 {
				if p.killIdleWorker() { // 回收一个 worker
					workerCount-- // worker 计数减 1
				}
			}
			idle = true                // 标记为空闲
			timeout.Reset(idleTimeout) // 复用定时器
		}
	}

	if p.wait { // 调用了 StopWait() 方法，需要运行等待队列中的任务，直至队列清空
		p.runQueuedTasks()
	}

	// 终止所有 worker
	for workerCount > 0 {
		p.workerQueue <- nil // 发送终止信号给 worker
		workerCount--        // worker 计数减 1，直至为 0 退出循环
	}
	wg.Wait() // 阻塞等待所有 worker 完成

	timeout.Stop() // 停止定时器
}
```

这个方法代码量稍微有点多，不过结合我上面画的流程图，其实也好理解。我在代码注释中也标出了两种任务处理模式：等待队列优先模式和直通模式。

我们先看等待队列优先模式：

```go
// 队列优先模式：优先检测等待队列
if p.waitingQueue.Len() != 0 {
    if !p.processWaitingQueue() {
        break Loop // 协程池已经停止
    }
    continue // 队列不为空则继续下一轮循环
}
```

如果等待队列不为空，则优先处理等待队列。`p.processWaitingQueue` 方法实现如下：

```go
// 处理等待队列
func (p *WorkerPool) processWaitingQueue() bool {
	select {
	case task, ok := <-p.taskQueue: // 接收到新任务
		if !ok { // 协程池停止时会关闭任务通道，如果 !ok 说明协程池已停止，返回 false，不再继续处理
			return false
		}
		p.waitingQueue.PushBack(task) // 将新任务加入等待队列队尾
	case p.workerQueue <- p.waitingQueue.Front(): // 从等待队列队头获取任务并放入工作队列
		p.waitingQueue.PopFront() // 任务已经开始处理，所以要从等待队列中移除任务
	}
	atomic.StoreInt32(&p.waiting, int32(p.waitingQueue.Len())) // 原子修改等待队列中任务计数
	return true
}
```

这个方法中有两个 case 需要处理：

1. 接收到新任务，直接加入到等待队列 `waitingQueue` 的队尾。
2. 从等待队列 `waitingQueue` 的队头获取任务并放入工作队列 `workerQueue`。

这与前文流程图中的讲解吻合。

任务交给工作队列 `workerQueue` 以后，谁来处理 `workerQueue` 中的任务呢？我们接着往下看直通模式的代码。

直通模式的代码中同样使用 select 多路复用，将逻辑分成了两个 case 来处理：

```go
// 直通模式：开始处理提交上来的新任务
select {
case task, ok := <-p.taskQueue: // 接收到新任务
    ...
case <-timeout.C: // 空闲超时处理
    ...
}
```

两个 case 分别实现任务执行和空闲超时处理。

我们先来看处理任务的 case：

```go
case task, ok := <-p.taskQueue: // 接收到新任务
    if !ok { // 协程池停止时会关闭任务通道，如果 !ok 说明协程池已停止，退出循环
        break Loop
    }

    select {
    case p.workerQueue <- task: // 尝试派发任务
    default: // 没有空闲的 worker，无法立即派发任务
        if workerCount < p.maxWorkers { // 如果协程池中的活跃协程数量小于最大值，那么创建一个新的协程（worker）来执行任务
            wg.Add(1)
            go worker(task, p.workerQueue, &wg) // 创建新的 worker 执行任务
            workerCount++                       // worker 记数加 1
        } else { // 已达协程池容量上限
            p.waitingQueue.PushBack(task)                              // 将任务提交到等待队列
            atomic.StoreInt32(&p.waiting, int32(p.waitingQueue.Len())) // 原子更新等待计数
        }
    }
    idle = false // 标记为非空闲
```

直通模式下，有新的任务提交进来，首先会尝试直接将其加入工作队列 `workerQueue` 中，如果任务下发失败，则说明当前时刻没有空闲的工作协程（worker），无法立即派发任务。那么继续比较当前正在执行的工作协程数量（workerCount）和协程池大小（maxWorkers），如果协程池中的活跃协程数量小于最大值，那么创建一个新的协程（worker）来执行任务。否则，说明正在执行的工作协程数量已达协程池容量上限，那么将任务提交到等待队列 `waitingQueue` 中。那么下一次 `for` 循环执行的时候，检测到 `waitingQueue` 中有任务，就会优先处理 `waitingQueue`。这也就实现了两种模式的切换。

我们再来看下工作协程 `worker` 是如何执行任务的：

```go
// 工作协程，执行任务并在收到 nil 信号时停止
func worker(task func(), workerQueue chan func(), wg *sync.WaitGroup) {
	for task != nil { // 循环执行任务，直至接收到终止信号 nil
		task()               // 执行任务
		task = <-workerQueue // 接收新任务
	}
	wg.Done() // 标记 worker 完成
}
```

可以发现，这里使用 `for` 循环来不停的执行提交过来的任务，直至从 `workerQueue` 中接收到终止信号 `nil`。那么这个终止信号是何时下发的呢？往下看你马上能找到答案。

接下来我们看一下直通模式的另外一个 case 逻辑：

```go
case <-timeout.C: // 空闲超时处理
    // 在一个空闲超时周期内，存在空闲的 workers，那么停止一个 worker
    if idle && workerCount > 0 {
        if p.killIdleWorker() { // 回收一个 worker
            workerCount-- // worker 计数减 1
        }
    }
    idle = true                // 标记为空闲
    timeout.Reset(idleTimeout) // 复用定时器
```

这里使用定时器来管理超过特定时间，未收到任务，需要关闭空闲的工作协程（worker）。

关闭 `worker` 的方法是 `p.killIdleWorker`：

```go
// 停止一个空闲 worker
func (p *WorkerPool) killIdleWorker() bool {
	select {
	case p.workerQueue <- nil: // 发送终止信号给工作协程（worker）
		// Sent kill signal to worker.
		return true
	default:
		// No ready workers. All, if any, workers are busy.
		return false
	}
}
```

这里正是通过给 `workerQueue` 发送 `nil` 来作为终止信号，以此来实现通知 `worker` 退出的。

看完了 `Submit` 和 `dispatch` 方法源码，你现在是否对协程池有了更深入的认知呢？你可以再回顾一下我在前文中画的任务调度流程图，加深印象。

workerpool 的源码就讲解到这里，其他方法实现其实都比较简单，就交给你自己去探索了。你可以参考我的中文注释版源码：[https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/workerpool/workerpool.go](https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/workerpool/workerpool.go)。

### 总结

协程池作为 Go 中不那么常用的技术，依然有其存在的价值，本文介绍的 workerpool 项目是一个协程池的实现。

workerpool 用法非常简单，仅需要通过 `workerpool.New(n)` 函数既可创建一个大小为 `n` 的协程池，之后通过 `wp.Submit(task)` 既可以提交任务到协程池。

workerpool 内部提供了 3 个队列来对任务进行派发调度，任务提交队列 `taskQueue` 和工作队列 `workQueue` 都是使用 channel 实现的，并且无缓冲，真正带有缓冲效果的队列是等待队列 `WaitingQueue`，这个是真正的队列实现，采用双端队列，而非 channel，并且不限制队列长度。也就是说，无论排队多少个任务，workerpool 都不会阻止新任务的提交。所以，我们在创建协程池时需要设置一个合理的大小限制，以防止等待队列无限增长，任务很长一段时间都得不到执行。

此外，workerpool 内部虽然会维护一个协程池，但超过一定空闲时间没有任务提交过来，工作协程是会关闭的，之后新任务进来再次启动新的协程，因为启动新协程开销小，所以没长久驻留协程。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/goroutine/workerpool) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ workerpool GitHub 源码：[https://github.com/gammazero/workerpool](https://github.com/gammazero/workerpool)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/workerpool/workerpool.go](https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/workerpool/workerpool.go)
+ 本文永久地址：[https://jianghushinian.cn/2025/05/09/goroutine-workerpool/](https://jianghushinian.cn/2025/05/09/goroutine-workerpool/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

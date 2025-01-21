---
title: 'Go 并发控制：semaphore 详解'
date: 2025-01-21 23:33:45
tags:
- Go
- 并发编程
categories:
- Go
---

今天我们来介绍一个 [Go 官方库 x](https://go.dev/wiki/X-Repositories) 提供的扩展并发原语 `semaphore`，译为“信号量”。因为它就像一个信号一样控制多个 goroutine 之间协作。

<!-- more -->

### 概念讲解

我先简单介绍下信号量的概念，为不熟悉的读者作为补充知识。

一个生活中的例子：假设一个餐厅总共有 10 张餐桌，每来 1 位顾客占用 1 张餐桌，那么同一时间共计可以有 10 人在就餐，超过 10 人则需要排队等位；如果有 1 位顾客就餐完成，则可以让排队等待的第 1 位顾客来就餐。

如果加入信号量，则可以这样理解：

+ 10 个餐桌是有限的资源，**即信号量初始值为 10**。
+ 当顾客进入餐厅，如果餐桌有空，顾客可以被分配 1 个餐桌，**信号量的值减 1**。
+ 如果没有空餐桌，顾客需要排队等待，直到有空餐桌为止（**信号量值为 0 时**，新的顾客必须等待）。
+ 有顾客就餐完成后，餐桌空出，**信号量的值加 1**，新的顾客（排队等待中的顾客，或者无人排队时新来的顾客）可以被分配餐桌。

这就是信号量的应用场景，信号量帮助管理餐桌的数量，确保餐厅在任何时刻不会接待超过可容纳的顾客数量。

如果放在 Go 程序中，我们就可以使用信号量来实现任务池的功能，控制有限个 goroutine 来并发执行任务。这个我们放在后面来实现，先看下 `semaphore` 源码是如何实现的。

### 源码解读

我们可以在 `semaphore` 文档中看到其定义和实现的 exported 方法：

> [https://pkg.go.dev/golang.org/x/sync@v0.10.0/semaphore#pkg-index](https://pkg.go.dev/golang.org/x/sync@v0.10.0/semaphore#pkg-index)
>

```go
type Weighted
    func NewWeighted(n int64) *Weighted
    func (s *Weighted) Acquire(ctx context.Context, n int64) error
    func (s *Weighted) Release(n int64)
    func (s *Weighted) TryAcquire(n int64) bool
```

`semaphore.Weighted` 是一个结构体表示信号量对象，`NewWeighted` 是它的构造函数，用于实例化一个包含 `n` 个资源的信号量对象，即信号量的初始值为 `n`。

它实现了 3 个方法，分别是：

+ `Acquire`：用来请求 `n` 个资源，即对信号量的值进行减 `n` 操作。如果资源不足，则阻塞等待，直到有足够的资源数，或者 `ctx` 被取消。
+ `Release`：用来释放 `n` 个资源，即对信号量的值进行加 `n` 操作。
+ `TryAcquire`：用来请求 `n` 个资源，即对信号量的值进行减 `n` 操作。与 `Acquire` 不同的是，`TryAcquire` 不会阻塞等待，成功返回 `true`，失败返回 `false`。

现在，你是否能将 `semaphore.Weighted` 对象和它实现的几个方法与前文中餐厅就餐的例子对应上了呢？

接下来我们看一下 `Weighted` 结构体是如何定义的：

> [https://github.com/golang/sync/blob/v0.9.0/semaphore/semaphore.go](https://github.com/golang/sync/blob/v0.9.0/semaphore/semaphore.go)
>

```go
// Weighted 信号量结构体
type Weighted struct {
	size    int64      // 资源总数量
	cur     int64      // 当前已经使用的资源数
	mu      sync.Mutex // 互斥锁，保证对其他属性的操作并发安全
	waiters list.List  // 等待者队列，使用列表实现
}
```

`Weighted` 结构体包含 4 个字段：

+ `size` 是资源总数。在餐厅的例子中就是 10 张餐桌。
+ `cur` 记录了当前已经使用的资源数。在餐厅的例子中就是已经被顾客占用的餐桌数，`size - cur` 就是剩余资源数，即空餐桌数。
+ `mu` 是互斥锁，用于保证对其他属性的操作是并发安全的。
+ `waiters` 是等待者队列，记录所有排队的等待者。在餐厅的例子中，当 10 张餐桌都被占满，新来的顾客就要进入等待者队列。

构造函数 `NewWeighted` 实现如下：

```go
// NewWeighted 构造一个信号量对象
func NewWeighted(n int64) *Weighted {
	w := &Weighted{size: n}
	return w
}
```

此外，`semaphore` 包还为等待者定义了一个结构体 `waiter`：

```go
// 等待者结构体
type waiter struct {
	n     int64           // 请求资源数
	ready chan<- struct{} // 当获取到资源时被关闭，用于唤醒当前等待者
}
```

`waiter` 结构体包含 2 个字段：

+ `n` 记录了当前等待者请求的资源数。
+ `ready` 是一个 `channel` 类型，当资源满足，`ready` 会被 `close` 掉，等待者就会被唤醒。

稍后你将看到它的用处。

我们回过头来看下 `Weighted` 结构体的第一个方法 `Acquire` 的实现：

```go
// Acquire 请求 n 个资源
// 如果资源不足，则阻塞等待，直到有足够的资源数，或者 ctx 被取消
// 成功返回 nil，失败返回 ctx.Err() 并且不改变资源数
func (s *Weighted) Acquire(ctx context.Context, n int64) error {
	done := ctx.Done()

	s.mu.Lock() // 加锁保证并发安全

	// 如果在分配资源前 ctx 已经取消，则直接返回 ctx.Err()
	select {
	case <-done:
		s.mu.Unlock()
		return ctx.Err()
	default:
	}

	// 如果资源数足够，且不存在其他等待者，则请求资源成功，将 cur 加上 n，并返回
	if s.size-s.cur >= n && s.waiters.Len() == 0 {
		s.cur += n
		s.mu.Unlock()
		return nil
	}

	// 如果请求的资源数大于资源总数，不可能满足，则阻塞等待 ctx 取消，并返回 ctx.Err()
	if n > s.size {
		// Don't make other Acquire calls block on one that's doomed to fail.
		s.mu.Unlock()
		<-done
		return ctx.Err()
	}

	// 资源不够或者存在其他等待者，则继续执行

	// 加入等待队列
	ready := make(chan struct{})    // 创建一个 channel 作为一个属性记录到等待者对象 waiter 中，用于后续通知其唤醒
	w := waiter{n: n, ready: ready} // 构造一个等待者对象 waiter
	elem := s.waiters.PushBack(w)   // 将 waiter 追加到等待者队列
	s.mu.Unlock()

	// 使用 select 实现阻塞等待
	select {
	case <-done: // 检查 ctx 是否被取消
		s.mu.Lock()
		select {
		case <-ready: // 检查当前 waiter 是否被唤醒
			// 进入这里，说明是 ctx 被取消后 waiter 被唤醒
			s.cur -= n        // 那么就当作 waiter 没有被唤醒，将请求的资源数还回去
			s.notifyWaiters() // 通知等待队列，检查队列中下一个 waiter 资源数是否满足
		default: // 将当前 waiter 从等待者队列中移除
			isFront := s.waiters.Front() == elem // 当前 waiter 是否为第一个等待者
			s.waiters.Remove(elem)               // 从队列中移除
			// 如果当前 waiter 是队列中第一个等待者，并且还有剩余的资源数
			if isFront && s.size > s.cur {
				s.notifyWaiters() // 通知等待队列，检查队列中下一个 waiter 资源数是否满足
			}
		}
		s.mu.Unlock()
		return ctx.Err()

	case <-ready: // 被唤醒
		select {
		case <-done: // 再次检查 ctx 是否被取消
			// 进入这里，说明 waiter 被唤醒后 ctx 却被取消了，当作未被唤醒来处理
			s.Release(n) // 释放资源
			return ctx.Err()
		default:
		}
		return nil // 成功返回 nil
	}
}
```

这是 `Weighted` 结构体实现最为复杂的一个方法，不过我在代码中写了非常详细的注释，来帮助你理解。

`Acquire` 主要逻辑为：

+ 先检查 `ctx` 是否被取消，如果在分配资源前 `ctx` 已经取消，则直接返回 `ctx.Err()`。
+ 检查当前剩余资源数是否满足请求的资源数，如果资源数足够，且不存在其他等待者，则请求资源成功，将 `cur` 加上 `n`，并返回。
+ 校验请求的资源数是否合法，如果请求的资源数大于资源总数，则不可能被满足，此时会阻塞等待 `ctx` 被取消，并返回 `ctx.Err()`。
+ 如果目前资源不够，或者存在其他等待者，则代码将继续执行，进入阻塞等待逻辑：
    - 首先构造一个等待者对象 `waiter`，并将其加入等待者队列。
    - 接着，使用 `select` 实现阻塞等待。此时又分两种情况，`ctx` 被取消，或者 `waiter` 被唤醒。
        * 如果是 `ctx` 被取消，还会再检查一下 `waiter` 是否被唤醒，如果被唤醒，**则还是以 `ctx` 被取消为准**，会当作 `waiter` 没有被唤醒，将请求的资源数还回去，并调用 `s.notifyWaiters()` 通知等待者队列，检查队列中下一个 `waiter` 资源数是否满足；如果没被唤醒，则将当前 `waiter` 从等待者队列中移除，如果当前 `waiter` 是队列中第一个等待者，并且还有剩余的资源数，则还会调用 `s.notifyWaiters()` 通知等待者队列，检查队列中下一个 `waiter` 资源数是否满足。
        * 如果是 `waiter` 被唤醒，还会再检查一下 `ctx` 是否被取消，如果被取消，**则以 `ctx` 被取消为准**，会调用 `s.Release(n)` 释放掉当前 `waiter` 请求的资源；如果没被取消，则返回 `nil` 表示请求资源成功。

那么接下来，我们看看释放资源的 `Release` 方法逻辑：

```go
// Release 释放 n 个资源
func (s *Weighted) Release(n int64) {
	s.mu.Lock() // 加锁保证并发安全
	s.cur -= n  // 释放资源
	if s.cur < 0 {
		s.mu.Unlock()
		panic("semaphore: released more than held")
	}
	s.notifyWaiters() // 通知等待队列，检查队列中下一个 waiter 资源数是否满足
	s.mu.Unlock()
}
```

这里的逻辑就简单多了，首先执行 `s.cur -= n` 减少当前已经使用的资源数，即这一步就是释放资源操作。注意这里还对` s.cur` 是否小于 0 进行了判断，所以使用时一定是申请多少资源就释放多少资源，不要用错。接着同样调用 `s.notifyWaiters()` 通知等待者队列，检查队列中下一个 `waiter` 资源数是否满足。

现在，是时候看看 `notifyWaiters` 方法是如何实现的了：

```go
// 检查队列中下一个 waiter 资源数是否满足
func (s *Weighted) notifyWaiters() {
	// 循环检查下一个 waiter 请求的资源数是否满足，满足则出队，不满足则终止循环
	for {
		next := s.waiters.Front() // 获取队首元素
		if next == nil {
			break // 没有 waiter 了，队列为空终止循环
		}

		w := next.Value.(waiter)
		if s.size-s.cur < w.n { // 当前 waiter 资源数不满足，退出循环
			// 不继续查找队列中后续 waiter 请求资源是否满足，避免产生饥饿
			break
		}

		// 资源数满足，唤醒 waiter
		s.cur += w.n           // 记录使用的资源数
		s.waiters.Remove(next) // 从队列中移除 waiter
		close(w.ready)         // 利用关闭 channel 的操作，来唤醒 waiter
	}
}
```

`notifyWaiters` 方法内部的核心逻辑是：循环检查下一个 `waiter` 请求的资源数是否满足，满足则出队，不满足则终止循环。

可以发现，`next` 是队首元素，所以说等待者队列是先进先出（`FIFO`）的。

这里还有一个要注意的点是，当前 `waiter` 资源数不满足时，直接退出循环，而不再继续查找队列中后续 `waiter` 请求资源是否满足。比如当前可用资源数为 5，等待者队列中有两个等待者 `waiter1` 和 `waiter2`，`waiter1` 请求的资源数是 10，`waiter2` 请求的资源数是 1，此时就会退出循环，`waiter1` 和 `waiter2` 都继续等待，而不会先将资源分配给 `waiter2`。这样做是为了避免之后的 `waiter3`、`waiter4`... 总是比 `waiter1` 请求的资源数小，而导致 `waiter1` 长时间阻塞，从而产生饥饿。

`Weighted` 还有最后一个方法 `TryAcquire` 我们一起来看看是如何实现的：

```go
// TryAcquire 尝试请求 n 个资源
// 不阻塞，成功返回 true，失败返回 false 并且不改变资源数
func (s *Weighted) TryAcquire(n int64) bool {
	s.mu.Lock() // 加锁保证并发安全
	// 剩余资源数足够，且不存在其他等待者，则请求资源成功
	success := s.size-s.cur >= n && s.waiters.Len() == 0
	if success {
		s.cur += n // 记录当前已经使用的资源数
	}
	s.mu.Unlock()
	return success
}
```

这个方法实现也很简单，就是检查剩余资源数是否足够，且不存在其他等待者，如果满足，则请求资源成功，增加当前已经使用的资源数 `s.cur += n`，然后返回 `true`，否则返回 `false`。

至此，`semaphore` 包的源码就讲解完成了。

### 使用示例

熟悉了 `semaphore` 包的源码，那么如何使用就不在话下了。如下是 `semaphore` 包文档中提供的 `worker pool` 模式示例，演示如何使用信号量来限制并行任务中运行的 goroutine 数量。代码如下：

> [https://pkg.go.dev/golang.org/x/sync@v0.10.0/semaphore#example-package-WorkerPool](https://pkg.go.dev/golang.org/x/sync@v0.10.0/semaphore#example-package-WorkerPool)
>

```go
package main

import (
	"context"
	"fmt"
	"log"
	"runtime"

	"golang.org/x/sync/semaphore"
)

func main() {
	ctx := context.TODO()

	var (
		maxWorkers = runtime.GOMAXPROCS(0)                    // worker pool 支持的最大 worker 数量，取当前机器 CPU 核心数
		sem        = semaphore.NewWeighted(int64(maxWorkers)) // 信号量，资源总数即为最大 worker 数量
		out        = make([]int, 32)                          // 总任务数量
	)

	// 一次最多启动 maxWorkers 数量个 goroutine 计算输出
	for i := range out {
		// 当最大工作数 maxWorkers 个 goroutine 正在执行时，Acquire 会阻塞直到其中一个 goroutine 完成
		if err := sem.Acquire(ctx, 1); err != nil { // 请求资源
			log.Printf("Failed to acquire semaphore: %v", err)
			break
		}

		// 开启新的 goroutine 执行计算任务
		go func(i int) {
			defer sem.Release(1)         // 任务执行完成后释放资源
			out[i] = collatzSteps(i + 1) // 执行 Collatz 步骤计算
		}(i)
	}

	// 获取所有的 tokens 以等待全部 goroutine 执行完成
	if err := sem.Acquire(ctx, int64(maxWorkers)); err != nil {
		log.Printf("Failed to acquire semaphore: %v", err)
	}

	fmt.Println(out)

}

// collatzSteps computes the number of steps to reach 1 under the Collatz
// conjecture. (See https://en.wikipedia.org/wiki/Collatz_conjecture.)
func collatzSteps(n int) (steps int) {
	if n <= 0 {
		panic("nonpositive input")
	}

	for ; n > 1; steps++ {
		if steps < 0 {
			panic("too many steps")
		}

		if n%2 == 0 {
			n /= 2
			continue
		}

		const maxInt = int(^uint(0) >> 1)
		if n > (maxInt-1)/3 {
			panic("overflow")
		}
		n = 3*n + 1
	}

	return steps
}
```

这段代码利用信号量（`semaphore`）实现一个工作池（`worker pool`），来限制最大并发协程（`goroutine`）数。

通过 `runtime.GOMAXPROCS(0)` 可以获取当前机器 CPU 核心数（比如在我的 Apple M1 Pro 机器中这个值是 10），以此作为 `worker pool` 支持的最大 `worker` 数量，这也是信号量的资源总数。`out` 切片长度为 32，即总任务数为 32。使用 `for` 循环来开启新的 goroutine 执行计算任务 `collatzSteps(i + 1)`，不过在开启新的 goroutine 前，会调用 `sem.Acquire(ctx, 1)` 向 `worker pool` 申请一个资源，当最大工作数 `maxWorkers` 个 goroutine 正在执行时，`Acquire` 会阻塞直到其中一个 goroutine 完成，任务执行完成后在 `defer` 语句中调用 `sem.Release(1)` 释放资源。

> NOTE:
>
> 关于计算任务 `collatzSteps` 我们其实不必深究，只需要知道这是一个耗时任务即可。简单来说，Collatz Conjecture 是一个数学猜想，提出了一个看似简单的整数序列规则，对于任意正整数 `n` 执行操作，如果 `n` 是偶数，则将 `n` 除以 2，如果 `n` 是奇数，则将 `n` 乘以 3 再加 1（即 `3n + 1`），反复执行这些操作后，最终所有整数都将达到 1。`collatzSteps` 函数实现了计算在 Collatz 猜想下一个正整数 `n` 需要多少步才能达到 1。如果你实在感兴趣可以点击链接 [https://en.wikipedia.org/wiki/Collatz_conjecture](https://en.wikipedia.org/wiki/Collatz_conjecture) 了解一下。
>

值得注意的是，在 `for` 循环提交完所有任务后，使用 `sem.Acquire(ctx, int64(maxWorkers))` 获取了信号量中全部资源，这样就能够确保所有任务执行完成后才会退出，它的作用类似 `sync.WaitGroup.Wait()`。

那么接下来我们使用 `sync.WaitGroup` 来实现同样的功能：

> https://github.com/jianghushinian/blog-go-example/blob/main/x/sync/semaphore/waitgroup/main.go
> 
```go
func main() {
	var (
		maxWorkers = runtime.GOMAXPROCS(0)           // 获取系统可用的最大 CPU 核心数
		out        = make([]int, 32)                 // 存储 Collatz 结果
		wg         sync.WaitGroup                    // 用于等待 goroutine 完成
		sem        = make(chan struct{}, maxWorkers) // 用于限制最大并发数
	)

	for i := range out {
		// 通过 sem 管理并发，确保最多只有 maxWorkers 个 goroutine 同时执行
		sem <- struct{}{} // 如果 sem 已满，这里会阻塞，直到有空闲槽位

		// 增加 WaitGroup 计数
		wg.Add(1)

		go func(i int) {
			defer wg.Done()          // goroutine 完成时，减少 WaitGroup 计数
			defer func() { <-sem }() // goroutine 完成时，从 sem 中释放一个槽位

			// 执行 Collatz 步骤计算
			out[i] = collatzSteps(i + 1)
		}(i)
	}

	// 等待所有 goroutine 完成
	wg.Wait()

	// 输出结果
	fmt.Println(out)
}
```

现在对比一下使用 `sync.WaitGroup` 和 `semaphore` 实现的代码，是否能加深你对信号量这一功能的理解呢？

最后，我们再来留个作业，请使用 `errgroup` 实现一遍这个示例程序。

> NOTE:
>
> 如果你对 `sync.WaitGroup` 或 `errgroup` 不熟悉，可以参考我的文章「[Go 并发控制：sync.WaitGroup 详解](https://jianghushinian.cn/2024/12/23/sync-waitgroup/)」和「[Go 并发控制：errgroup 详解](https://jianghushinian.cn/2024/11/04/x-sync-errgroup/)」。
>

### 总结

本文对 Go 中的扩展并发原语 `semaphore` 进行了讲解，并带你看了其源码的实现，以及介绍了如何使用。

不知道你有没有发现，在初始化 `semaphore` 时，传递的 `n` 如果为 1，那么这个信号量其实就相当于互斥锁 `sync.Mutex`。

`semaphore` 可以用来实现 `worker pool` 模式，并且使用套路或场景与 `sync.WaitGroup` 也比较相似，你可以对比学习。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/x/sync/semaphore) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Go 并发控制：sync.WaitGroup 详解：[https://jianghushinian.cn/2024/12/23/sync-waitgroup/](https://jianghushinian.cn/2024/12/23/sync-waitgroup/)
+ Go 并发控制：errgroup 详解：[https://jianghushinian.cn/2024/11/04/x-sync-errgroup/](https://jianghushinian.cn/2024/11/04/x-sync-errgroup/)
+ semaphore Documentation：[https://pkg.go.dev/golang.org/x/sync@v0.10.0/semaphore](https://pkg.go.dev/golang.org/x/sync@v0.10.0/semaphore)
+ semaphore GitHub 源码：[https://github.com/golang/sync/blob/v0.9.0/semaphore/semaphore.go](https://github.com/golang/sync/blob/v0.9.0/semaphore/semaphore.go)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/x/sync/semaphore](https://github.com/jianghushinian/blog-go-example/tree/main/x/sync/semaphore)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

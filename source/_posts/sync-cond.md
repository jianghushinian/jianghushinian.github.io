---
title: 'Go 并发控制：sync.Cond 详解'
date: 2024-12-29 23:11:50
tags:
- Go
- 并发编程
categories:
- Go
---

在 Go 中因为 channel 的存在，`sync.Cond` 并发原语并不常用。不过在一些开源组件中还能能见到 `sync.Cond` 的应用，比如 Kubernetes 用它来实现并发等待队列，这也是 `sync.Cond` 的典型应用场景。本文将通过源码和示例带你学会 `sync.Cond` 的正确用法。

<!-- more -->

### 源码解读

我们可以在 `sync.Cond` 文档 [https://pkg.go.dev/sync@go1.23.0#Cond](https://pkg.go.dev/sync@go1.23.0#Cond) 中看到其定义和实现的 exported 方法：

```go
type Cond
    func NewCond(l Locker) *Cond
    func (c *Cond) Broadcast()
    func (c *Cond) Signal()
    func (c *Cond) Wait()
```

`sync.Cond` 是一个结构体，`NewCond` 函数接收一个互斥锁对象 `sync.Locker` 并构造一个 `sync.Cond` 结构体指针。这里的互斥锁对象 `sync.Locker` 是一个接口，`sync.Mutex` 和 `sync.RWMutex` 都实现了此接口。这把互斥锁是 `sync.Cond` 实现的关键所在，也是导致 `sync.Cond` 经常容易用错的“罪魁祸首”。其他 3 个方法，就是 `sync.Cond` 提供给我们用来进行并发控制的全部方法了。

你可能猜到了，`Cond` 这个单词是 `Condition` 的缩写，所以 `sync.Cond` 并发原语是围绕着**一个条件**来设计的，**它的主要功能就是通过一个条件来实现阻塞和唤醒一组需要协作的 goroutine**。

当调用 `Wait` 方法时，当前 goroutine 会被阻塞，直到被其他 goroutine 中调用的 `Broadcast` 或 `Signal` 方法唤醒。这看起来有点类似 `sync.WaitGroup`，不过却不太一样，等阅读完了本文，你就能明白二者的差异了。

> NOTE:
>
> 如果你对 `sync.WaitGroup` 不熟悉，可以参考我的另一篇文章[《Go 并发控制：sync.WaitGroup 详解》](https://jianghushinian.cn/2024/12/23/sync-waitgroup/)。
>

目前，我们并没有看到所谓的**条件**到底是什么，这个概念似乎有点抽象，别着急，一会你就明白了。那么接下来，我们对 `sync.Cond` 源码进行进一步解读。

`sync.Cond` 构造函数实现如下：

> [https://github.com/golang/go/blob/go1.23.0/src/sync/cond.go](https://github.com/golang/go/blob/go1.23.0/src/sync/cond.go)
>

```go
func NewCond(l Locker) *Cond {
	return &Cond{L: l}
}
```

这里没什么逻辑，只是记录了传进来的 `sync.Locker` 对象。

`sync.Cond` 结构体定义如下：

```go
type Cond struct {
	noCopy noCopy

	// L is held while observing or changing the condition
	L Locker

	notify  notifyList
	checker copyChecker
}
```

首先，看过我写的前几篇并发编程相关文章的读者对 `noCopy` 属性应该再熟悉不过了，就是用来防止 `sync.Cond` 结构体被复制用的。不了解的读者可以看一下我的另一篇文章[《Go 中空结构体惯用法，我帮你总结全了！》](https://jianghushinian.cn/2024/06/02/i-have-summarized-all-the-usages-of-empty-struct-in-go-for-you/)。

接着，就是互斥锁属性字段 `L`，根据注释可知，在检查或者修改 `condition`（即条件）时需要持有锁。

`notify` 属性则是用来记录被阻塞等待的队列，它的主要作用是维护一个通知列表，用于在调用 `Wait` 和 `Signal/Broadcast` 时高效地协调 goroutines 的阻塞和唤醒。其定义如下：

```go
type notifyList struct {
	wait   uint32         // 当前进入等待状态的 goroutine 数量
	notify uint32         // 当前已被通知可以继续运行的 goroutine 数量
	lock   uintptr        // 用于保护 wait/notify 的锁
	head   unsafe.Pointer // 等待队列的头指针
	tail   unsafe.Pointer // 等待队列的尾指针
}
```

可以看出 `notifyList` 是一个链表实现。

现在还剩下最后一个属性 `checker`，这个属性同样用于防止 `sync.Cond` 结构体被复制。`noCopy` 类型可以辅助 `go vet` 工具在编译时做静态类型检查，而 `copyChecker` 则可以在运行时进行动态检查。其实现如下：

```go
type copyChecker uintptr

// 检查是否被复制，如果发生复制，则触发 panic
func (c *copyChecker) check() {
	if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
		!atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
		uintptr(*c) != uintptr(unsafe.Pointer(c)) {
		panic("sync.Cond is copied")
	}
}
```

接下来我们看下 `sync.Cond` 实现的 3 个方法：

```go
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}

func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify)
}

func (c *Cond) Broadcast() {
	c.checker.check()
	runtime_notifyListNotifyAll(&c.notify)
}
```

没错，`sync.Cond` 这 3 个方法竟然只有这么几行代码。这得益于 Go 将复杂的逻辑都交给更底层的 `runtime` 包去实现了。不过我们大可不必深究更底层的实现，我们只需要梳理清楚这里面的逻辑，就能理解 `sync.Cond` 的设计和用法了。

首先，这 3 个方法有一个共性，就是都会调用 `c.checker.check()` 来检查对象 `Cond` 是否被复制，检查过后才是主逻辑。

`Wait` 方法会先调用 `runtime_notifyListAdd` 函数将调用者加入到通知列表中，接着通过 `c.L.Unlock()` 释放锁，然后再调用 `runtime_notifyListWait` 函数阻塞并等待通知，收到通知被唤醒后，则继续执行 `c.L.Lock()` 进行加锁操作。

所以，根据 `Wait` 方法的实现，我们大概可以猜到它的使用方法：

```go
c.L.Lock() // 调用 Wait 前先加锁
// ...
c.Wait() // 调用 Wait 阻塞等待通知，其内部先释放锁，然后阻塞等待，最后被唤醒时重新加锁
// ...
c.L.Unlock() // 调用 Wait 后需要释放锁
```

`Signal` 方法内部调用了 `runtime_notifyListNotifyOne` 函数来通知唤醒一个在通知列表中的调用者，并将其从通知列表中移除。

而 `Broadcast` 方法内部则调用了 `runtime_notifyListNotifyAll` 函数来通知唤醒整个通知列表中的全部调用者，并清空通知列表。

`sync.Cond` 源码要讲解的内容就这么多，现在我再来讲解 `sync.Cond` 的用法你就好理解了。

示例代码如下：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/main.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/main.go)
>

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	c := sync.NewCond(&sync.Mutex{})

	go func() { // 启动一个子 goroutine 进行等待
		fmt.Println("wait before")
		c.L.Lock()
		c.Wait() // 阻塞并等待通知
		c.L.Unlock()
		fmt.Println("wait after")
	}()

	time.Sleep(time.Second)

	fmt.Println("signal before")
	c.Signal() // 通知唤醒一个阻塞的 goroutine
	fmt.Println("signal after")

	time.Sleep(time.Second) // 确保子 goroutine 执行完成再退出
}
```

这个示例非常简单，启动一个子 goroutine 并调用 `c.Wait()` 进行等待，这会使当前子 goroutine 被加入到通知列表并阻塞。在主 goroutine 中等待 1 秒以后调用 `c.Signal()` 来通知唤醒一个阻塞的 goroutine，这个示例中只有一个 goroutine 在通知列表中，即上面启动的子 goroutine，所以必然是它被唤醒。程序最后会等待 1 秒中，确保子 goroutine 执行完成再退出。

执行示例代码，得到如下输出：

```bash
$ go run main.go
wait before
signal before
signal after
wait after
```

输出结果符合预期。

但是，上面的示例并没有发挥 `sync.Cond` 并发原语真正的作用，这个简单的示例使用 channel 实现似乎更加合理。并且，你有没有注意到，这里根本就没体现出**条件**这个概念。

我们重新实现一个 `sync.Cond` 使用示例如下：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/main.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/main.go)
>

```go
func main() {
	c := sync.NewCond(&sync.Mutex{})
	condition := false // 定义一个条件变量

	go func() { // 启动一个子 goroutine 进行等待
		fmt.Println("wait before")
		c.L.Lock()
		for !condition { // 通过循环检查条件是否满足
			c.Wait() // 阻塞并等待通知
		}
		fmt.Println("condition met, continue execution")
		c.L.Unlock()
		fmt.Println("wait after")
	}()

	time.Sleep(time.Second)

	fmt.Println("signal before")
	c.L.Lock()
	condition = true // 改变条件变量的状态
	c.L.Unlock()
	c.Signal() // 通知唤醒一个阻塞的 goroutine
	fmt.Println("signal after")

	time.Sleep(time.Second) // 确保子 goroutine 执行完成再退出
}
```

这个示例程序，在原有示例代码的基础上，增加了条件变量 `condition`。在子 goroutine 中调用 `c.Wait()` 之前使用 `for` 循环检查条件是否满足，如果不满足，才会调用 `c.Wait()` 进行阻塞等待，否则，条件满足则直接执行后续逻辑。

在主 goroutine 中调用 `c.Signal()` 之前，加锁保护并修改了条件变量 `condition` 的值为 `true`，然后才会通知唤醒被阻塞的子 goroutine。

子 goroutine 被唤醒后，`for !condition` 判断条件不在成立，程序会退出循环向下继续执行。

这才是 `sync.Cond` 的典型实用场景。

执行示例代码，得到如下输出：

```bash
$ go run main.go
wait before
signal before
signal after
condition met, continue execution
wait after
```

其实，在 `sync.Cond` 源码中，`Wait` 方法注释部分已经给出正确用法：

```go
//	c.L.Lock()
//	for !condition() {
//	    c.Wait()
//	}
//	... make use of condition ...
//	c.L.Unlock()
```

所以，`Wait` 方法的正确用法是结合**互斥锁**和**条件变量**一起来使用的。

那么，`Wait` 方法为什么要这样设计呢？其实就是为了让用户能够在并发安全的情况下，对某个条件进行检查，来决定是否继续等待还是向下执行代码。而在调用 `Signal/Broadcast` 方法前，能够在并发安全的情况下对条件变量进行修改。

那么到现在，再回过头理解为什么构造 `sync.Cond` 对象时需要一把互斥锁这件事，也就说的通了。这把锁由调用方传递进来，那么调用方就可以借助这把锁，并发安全的修改条件变量。而在 `Wait` 方法内部，会在调用 `runtime_notifyListWait` 函数阻塞当前 goroutine 之前释放锁，这样用户才有机会使用同一把锁，在业务代码中，加锁并修改条件变量。然后调用 `Signal/Broadcast` 来通知唤醒 `Wait` 方法。`Wait` 方法唤醒后，会再次加锁，这样我们在外层使用 `for !condition` 检查条件变量时就是并发安全的。如果条件成立，则由调用方释放锁，并继续执行业务代码。

我们可以总结出正确使用 `sync.Cond` 的套路：

1. 要在调用 `Wait` 方法之前加锁，调用后释放锁，并且需要一个 `for` 循环来不断的检测条件变量是否满足。
2. 在业务代码中加锁来并发安全的修改条件变量。
3. 每次修改条件变量后，都要调用 `Signal/Broadcast` 方法来唤醒被 `Wait` 方法阻塞的 goroutine。

你一定要把这个套路当作 `sync.Cond` 的使用模板，印在你的脑海中。

那么接下来，我们以一个更加真实的案例，来体会 `sync.Cond` 的用法。

### 使用示例

我带你使用 `sync.Cond` 实现一个并发等待队列，以此来彻底掌握 `sync.Cond`。

这里声明了一个接口，来定义队列需要实现的几个方法：

```go
type Interface interface {
	Add(item any)                   // 元素入队
	Get() (item any, shutdown bool) // 元素出队
	Len() int                       // 获取队列长度
	ShutDown()                      // 关闭队列
	ShuttingDown() bool             // 队列是否已经关闭
}
```

既然是队列，那么就要有入队操作 `Add` 和出队操作 `Get`，`Len` 用来获取队列当前长度，调用 `ShutDown` 方法可以关闭队列，关闭后的队列无法再进行入队操作，`ShuttingDown` 方法则返回队列是否已经关闭。

接下来，我们就来实现这个接口。

首先，我们定义一个结构体 `Queue` 作为并发等待队列的实现：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/queue/queue.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/queue/queue.go)
>

```go
// Queue 并发等待队列
type Queue struct {
	// 条件变量
	cond *sync.Cond

	// 队列
	queue []any

	// 队列是否关闭的标识
	shuttingDown bool
}
```

`Queue`  有 3 个属性：

使用指针类型的 `sync.Cond` 作为 `Queue` 的第一个属性，因为 `sync.NewCond` 返回的就是指针类型。

使用一个支持任意类型的切片 `[]any` 作为队列，用于保存队列中每一项元素。

最后的 `bool` 类型属性 `shuttingDown` 用来标识队列是否已经被关闭，为 `true` 表示关闭。

我们再为并发等待队列定义一个构造函数 `New`：

```go
// New 创建一个并发等待队列
func New() *Queue {
	return &Queue{
		cond: sync.NewCond(&sync.Mutex{}),
	}
}
```

接着，实现入队方法 `Add`：

```go
// Add 元素入队，如果队列已经关闭，则直接返回，无法入队
func (q *Queue) Add(item any) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	if q.shuttingDown { // 如果队列已经关闭，则直接返回，不再入队
		return
	}

	q.queue = append(q.queue, item) // 入队
	q.cond.Signal()                 // 唤醒一个等待者，通知队列中有数据了
}
```

当我们将元素 `item` 加入到队列 `q.queue` 中后，就会调用 `q.cond.Signal()` 来唤醒一个等待者。所以 `q.queue` 就是我们的**条件**。注意，`Add` 方法内部的逻辑都进行了加锁操作。

然后，就应该要实现出队方法 `Get` 了：

```go
// Get 从队列中获取一个元素，如果队列为空则阻塞等待
// 第二个返回值标识队列是否已经关闭，已关闭则返回 true，无法获取到数据
func (q *Queue) Get() (item any, shutdown bool) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	for len(q.queue) == 0 && !q.shuttingDown {
		q.cond.Wait() // 如果队列为空且未关闭，阻塞等待队列中有数据时被唤醒
	}
	if len(q.queue) == 0 { // 如果此时队列为空，那么 q.shuttingDown 必然为 true，说明队列已经被关闭了
		return nil, true
	}

	// NOTE: 如果 queue 不为空，shuttingDown 可能为 true 也可能为 false，都继续往下执行
	// 即使标记队列已经被关闭了，也要清空 queue

	// 出队逻辑
	item = q.queue[0]
	q.queue[0] = nil // 主动清除引用，帮助 GC 回收
	q.queue = q.queue[1:]

	return item, false
}
```

`Get` 方法返回两个值，第一个值 `item` 是出队元素，第二个值 `shutdown` 标识队列是否已经关闭。

这里在是否调用 `q.cond.Wait()` 进行阻塞的判断条件是 `for len(q.queue) == 0 && !q.shuttingDown`，因为不止 `q.queue` 这一个判别条件，当队列被关闭，也不应该阻塞在这里，因为永远也不会获得元素。

这里有一个值得注意的点，就是出队时，我们将 `q.queue[0]` 置为了 `nil`，这样会解除切片底层数组对 `item` 的引用，让 GC 尽早对其进行回收，避免极端情况出现内存泄漏。

现在，我们已经实现了一个并发等待队列的核心逻辑，入队和出队。入队时会检测队列是否已经关闭，如果队列已经关闭，则直接返回，不再入队。出队时会检测队列是否为空，同时检测队列是否已经关闭，如果队列未关闭，且为空，则阻塞等待，直到有值或队列被关闭。

其他几个方法都比较简单，我一并列出来：

```go
// ShutDown 关闭队列
func (q *Queue) ShutDown() {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.shuttingDown = true // 标记队列关闭
	q.cond.Broadcast()
}

// ShuttingDown 队列是否关闭
func (q *Queue) ShuttingDown() bool {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	return q.shuttingDown // 返回队列是否关闭标识
}

// Len 获取队列长度
func (q *Queue) Len() int {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	return len(q.queue) // 返回队列当前长度
}
```

至此，一个并发等待队列就实现完成了。

我们可以为这个并发等待队列编写一个测试用例：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/queue/queue_test.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/queue/queue_test.go)
>

```go
func TestBasic(t *testing.T) {
	tests := []struct {
		queue *queue.Queue
	}{
		{
			queue: queue.New(),
		},
		{
			queue: queue.New(),
		},
	}
	for _, test := range tests {
		// If something is seriously wrong this test will never complete.

		// Start producers
		const producers = 50
		producerWG := sync.WaitGroup{}
		producerWG.Add(producers)
		for i := 0; i < producers; i++ {
			go func(i int) {
				defer producerWG.Done()
				for j := 0; j < 50; j++ {
					test.queue.Add(i)
					time.Sleep(time.Millisecond)
				}
			}(i)
		}

		// Start consumers
		const consumers = 10
		consumerWG := sync.WaitGroup{}
		consumerWG.Add(consumers)
		for i := 0; i < consumers; i++ {
			go func(i int) {
				defer consumerWG.Done()
				for {
					item, quit := test.queue.Get()
					if item == "added after shutdown!" {
						t.Errorf("Got an item added after shutdown.")
					}
					if quit {
						return
					}
					t.Logf("Worker %v: processing %v", i, item)
				}
			}(i)
		}

		producerWG.Wait()
		test.queue.ShutDown()
		test.queue.Add("added after shutdown!")
		consumerWG.Wait()
		if test.queue.Len() != 0 {
			t.Errorf("Expected the queue to be empty, had: %v items", test.queue.Len())
		}
	}
}
```

这里并发启动了 50 个生产者 goroutine，每个 goroutine 向队列中写入 50 个元素。启动了 10 个消费者 goroutine，来并发的消费队列。在生产者生产完成后，会调用 `test.queue.ShutDown()` 关闭队列，然后再次尝试向队列中添加一个元素，等待消费者消费完成，最终判断队列长队是否为 0。

执行这个测试用例，得到如下输出：

```go
$ go test -v -run=TestBasic
=== RUN   TestBasic
    queue_test.go:58: Worker 9: processing 2
    queue_test.go:58: Worker 9: processing 16
    ...
    queue_test.go:58: Worker 1: processing 36
    queue_test.go:58: Worker 5: processing 13
    queue_test.go:58: Worker 7: processing 24
--- PASS: TestBasic (0.26s)
PASS
ok      github.com/jianghushinian/blog-go-example/sync/cond/queue       0.679s
```

这个输出日志中省略了大量的 `t.Logf()` 打印，不过这足以演示队列的正确性。

更多测试用例，你可以在这里看到：[https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/queue/queue_test.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond/queue/queue_test.go)。

其实，我们实现的这个并发等待队列，正是 Kubernetes client-go 中非常关键的一个组件 [workqueue](https://github.com/kubernetes/kubernetes/blob/v1.30.0/staging/src/k8s.io/client-go/util/workqueue/queue.go) 的精简版。搞懂了这个并发等待队列的实现，再去看 `workqueue` 的源码就很容易上手了，祝你好运 :)。

### 总结
本文对 Go 中的 `sync.Cond` 并发原语进行了讲解，并带你看了其源码的实现，以及介绍了如何使用。

你一定要记得我们总结出来的 `sync.Cond` 使用套路，不要用错。我们最终实现的并发等待队列 `Queue` 是 Kubernetes client-go 中关键组件 `workqueue` 的微小实现，你也一定要掌握。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Go 并发控制：sync.WaitGroup 详解：[https://jianghushinian.cn/2024/12/23/sync-waitgroup/](https://jianghushinian.cn/2024/12/23/sync-waitgroup/)
+ Go 中空结构体惯用法，我帮你总结全了！：[https://jianghushinian.cn/2024/06/02/i-have-summarized-all-the-usages-of-empty-struct-in-go-for-you/#标识符](about:blank)
+ sync.Cond Documentation：[https://pkg.go.dev/sync@go1.23.0#Cond](https://pkg.go.dev/sync@go1.23.0#Cond)
+ sync.Cond 源码：[https://github.com/golang/go/blob/go1.23.0/src/sync/cond.go](https://github.com/golang/go/blob/go1.23.0/src/sync/cond.go)
+ Kubernetes client-go workqueue 源码：[https://github.com/kubernetes/kubernetes/blob/v1.30.0/staging/src/k8s.io/client-go/util/workqueue/queue.go](https://github.com/kubernetes/kubernetes/blob/v1.30.0/staging/src/k8s.io/client-go/util/workqueue/queue.go)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond](https://github.com/jianghushinian/blog-go-example/tree/main/sync/cond)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

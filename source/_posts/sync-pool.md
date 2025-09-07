---
title: 'Go 并发控制：sync.Pool 详解'
date: 2025-09-07 17:43:04
tags:
- Go
- 并发编程
categories:
- Go
---

`sync.Pool` 是 Go 并发原语中用于对象池化的工具，主要用于缓存和复用临时对象，以减少内存分配和垃圾回收的压力。

本文将带大家一起来深入探讨 `sync.Pool`，包括使用示例和源码解读，让你彻底理解 `sync.Pool` 的设计。

<!-- more -->

### 简介

`sync.Pool` 核心功能是能够缓存对象，避免其缓存的对象在一定时间内被垃圾回收掉。

因此 `sync.Pool` 的主要作用也就体现了出来：减少内存分配和回收压力。

如果一个对象被频繁的创建和删除，那么对内存分配和 GC 压力就会比较大，使用 `sync.Pool` 能够将对象缓存到池中，避免频繁创建和删除对象。

`sync.Pool` 是一个结构体，其全部公开属性如下：

```go
type Pool
    New func() any
    func (p *Pool) Get() any
    func (p *Pool) Put(x any)
```

每个属性含义如下：

+ `New` 字段：当池中没有可用对象时，调用 `New` 函数创建一个新对象。
+ `Get` 方法：从池中获取一个对象。如果池为空，则调用 `New` 创建新对象。
+ `Put` 方法：将对象放回池中，以便复用。

接下来我们一起来看下 `sync.Pool` 如何使用。

### 使用示例

`sync.Pool` 使用示例如下：

```go
package main

import (
	"bytes"
	"io"
	"os"
	"sync"
	"time"
)

var bufPool = sync.Pool{
	New: func() any {
		// The Pool's New function should generally only return pointer
		// types, since a pointer can be put into the return interface
		// value without an allocation:
		return new(bytes.Buffer)
	},
}

// timeNow is a fake version of time.Now for tests.
func timeNow() time.Time {
	return time.Unix(1136214245, 0)
}

func Log(w io.Writer, key, val string) {
	b := bufPool.Get().(*bytes.Buffer)
	b.Reset()
	// Replace this with time.Now() in a real logger.
	b.WriteString(timeNow().UTC().Format(time.RFC3339))
	b.WriteByte(' ')
	b.WriteString(key)
	b.WriteByte('=')
	b.WriteString(val)
	w.Write(b.Bytes())
	bufPool.Put(b)
}

func main() {
	Log(os.Stdout, "path", "/search?q=flowers")
}
```

这是 [sync.Pool](https://pkg.go.dev/sync@go1.25.0#Pool) 官方文档中示例代码。

首先，在第 11 行直接通过 `sync.Pool{}` 语法实例化一个 `Pool` 对象 `bufPool`，并且这里还初始化了 `New` 函数，其返回一个 `*bytes.Buffer` 对象。

接着，在第 25 行的 `Log` 函数内部使用了 `bufPool`，通过 `Get` 方法得到一个 `*bytes.Buffer` 类型的对象 `b`，然后向 `b` 中写入数据，最后别忘了使用 `Put` 方法将 `*bytes.Buffer` 对象“还回去”，将其缓存在池中，以便下次使用。

最后，在 `main` 函数中调用 `Log` 函数，并将结果写入标准输出。

执行示例代码，得到输出如下：

```bash
$ go run main.go
2006-01-02T15:04:05Z path=/search?q=flowers
```

根据以上使用示例，我们可以总结 `sync.Pool` 使用套路：

1. 实例化一个 `sync.Pool` 对象，并且赋值 `New` 属性，用户构造缓存对象。
2. 通过 `p.Get()` 取出对象 `obj` 使用。
    1. 如果池中有，就直接返回。
    2. 如果没有，调用 `New` 属性构造函数，构造一个新的对象并返回（如果没有 `New` 属性，则返回 `nil`）。
3. 对象使用完成后记得调用 `p.Put(obj)` 重新放入池中，以便下次使用。

由此可见，`sync.Pool` 适用于以下场景：

+ 频繁创建和销毁的对象：如临时缓冲区。
+ 减少内存分配：通过复用对象，减少 GC 压力。
+ 无状态对象：池中的对象不应包含与特定上下文相关的状态。

此外，在使用 `sync.Pool` 时有两点需要我们特别注意：

+ 对象重置：从池中获取的对象可能包含之前的状态，使用前需要重置。
+ 对象生命周期：池中的对象可能会被 GC 回收，因此不能依赖池中的对象长期存在。

`sync.Pool` 的设计中有一个比较有意思的点，一个对象被放入池中以后，如果没被使用，则连续两次 GC 后，这个对象一定会被释放。

那么，你是否好奇，`sync.Pool` 内部是如何实现这一机制的呢？咱们接着往下看，我们一起通过源码来揭开 `sync.Pool` 的神秘面纱。

### 实现原理

学习了 `sync.Pool` 如何使用，接下来我们一起通过阅读源码的方式来深入到 `sync.Pool` 的原理学习。

#### sync.Pool 结构体

`sync.Pool` 是一个结构体，其定义如下：

```go
type Pool struct {
	// 禁止复制
	noCopy noCopy

	// 空闲对象，poolLocal 指针类型
	local unsafe.Pointer
	// 数组大小
	localSize uintptr

	// 回收站
	victim unsafe.Pointer
	// 数组大小
	victimSize uintptr

	// New 是一个可选的函数，调用 Get 方法时，如果缓存池中没有可用对象，则调用此方法生成一个新的值并返回，否则返回 nil
	// 该函数不能在并发调用 Get 时被修改
	New func() any
}
```

其中 `New` 属性我们已经使用过了，调用 `Get` 方法时，如果缓存池中没有可用对象，则调用此方法生成一个新的值并返回。

`noCopy` 属性用来标记禁止复制，所以我们在拿到 `sync.Pool` 实例化对象后，记得一定不要让其产生复制操作。

`sync.Pool` 有两个核心字段分别是 `local` 和 `victim`，二者都是 `poolLocal` 指针类型，用来存储缓存对象。`local` 是当前 P 本地缓存的对象，而 `victim` 则可以理解为 Windows 操作系统的“回收站”。

Go 在触发垃圾回收时，`sync.Pool` 会做两件事：

1. 将所有缓存的 `victim` 中的对象移除。
2. 把所有缓存的 `local` 中对象移动到 `victim`。

进入到 `victim` 中的对象最终会有两种结果：

1. 当发生 GC 时，对象会被移除。
2. 如果还未发生 GC，而是优先调用了 `Get` 方法，那么这个对象就会被重新使用。

所以说，`victim` 就是 Windows 电脑中的“回收站”，我们在电脑中删除文件时，先到回收站，然后在回收站里可以彻底删除。

`poolLocal` 同样是一个结构体，其定义如下：

```go
type poolLocalInternal struct {
	// 私有对象
	private any
	// 共享队列，这是一个 lock-free 双向队列
	shared poolChain // Local P can pushHead/popHead; any P can popTail.
}

type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

`poolLocal` 中包含了 `poolLocalInternal` 和 `pad` 两个属性。

其中 `pad` 属性并不是用来存放数据的，而是用于将 `poolLocal` 结构体所占用的内存对齐到 128 的整数倍。这是为了解决伪共享（`false sharing`）问题，以此来独占 CPU 高速缓存的 CacheLine。

而 `poolLocalInternal` 结构体内部，才是用来存储缓存数据的。其中 `private` 是一个私有对象，用于记录当前 P 下缓存的对象，`shared` 是一个双向队列（一个 `lock-free` 的双向链表结构），用于记录多个 P 中共享的缓存对象，当前 P 能够进行 `pushHead/popHead` 操作，其他 P 能够进行 `popTail` 操作，从而在当前 P 中窃取缓存对象。

这里所说的 P 是指 Go GMP 模型中的处理器（P），之所以设计为当前 P 从队头进行读写，其他 P 从队尾进行获取操作，目的是在不加锁的情况下保证并发安全。

`sync.Pool` 为每个处理器（P）维护一个本地的 `poolLocal` 结构，其中包含一个 `shared` 队列。这个 `shared` 队列的类型是 `poolChain`，它是一个由多个 `poolDequeue` 节点组成的双向链表结构。每个 `poolDequeue` 都是一个固定大小的环形队列（r`ing buffer`），并且每个新节点的容量通常是前一个节点的两倍。

`poolDequeue` 被设计为一个单生产者（`single-producer`）/多消费者（`multi-consumer`） 的无锁队列（`lock-free`）：

+ 生产者：即当前 `P`，可以执行 `pushHead`（在头部添加）和 `popHead`（从头部弹出）操作。
+ 消费者：包括当前 `P`（也可以消费）和其他 `P`。其他 `P`只能执行 `popTail`（从尾部弹出）操作。

对于 `poolChain` 的介绍就到这里，不再继续深入，避免陷入其中，我们应该继续回到 `sync.Pool` 本身方法的学习。

#### Put 方法

`Put` 方法用于添加一个对象到池中，其实现如下：

```go
// Put 添加一个元素到池中
func (p *Pool) Put(x any) {
	if x == nil { // x 为 nil 直接返回
		return
	}

	// pin() 把当前 goroutine 固定在当前的 P 上
	// 同时返回 local 对象（*poolLocal）和当前 P id
	l, _ := p.pin()

	if l.private == nil {
		l.private = x // 如果 private 为 nil，则直接将 x 赋值给它
	} else {
		l.shared.pushHead(x) // 否则，将 x push 到共享队列队头
	}

	// 将当前 goroutine 从当前 P 上解除固定
	runtime_procUnpin()
}
```

可以发现，`Put` 方法实现逻辑相当简单。

其中 `p.pin()` 和 `runtime_procUnpin()` 是必须成对出现的调用，有点类似互斥锁的加锁/解锁操作，并且同样是用来解决并发问题的。不同的是，`pin` 操作更加轻量，`p.pin()` 能够将当前 goroutine 固定在当前的 P 上。因为在一个 P 上，同一时刻只会运行一个 goroutine，所以，接下来在当前 goroutine 中操作当前 P 上的任何对象都无需加锁，从而避免的并发问题。

调用 `p.pin()` 能够拿到存储在当前 P 中的 `*poolLocal` 对象和当前 P `ID`，有了 `*poolLocal` 对象，就可以判断 `l.private` 是否为空，如果值为 `nil`，那么直接将对象 `x` 赋值到 `private` 属性中缓存起来。否则，将对象 `x` 存储到共享队列 `l.shared` 中。

最后，记得调用 `runtime_procUnpin()` 解除 goroutine 和 P 的绑定。

#### Get 方法

`Get` 方法用于从池中获取一个对象，其实现如下：

```go
// Get 从 [Pool] 中选择一个任意项，将其从 Pool 中移除，然后返回给调用者。
// Get 可以选择忽略池并将其视为空。
// 调用者不应假设传递给 [Pool.Put] 的值与 Get 返回的值之间存在任何关系。
//
// 如果 Get 返回 nil 且 p.New 非零，则 Get 返回调用 p.New 的结果。
func (p *Pool) Get() any {
	// 把当前 goroutine 固定在当前的 P 上
	// 拿到 local 对象（*poolLocal，该 P 的本地池）和当前 P id
	l, pid := p.pin()

	// 获取当前 P 中的 private
	x := l.private
	l.private = nil
	if x == nil { // private 不存在
		// 尝试从当前 P 的共享队列中弹出空闲对象
		// 因为 shared 队列只有所属的 P 会操作头部（生产者），所以 popHead 操作也无需加锁
		x, _ = l.shared.popHead()
		if x == nil { // 触发慢路径
			// 当前 P 的本地池为空，则尝试从其他 P 窃取或从 victim 缓存获取
			x = p.getSlow(pid)
		}
	}

	// 解除 pin
	runtime_procUnpin()

	if x == nil && p.New != nil {
		x = p.New() // 如果所有缓存都未找到对象，且用户提供了 New 函数，则创建一个新对象
	}
	return x
}
```

与 `Put` 方法一样，`Get` 方法的逻辑也通过 `p.pin()` 和 `runtime_procUnpin()` 进行保护。

Get 方法在缓存中获取空闲对象的搜索路径如下：

1. 从 `l.private` 中获取对象。
2. 从本地共享队列 `l.shared` 中获取对象。
3. 慢路径（尝试从其他 P 窃取或从 `victim` 回收站中获取）。

慢路径源码实现如下：

```go
func (p *Pool) getSlow(pid int) any {
	// See the comment in pin regarding ordering of the loads.
	size := runtime_LoadAcquintptr(&p.localSize) // load-acquire
	locals := p.local                            // load-consume

	// 尝试从其他进 P 的共享队列中窃取一个元素
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size)) // 计算其他 P 的索引
		if x, _ := l.shared.popTail(); x != nil {    // 从尾部窃取
			return x
		}
	}

	// 如果窃取也失败了，则转而检查 victim 缓存
	size = atomic.LoadUintptr(&p.victimSize) // 获取 victim 缓存大小
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil { // 先检查 victim 的 private
		l.private = nil // 从 victim 中移除后再返回
		return x
	}
	for i := 0; i < int(size); i++ { // 再检查 victim 的 shared
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	atomic.StoreUintptr(&p.victimSize, 0) // 标记 victim 为空

	return nil
}

// 根据给定的索引 i，计算出指向 local 数组（[P]poolLocal）中第 i 个 poolLocal 元素的指针
func indexLocal(l unsafe.Pointer, i int) *poolLocal {
	lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
	return (*poolLocal)(lp)
}
```

可以发现，`getSlow` 方法内部会逐个计算其他 P 的索引，然后从对应 P 的共享队列尾部 `l.shared.popTail()` 窃取缓存对象。

如果遍历完所有 P 的共享缓存，都没能找到缓存对象，则继续检查 `victim` 中的缓存数据。

如果 `victim` 的 `private` 中有数据，则直接返回，否则继续检查 `victim` 的共享队列，如果 `victim` 的共享队列中没能找到数据，最终才会返回 `nil`。

现在 `sync.Pool` 实现缓存对象的主体逻辑已经串通了，但是还有一点没有串接起来，`victim` 是何时被赋值的？

#### pin 操作

要找到 `victim` 的赋值操作，还需要先理解 `pin` 方法的内部实现，其实现如下：

```go
// 将当前 goroutine 固定（pin）到其运行的 P（逻辑处理器）上，并返回该 P 对应的本地缓存池 (*poolLocal) 和 P 的 id
// 调用方必须在使用完成后调用 runtime_procUnpin() 取消固定
func (p *Pool) pin() (*poolLocal, int) {
	if p == nil { // 空指针检查
		panic("nil Pool")
	}

	// 固定 P，调用 runtime 函数，禁止当前 G 被抢占，并将其固定到当前 P，同时返回 P 的 id
	// 这是后续无锁操作的基础
	pid := runtime_procPin()
	// 原子加载本地池信息
	s := runtime_LoadAcquintptr(&p.localSize) // load-acquire
	l := p.local                              // load-consume
	if uintptr(pid) < s {                     // 快速路径（常见情况）
		// 如果当前 P 的 id 在 local 数组的有效大小范围内，则通过 indexLocal 函数计算地址，直接返回对应的 poolLocal 和 pid
		return indexLocal(l, pid), pid
	}

	// 慢路径（初始化或扩容）
	return p.pinSlow()
}

// pin 方法的“慢路径”（slow path）
// 负责在特定情况下初始化或重新分配 Pool 的本地存储数组 (local)，
// 并确保该 Pool 被注册到全局的 allPools 列表中以便垃圾回收 (GC) 时进行清理
func (p *Pool) pinSlow() (*poolLocal, int) {
	// Retry under the mutex.
	// Can not lock the mutex while pinned.
	runtime_procUnpin() // 解除当前 G 与 P 的绑定，为获取全局锁做准备

	allPoolsMu.Lock() // 加全局互斥锁，保护对 allPools 和 Pool 的 local 等字段的并发访问
	defer allPoolsMu.Unlock()

	pid := runtime_procPin() // 重新固定 G 到 P
	// poolCleanup won't be called while we are pinned.
	s := p.localSize
	l := p.local
	if uintptr(pid) < s { // 在锁保护下再次检查 local 数组是否已由其他 goroutine 初始化（双重检查锁定模式）
		return indexLocal(l, pid), pid
	}

	// 如果 Pool 尚未注册，则将其添加到 allPools 全局切片中，以便后续 GC 时能执行 poolCleanup 清理其缓存
	if p.local == nil {
		allPools = append(allPools, p)
	}

	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size) // 根据当前的 GOMAXPROCS（即 P 的数量）创建一个新的 poolLocal 数组
	// 记录初始化的 poolLocal 数组
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	runtime_StoreReluintptr(&p.localSize, uintptr(size))     // store-release
	return &local[pid], pid                                  // 返回新创建的、当前 P 对应的 *poolLocal 和 P 的 id
}

```

可以看到，`pin` 操作也有快慢路径之分，慢路径 `p.pinSlow()` 一般出现在初始化场景中。

这里，我们需要重点关注的是如下这段代码：

```go
if p.local == nil {
    allPools = append(allPools, p)
}
```

如果 `Pool` 尚未注册，即 `p.local == nil`，则将其添加到 `allPools` 全局切片变量中，以便后续 GC 时能执行 `poolCleanup` 操作清理其缓存。

那么这个 `allPools` 是干什么的？我们接着往下看与 GC 相关的代码。

#### GC 垃圾回收

`sync.Pool` 中与 GC 相关的代码实现如下：

```go
//go:linkname poolCleanup
func poolCleanup() {
	// 此函数在垃圾回收（GC）开始，程序暂停（STW）时被调用
	// 它自身一定不能分配内存，并且很可能不应调用任何运行时函数（runtime functions）

	// Drop victim caches from all pools.
	for _, p := range oldPools {
		p.victim = nil // 清空回收站
		p.victimSize = 0
	}

	// Move primary cache to victim cache.
	for _, p := range allPools {
		p.victim = p.local // 从主缓存移到回收站
		p.victimSize = p.localSize
		p.local = nil // 主缓存置空
		p.localSize = 0
	}

	oldPools, allPools = allPools, nil
}

var (
	// 保护 allPools 的互斥锁
	allPoolsMu Mutex

	// allPools 是拥有非空主缓存（non-empty primary caches）的 pool 的集合
	// 保证并发安全的机制有两种：1) 通过 allPoolsMu 互斥锁和 pinning（固定）机制；2) 通过垃圾回收时的程序暂停 STW（Stop-The-World）
	allPools []*Pool

	// oldPools 是可能拥有非空 victim 缓存（non-empty victim caches）的 pool 的集合
	// 保证并发安全机制为 STW（Stop-The-World）
	oldPools []*Pool
)

func init() {
	// 将 poolCleanup 注册到 runtime，确保每次 GC 开始时自动被调用
	runtime_registerPoolCleanup(poolCleanup)
}
```

这段代码要从下往上解读。

首先在 `init` 函数中 将 `poolCleanup` 注册到 `runtime`，这样 `poolCleanup` 函数会在每次 GC 开始时自动被调用。

`poolCleanup` 函数内部会操作 `allPools` 和 `oldPools` 两个全局变量，因为是在 GC STW 时执行，不会存在并发问题，所以无需加锁。

而 `poolCleanup` 函数内的源码实现，正是我们在前文中所讲的，Go 在触发垃圾回收时，`sync.Pool` 会做两件事：

1. 将所有缓存的 `victim` 中的对象移除。
2. 把所有缓存的 `local` 中对象移动到 `victim`。

现在 `victim` 的赋值操作也找到了。当然，`sync.Pool` 的核心源码也随之解读完了。

#### sync.Pool 执行流

通过以上源码的分析，你可能还有些发懵，没关系，这是正常现象，我也是阅读了好几遍 `sync.Pool` 源码，才搞清楚其逻辑的。

如果你坚持阅读到这里，那么恭喜你，离真正理解 `sync.Pool` 更近了一步。

我们可以通过几张流程图，再来梳理一下 `sync.Pool` 的执行流，以此来加深对 `sync.Pool` 源码的理解。

`Put` 操作执行流程如下：

![Put](Pool_Put.png)

Get 操作执行流程如下：

![Get](Pool_Get.png)


至此，`sync.Pool` 原理就解读完了。

### 总结

本文带大家一起学习了 Go 并发原语 `sync.Pool`，这是一个在并发场景下非常有效的解决对象复用的手段。

通过源码解读，我们知道 `sync.Pool` 默认缓存数据会存储在 `local` 中，在触发 GC 时则被移动到 `victim`,`victim` 就像一个回收站，其内部的数据要被重新利用，要么被彻底删除。

你有没有想过，`sync.Pool` 为什么要设计成调用两次 GC 才会回收对象呢？

其实这是为了防止 GC 引起的性能抖动。如果只调用一次 GC，就回收对象，则可能导致对象被频繁的创建和回收，并不能有效起到缓存的作用。那如果调用 3 次 GC 再回收行不行呢？理论上可以，但不建议这样做，其实这是一个内存和性能之间的取舍问题，如果缓存数据没有被使用，还长期存放在内存中，则势必会造成内存的浪费。两次 GC 才回收对象，应该是一个比较合理的经验值。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/sync/pool) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ sync.Pool Documentation：[https://pkg.go.dev/sync@go1.25.0#Pool](https://pkg.go.dev/sync@go1.25.0#Pool)
+ sync.Pool GitHub 源码：[https://github.com/golang/go/blob/go1.25.1/src/sync/pool.go](https://github.com/golang/go/blob/go1.25.1/src/sync/pool.go)
+ 10 | Pool：性能提升大杀器：[https://time.geekbang.org/column/article/301716](https://time.geekbang.org/column/article/301716)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/sync/pool](https://github.com/jianghushinian/blog-go-example/tree/main/sync/pool)
+ 本文永久地址：[https://jianghushinian.cn/2025/09/07/sync-pool/](https://jianghushinian.cn/2025/09/07/sync-pool/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

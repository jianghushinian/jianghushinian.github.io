---
title: 'Go 并发控制：sync.WaitGroup 详解'
date: 2024-12-23 23:01:47
tags:
- Go
- 并发编程
categories:
- Go
---

前段时间我在[《Go 并发控制：errgroup 详解》](https://jianghushinian.cn/2024/11/04/x-sync-errgroup/)一文中讲解了 `errgroup` 的用法和源码，通过源码我们知道 `errgroup` 内部是使用 `sync.WaitGroup` 实现的，那么本文就更进一步，来探索下 `sync.WaitGroup` 源码是如何实现的。

<!-- more -->

### 使用示例

`sync.WaitGroup` 可以用来**阻塞等待**一组并发任务（goroutine）的完成，使用示例如下：

```go
package main

import (
	"fmt"
	"net/http"
	"sync"
)

func main() {
	var urls = []string{
		"http://www.golang.org/",
		"http://www.google.com/",
		"http://www.somestupidname.com/", // 这是一个错误的 URL，会导致任务失败
	}

	var wg sync.WaitGroup
	errs := make([]error, len(urls)) // 使用 slice 收集错误

	for i, url := range urls {
		wg.Add(1)
		go func() {
			defer wg.Done()
			resp, err := http.Get(url)
			if err != nil {
				errs[i] = fmt.Errorf("failed to fetch %s: %v", url, err)
				return
			}
			defer resp.Body.Close()
			fmt.Printf("fetch url %s status %s\n", url, resp.Status)
		}()
	}

	wg.Wait()

	// 处理所有错误
	for i, err := range errs {
		if err != nil {
			fmt.Printf("fetch url %s error: %s\n", urls[i], err)
		}
	}
}
```

示例中，我们使用 `sync.WaitGroup` 来启动 3 个 goroutine 并发访问 3 个不同的 `URL`，并在成功时打印响应状态码，或失败时记录错误信息。

执行示例代码，得到如下输出：

```bash
$ go run waitgroup/main.go
fetch url http://www.google.com/ status 200 OK
fetch url http://www.golang.org/ status 200 OK
fetch url http://www.somestupidname.com/ error: failed to fetch http://www.somestupidname.com/: Get "http://www.somestupidname.com/": dial tcp: lookup www.somestupidname.com: no such host
```

我们得到了两个成功的响应，并记录了一条错误信息。

根据示例，我们可以抽象出 `sync.WaitGroup` 最典型的惯用法：

```go
var wg sync.WaitGroup

for ... {
	wg.Add(1)

	go func() {
		defer wg.Done()
		// do something
	}()
}

wg.Wait()
```

`sync.WaitGroup` 零值可用，它会在内部维护一个计数器，`wg.Add(1)`会将 `sync.WaitGroup` 计数器的值加 1，表示增加一个 goroutine 计数；`wg.Done()` 则将计数器的值减 1，表示一个 goroutine 任务已经完成；`wg.Wait()` 会阻塞调用者所在的 goroutine，直到计数器的值为 0。

### 源码解读

本文以 Go 1.23.0 版本[源码](https://github.com/golang/go/blob/go1.23.0/src/sync/waitgroup.go)为基础进行讲解。

#### WaitGroup 结构体

首先 `sync.WaitGroup` 定义如下：

> [https://github.com/golang/go/blob/go1.23.0/src/sync/waitgroup.go](https://github.com/golang/go/blob/go1.23.0/src/sync/waitgroup.go)
>

```go
// WaitGroup 结构体
type WaitGroup struct {
	noCopy noCopy // 避免复制

	state atomic.Uint64 // 高 32 位是计数器（counter）的值，低 32 位是等待者（waiter）的数量
	sema  uint32        // 信号量，用于 阻塞/唤醒 waiter
}
```

`sync.WaitGroup` 是一个结构体，所以这也是其零值可用的原因。

这个结构体包含 3 个字段：

+ `noCopy` 字段的类型也叫 `noCopy`，这个字段用于标识 `sync.WaitGroup` 结构体**不可被复制**，vet 工具能够识别它。这个字段的具体细节我们暂且不必深究，它不是 `sync.WaitGroup` 的核心功能，在文章最后再来解释它。
+ `state` 字段是一个原子类型 `atomic.Uint64`，所以对 `state` 字段的修改能够保证原子性。它比较有意思，`sync.WaitGroup` 结构体使用这一个字段来表示两个“变量”值，高 32 位是计数器（`counter`）的值，低 32 位是等待者（`waiter`）的数量。我们调用 `wg.Add(1)` 或 `wg.Done()` 时操作的就是计数器 `counter`；调用 `wg.Wait()` 时等待者 `waiter` 数量就会加 1。
+ `sema` 是一个信号量，用于阻塞/唤醒 `waiter`，即调用 `wg.Wait()` 时的阻塞和唤醒都依赖这个信号量。

`sync.WaitGroup` 结构体只有 3 个方法：`Add`、`Done` 和 `Wait`。

#### Done 方法

我们先来看 `Done` 方法的源码实现：

```go
// Done 将计数器（counter）值减 1
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

调用 `Done` 方法可以让 `counter` 值减 1。可以发现，调用 `wg.Done()` 方法实际上等价于调用 `wg.Add(-1)`，所以我们的重点还是要关注 `Add` 方法。

#### Add 方法

`Add` 方法源码实现如下：

```go
// Add 为计数器（counter）的值增加 delta（delta 可能为负数）
func (wg *WaitGroup) Add(delta int) {
	state := wg.state.Add(uint64(delta) << 32) // delta 左移 32 位后与 state 相加，即为 counter 值加上 delta
	v := int32(state >> 32)                    // state 右移 32 位得到 counter 的值
	w := uint32(state)                         // state 转成 uint32 拿到低 32 位的值，得到 waiter 的值

    // 如果 counter 值为负数，直接 panic
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
    
	// 并发调用 Wait 和 Add 会触发 panic
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

    // 条件成立说明 counter 值加上 delta 操作成功，返回
	if v > 0 || w == 0 {
		return
	}

	// 如果 counter 值为 0，并且还有被阻塞的 waiter，程序继续向下执行

	// 并发调用 Wait 和 Add 会触发 panic
	if wg.state.Load() != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

    // 目前 counter 值已经为 0，这里重置 waiter 数量为 0
	wg.state.Store(0)
	for ; w != 0; w-- { // 唤醒所有 waiter
		runtime_Semrelease(&wg.sema, false, 0)
	}
}
```

> NOTE:
>
> 为了方便你理解，我将 `Add` 方法源码中使用 `race` 包做竞态检查部分的代码去掉了，并适当的增加了空行。
>

`Add` 方法接收一个 `int` 类型的 `delta` 值，在方法第一行，先将这个值转换成 `uint64` 类型，然后通过移位操作，将其左移 32 位后与 `wg.state` 值相加，得到新的 `state`。我在前文说过，`wg.state` 字段的高 32 位表示 `counter` 值，所以这行代码的作用就是为 `counter` 值加上 `delta`。当 `delta` 值为正数，`counter` 值增加，当 `delta` 值为负数，`counter` 值减少。

接着，将 `state` 值右移 32 位，得到高 32 位的 `counter` 值 `v`；使用 `uint32(state)` 操作将 `uint64` 类型强转成 `uint32`，舍弃高 32 位，得到低 32 位的 `waiter` 值 `w`。注意，这里拿到的 `v` 和 `w` 是与 `delta` 计算后的最新值。

接下来会做两个校验，先对 `counter` 进行判断，如果 `v` 的值为负数，会触发 `panic`，所以我们在使用时要小心 `counter` 不能为负；然后又对并发调用 `Wait` 和 `Add` 方法的场景做了校验，如果并发调用二者，同样会触发 `panic`。

关于判断是否并发调用 `Wait` 和 `Add` 方法的场景，我在详细解释下：

+ `w != 0` 表明有等待者 `waiter` 存在，即已经有 goroutine 调用了 `wg.Wait()` 方法，正在阻塞等待，还未返回。
+ `delta > 0` 表明这次调用 `Add` 方法是要增加计数器 `counter` 的值，这也说明肯定不是通过调用 `wg.Done()` 方法触发的。
+ `v == int32(delta)` 表明在调用 `Add` 方法之前，`counter` 的值为 0。因为 `v` 是计算后的 `counter` 值，它等于 `delta`，就说明在计算之前 `counter` 的值为 0。

如果这三个条件同时满足，即 `w != 0 && delta > 0 && v == int32(delta)` 为 `true`，就说明我们在调用 `wg.Wait()` 方法以后，还未等到唤醒它，就马上又调用了 `wg.Add(delta)` 方法，此时就会触发 `panic`。所以，我们在使用时要记住，一定要在调用 `wg.Wait()` 之前调用 `wg.Add(delta)`。

做完了校验以后，就到了 `Add` 方法的第一个出口，如果 `v > 0` 说明我们正常调用了 `Add` 方法或 `Done` 方法，计数器 `counter` 此时还未清零，那么无需唤醒 `wg.Wait()` 的阻塞等待，直接返回即可；或者如果 `w == 0` 说明当前没有正在等待的 `waiter`，即还未调用 `wg.Wait()`，那么也可以直接返回。

那么现在，`Add` 方法还能继续往下执行的条件是：`v == 0 && w > 0`，即 `counter` 值为 0，并且还有被阻塞的 `waiter`。

既然计数器 `counter` 值已经为 0，那么就可以唤醒所有被阻塞的 `wg.Wait()` 调用了，这也是接下来的程序逻辑。

不过，这里会再次对并发调用 `Wait` 和 `Add` 方法的场景进行校验。如果此时从 `wg.state` 字段获取到的最新值与变量 `state` 值不一致，即 `wg.state.Load() != state` 为 `true`，则会触发 `panic`。所以，当 `counter` 值变为 0，程序即将唤醒被阻塞的 `waiter` 之前这一小段时间，不要并发的调用 `wg.Add(delta)` 来改变计数器的值。

最后，通过 `wg.state.Store(0)` 将 `waiter` 的值置为 0（因为此时 `counter` 值已经是 0 了，所以这个操作的目的是将 `waiter` 值置 0），并使用 `runtime_Semrelease` 来唤醒所有被阻塞的 `waiter`。

> NOTE:
>
> 关于 `runtime_Semrelease` 以及下文将要介绍的 `runtime_Semacquire` 方法则不必深究，这是 Go 语言底层 `runtime` 为我们实现的用于唤醒或阻塞当前 goroutine 的函数。
>

至此，`Add` 方法就分析完成了。

我们现在可以总结下 `Add` 方法的作用：`Add` 为计数器 `counter` 的值增加 `delta`（`delta` 可能为负数），如果计算结果 `counter` 为负数，则触发 `panic`；如果 `counter` 为正数，则正常返回；如果 `counter` 为 0，则唤醒所有被阻塞的 `waiter`。

所以 `Add` 方法主要用来管理计数器 `counter`，并在 `counter` 为 0 时，唤醒 `waiter`。

#### Wait 方法

现在，我们再来看下 `sync.WaitGroup` 结构体最后一个方法 `Wait` 的源码实现：

```go
// Wait 阻塞调用者当前的 goroutine（waiter），直到计数器（counter）值为 0
func (wg *WaitGroup) Wait() {
	for { // 开启无限循环保证 CAS 操作成功
		state := wg.state.Load()
		v := int32(state >> 32) // 拿到 counter 值
		// w := uint32(state)   // 拿到 waiter 值

        if v == 0 { // 如果 counter 值已经为 0，直接返回
			return
		}

        // 使用 CAS 操作增加 waiter 的数量
		if wg.state.CompareAndSwap(state, state+1) {
			runtime_Semacquire(&wg.sema) // 阻塞当前 waiter 所在的 goroutine，等待被唤醒

            // 并发调用 Wait 和 Add 会触发 panic
            if wg.state.Load() != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}

			return // 如果 state 值为 0，说明 waiter 所等待的任务全部完成，成功返回
		}
	}
}
```

> NOTE:
>
> 为了方便你理解，我同样将 `Wait` 方法源码中使用 `race` 包做竞态检查部分的代码去掉了，并适当的增加了空行。
>

首先，这里开启了一个无限 `for` 循环，这是为了重试下面的 CAS 操作，保证其执行成功。

`Wait` 方法同样使用移位操作拿到到高 32 位的 `counter` 值 `v` 和低 32 位的 `waiter` 值 `w`。因为 `w` 只会在竞态检查的代码中被用到，所以被我手动注释掉了。

接下来判断计数器 `counter` 的值是否为 0，如果 `v` 已经为 0，那么无需阻塞 `waiter`，直接返回即可。否则，需要对 `waiter` 的值进行加 1 操作，这里使用 CAS 操作（即 Compare And Swap）来完成。

所谓的 CAS 操作，就是先 Compare 再 Swap。当我们调用 `wg.state.CompareAndSwap(state, state+1)` 时，`CompareAndSwap` 方法会先判断 `wg.state` 值是否等于传进来的第一个参数 `state`，如果相等，则将其替换为第二个参数 `state+1` 的值，并返回 `true`；如果 `wg.state` 值与 `state` 不相等，则不会修改 `wg.state`，并返回 `false`。这样，就保证了对 `wg.state` 的修改是**原子性**的。

在并发场景中，CAS 操作可能失败，返回 `false`，所以需要结合最外层的 `for` 无限循环，来保证 CAS 操作成功。

一旦 CAS 操作成功，即 `waiter` 的数量加 1，就会使用 `runtime_Semacquire` 来阻塞当前 `waiter` 所在的 `goroutine`，等待被唤醒。而唤醒时机，就是在 `Add` 方法的最后对 `runtime_Semrelease(&wg.sema, false, 0)` 的调用。

当 `waiter` 被唤醒后，会对并发调用 `Wait` 和 `Add` 方法的场景进行校验。如果 `wg.state.Load() != 0` 为 `true`，则会触发 `panic`。因为 `Add` 方法在调用 `runtime_Semrelease` 唤醒所有 `waiter` 之前，已经通过 `wg.state.Store(0)` 将 `waiter` 的值置为 0 了，所以在不出现并发调用的情况下，`wg.state.Load()` 的值必然为 0。

而如果没有出现并发调用 `Wait` 和 `Add` 方法，则说明  `waiter` 所等待的任务全部完成，正常返回即可。

至此，`sync.WaitGroup` 结构体最后一个方法`Wait` 就分析完成了。

根据源码，我们能够分析出：`Wait` 方法主要用来管理 `waiter`，它会阻塞所有 `waiter`，并等待被 `Add` 唤醒。

#### noCopy 结构体

现在 `sync.WaitGroup` 结构体的核心功能就全部讲解完成了，是时候介绍下 `noCopy` 了。

`noCopy` 实际上也是一个结构体，其定义如下：

```go
// noCopy may be added to structs which must not be copied
// after the first use.
//
// See https://golang.org/issues/8005#issuecomment-190753527
// for details.
//
// Note that it must not be embedded, due to the Lock and Unlock methods.
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

`noCopy` 非常简单，就是一个空结构体实现了 `Locker` 接口。`noCopy` 结构体的唯一功能，就是用于辅助 vet 工具检查用的，vet 工具遇到它就会知道，这个结构体是不能被复制的，仅此而已。

此外，我在[《Go 中空结构体惯用法，我帮你总结全了！》](https://jianghushinian.cn/2024/06/02/i-have-summarized-all-the-usages-of-empty-struct-in-go-for-you)一文中 [#标识符](https://jianghushinian.cn/2024/06/02/i-have-summarized-all-the-usages-of-empty-struct-in-go-for-you/#%E6%A0%87%E8%AF%86%E7%AC%A6) 小节介绍了如何将它用在我们自定义的结构体中，感兴趣的读者可以点击链接跳转过去学习。

好了，`sync.WaitGroup` 源码解析就讲解到这里。

### 总结

你一定要记住 `sync.WaitGroup` 的惯用法，首先，它无需初始化，零值可用；其次，它会在内部维护一个计数器 `counter`，通过 `wg.Add(delta)` 或`wg.Done()` 来操作计数器的值；它还会维护一个等待者数量 `waiter`，调用 `wg.Wait()` 会阻塞 `waiter` 所在的 goroutine；当计数器 `counter` 的值为 0，所有 `waiter` 都会被唤醒。

还要注意不要并发调用 `Add` 和 `Wait` 方法，也不要让计数器 `counter` 的值为负数，不然会触发 `panic`。

虽然 `sync.WaitGroup` 的源码很少，可却因为里面使用了移位操作和一些边界条件的检查，使其不太容易理解。为此，我专门画了一副 `sync.WaitGroup` 三大方法的执行流程图，来助你分析 `sync.WaitGroup` 各个方法的执行流程和关联关系。

流程图如下：

![](sync-waitgroup.png)

`Done` 方法没什么好解释的，等价于 `Add(-1)`。

`Add` 方法在第一次出现 `return` 之前的代码（即 `检查 counter 大于 0，或 waiter 等于 0`），其实可以看作是增加计数器的功能，即 `delta` 值大于 0 的情况；而接下来的代码，则可以看作是调用 `Done` 方法，减少计数器的功能，即 `delta` 值小于 0 的情况。当计数器的值为 0，就会唤醒所有 `waiter`。

`Wait` 方法则用来管理 `waiter`，并阻塞 `waiter`，等待被 `Add` 方法唤醒。

如果你对上面的源码分析理解还觉得有点不够透彻，可以对照这幅图，多梳理几遍。看懂了这幅图，那么你就完全掌握了 `sync.WaitGroup`。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/sync/waitgroup) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Go 并发控制：errgroup 详解：[https://jianghushinian.cn/2024/11/04/x-sync-errgroup/](https://jianghushinian.cn/2024/11/04/x-sync-errgroup/)
+ Go 中空结构体惯用法，我帮你总结全了！：[https://jianghushinian.cn/2024/06/02/i-have-summarized-all-the-usages-of-empty-struct-in-go-for-you/#标识符](about:blank)
+ sync.WaitGroup Documentation：[https://pkg.go.dev/sync@go1.23.0#WaitGroup](https://pkg.go.dev/sync@go1.23.0#WaitGroup)
+ sync.WaitGroup 源码：[https://github.com/golang/go/blob/go1.23.0/src/sync/waitgroup.go](https://github.com/golang/go/blob/go1.23.0/src/sync/waitgroup.go)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/sync/waitgroup](https://github.com/jianghushinian/blog-go-example/tree/main/sync/waitgroup)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

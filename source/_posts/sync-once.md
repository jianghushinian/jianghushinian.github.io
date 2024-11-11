---
title: 'Go 并发控制：sync.Once 详解'
date: 2024-11-11 14:31:36
tags:
- Go
categories:
- Go
---

在 Go 语言的并发编程中，常常会遇到需要确保某个操作仅执行一次的场景。`sync.Once` 是 Go 标准库中的一个简单而强大的工具，专门用于解决这种需求。本文将深入解析 `sync.Once` 的使用方法和原理，帮助你更好地理解 `sync.Once` 在并发控制中的用法。

<!-- more -->

### sync.Once

`sync.Once` 是 Go 语言 `sync` 包中的一种同步原语。它可以确保一个操作（通常是一个函数）在程序的生命周期中**只被执行一次**，不论有多少 goroutine 同时调用该操作，这就保证了并发安全。

根据 `sync.Once` 的特点，很容易想到它的几种常见使用场景：

+ **单例模式**：确保某个对象或配置仅初始化一次，例如使用单例模式初始化数据库连接池、配置文件加载等。
+ **懒加载**：在需要时才加载某些资源，且保证它们只会加载一次。
+ **并发安全的初始化**：当初始化过程涉及多个 goroutine 时，使用 `sync.Once` 保证初始化函数不会被重复调用。

#### 快速上手

`sync.Once` 用法非常简单，示例如下：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			once.Do(onceBody)
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
}
```

首先使用 `var once sync.Once` 声明了一个 `sync.Once` 类型的变量 `once`，但不必显式初始化，这也是 `sync` 下很多包的惯用法。

然后定义了一个函数 `onceBody`，接着启动 10 个 goroutine 并发调用 `once.Do(onceBody)`，最终等待所有 goroutine 执行结束并退出。

执行示例代码，得到如下输出：

```bash
$ go run once/main.go 
Only once
```

和预期一样，`once.Do` 能够保证传递给它的函数 `onceBody` 只被执行一次。

其实就算不启用多个 goroutine，直接在主 goroutine 中调用多次 `once.Do(onceBody)`，也能保证只执行一次：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}
	for i := 0; i < 10; i++ {
		once.Do(onceBody)
	}
}
```

执行示例代码，得到如下输出：

```bash
$ go run once/main.go 
Only once
```

执行结果也是一样的，仅会执行一次。

至此，我们就学会了如何使用 `sync.Onec`。以上就是 `sync.Onec` 提供的全部 API 了，没错它仅对外暴露了一个 `Do` 方法。

现在，我想你应该知道如何使用 `sync.Onec` 来实现单例模式了，我在另一篇文章[《Go 常见设计模式之单例模式》](https://jianghushinian.cn/2022/03/04/Golang-%E5%B8%B8%E8%A7%81%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%E4%B9%8B%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F/)中也有讲解。

#### 详细介绍

我们快速上手了 `sync.Onec` 的使用方法，下面我们来看下 Go 官方是如何介绍 `sync.Onec` 的。

Go [官方文档](https://pkg.go.dev/sync@go1.23.0#Once) 对 `sync.Once` 的介绍只有简单三句话：

> <font style="color:rgb(32, 34, 36);">Once is an object that will perform exactly one action.</font>
>
> <font style="color:rgb(32, 34, 36);">A Once must not be copied after first use.</font>
>
> <font style="color:rgb(32, 34, 36);">In the terminology of </font>[<font style="color:rgb(32, 34, 36);">the Go memory model</font>](https://go.dev/ref/mem)<font style="color:rgb(32, 34, 36);">, the return from f “synchronizes before” the return from any call of once.Do(f).</font>
>

<font style="color:rgb(32, 34, 36);">意思是说：</font>

> Once 是一个对象，它会确保某个操作**只执行一次**。
>
> 在首次使用后，Once 对象**不能被复制**。
>
> 根据 Go 内存模型的术语，f 函数的返回 "<font style="color:rgb(32, 34, 36);">synchronizes before</font>" 于 once.Do(f) 的任何调用返回。
>

首先第一句话中所说的**只执行一次**的特性我们已经见识过了。

对于第二句话中的 Once 对象**不能被复制**，其实 `sync` 中很多对象都有这个特性，在我们稍后阅读源码时会有体现。

而第三句话不太好理解，实际上它想表达的是，在使用 `sync.Once` 的 `Do` 方法执行 `f` 函数后，`f` 的结果会对所有调用 `once.Do(f)` 的其他 goroutine 可见。这种“先行发生”（synchronizes before）的保证意味着，`f` 的执行结果会在所有调用 `once.Do(f)` 的 goroutine 中同步，因此所有 goroutine 都能获得一致的结果。

具体来说：

+ 当 `f` 函数在一个 goroutine 中被 `once.Do(f)` 首次调用时，`f` 会执行，并保证它的**效果**在内存中对其他 goroutine 可见。
+ 之后的所有 `once.Do(f)` 调用都不会重新执行 `f`，但它们会“同步” `f` 的结果，确保 `f` 的结果已经生效，并对调用它们的 goroutine 可见。

这样，`sync.Once` 可以在多 goroutine 场景中安全地执行初始化等需要确保一次性操作的函数，而无需担心数据不一致的问题。

此外我们还需要注意一点，`once.Do(f)` 接收的函数 `f` 是没有返回值，所以所说 `f` 函数的执行的**效果**是指它执行的**副作用**。

如下示例就是利用 `f` 函数执行的副作用来修改变量 `i` 的值：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	var i = 10
	onceBody := func() {
		i *= 2
	}
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			once.Do(onceBody)
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
	fmt.Println("i", i)
}
```

执行示例代码，得到如下输出：

```bash
$ go run once/main.go
i 20
```

`f` 函数对变量 `i` 的值影响仅有一次。

#### 源码解读

接下来我们再来看下 `sync.Onec` 源码，学习下它是如何实现的：

> [https://github.com/golang/go/blob/go1.23.0/src/sync/once.go](https://github.com/golang/go/blob/go1.23.0/src/sync/once.go)
>

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

package sync

import (
	"sync/atomic"
)

// Once is an object that will perform exactly one action.
//
// A Once must not be copied after first use.
//
// In the terminology of [the Go memory model],
// the return from f “synchronizes before”
// the return from any call of once.Do(f).
//
// [the Go memory model]: https://go.dev/ref/mem
type Once struct {
	// done indicates whether the action has been performed.
	// It is first in the struct because it is used in the hot path.
	// The hot path is inlined at every call site.
	// Placing done first allows more compact instructions on some architectures (amd64/386),
	// and fewer instructions (to calculate offset) on other architectures.
	done atomic.Uint32
	m    Mutex
}

// Do calls the function f if and only if Do is being called for the
// first time for this instance of [Once]. In other words, given
//
//	var once Once
//
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
//
//	config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//
	//	if o.done.CompareAndSwap(0, 1) {
	//		f()
	//	}
	//
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the o.done.Store must be delayed until after f returns.

	if o.done.Load() == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done.Load() == 0 {
		defer o.done.Store(1)
		f()
	}
}
```

嗯，你没看错 `sync.Onec` 的源码竟然如此简单，算上全部的注释和空行，也才仅有 78 行代码，而注释占了一大半行数。

首先来看 `Once` 结构体的定义：

```go
type Once struct {
	// done 指示操作是否已执行。
	// 它在结构体中位于首位（即第一个字段），因为它用于 hot path。
	// hot path 在每个调用点都进行了内联。
	// 将 done 放在首位在某些架构（如 amd64 和 386）上允许生成更紧凑的指令，
	// 而在其他架构上减少了指令数量（用于计算偏移量）。
	done atomic.Uint32
	m    Mutex
}
```

`Once` 结构体有两个属性，`done` 属性上的注释告诉我们它是有意被放在结构体第一个字段的，在某些架构能够减少 CPU 执行的指令数，以优化性能，作为 `hot path`。

关于为什么放在结构体第一个字段就能优化性能，简单一句话来解释就是，第一个字段与结构体本身的指针地址是相同的，访问 `Once` 结构体无需指针偏移操作，就可以直接操作 `done` 属性。`hot path` 更多解释的细节，我在另一篇文章[《Go 语言中的结构体内存对齐你了解吗？》](https://jianghushinian.cn/2024/07/07/do-you-understand-the-memory-alignment-of-structs-in-the-go/#hot-path)中有所讲解，你可以跳转过去查看。

另外 `done` 属性是 `atomic.Uint32` 类型，我们顺便来看下 `atomic.Uint32` 是如何定义的：

```go
// A Uint32 is an atomic uint32. The zero value is zero.
type Uint32 struct {
	_ noCopy
	v uint32
}

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

这里有一个特殊字段 `_ noCopy`，标识这个结构体不可复制，所以这也就是为什么前文中提到 Once 对象**不能被复制**的原因了。

使用 `noCopy` 字段来标识结构体不可复制，是 Go 语言中的惯用法，我在另一篇文章[《Go 中空结构体惯用法，我帮你总结全了！》](https://jianghushinian.cn/2024/06/02/i-have-summarized-all-the-usages-of-empty-struct-in-go-for-you/#%E6%A0%87%E8%AF%86%E7%AC%A6)中有讲解。

`Once` 结构体的 `n` 属性没什么好说的，就是一个互斥锁 `Mutex`。

接下来看` Do` 方法的实现：

```go
func (o *Once) Do(f func()) {
	// 注意：以下是一个错误的 Do 实现：
	//
	//	if o.done.CompareAndSwap(0, 1) {
	//		f()
	//	}
	//
	// Do 保证当它返回时，f 已完成。
	// 上述实现不满足该保证：
    // 给定有两个同时调用的情况，谁取得了 CompareAndSwap 就会执行 f，
    // 而第二个调用会立即返回，而不会等待第一个调用的 f 完成。
	// 这就是为什么慢路径需要退回到使用互斥锁 Mutex，
	// 并且为什么 o.done.Store 必须延迟到 f 返回之后。
	
	if o.done.Load() == 0 {
		// 慢路径（slow-path）分离，以允许对快路径（fast-path）进行内联。
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done.Load() == 0 {
		defer o.done.Store(1)
		f()
	}
}
```

可以看到，`Do` 方法内部还通过注释贴心的解释了为什么不使用 `o.done.CompareAndSwap(0, 1)` 的实现，而是使用 `o.done.Load()` + `Mutex` 的实现。

`Do` 方法处理两种 `case`，先是 `if o.done.Load() == 0` 的判断，这是一个 `fast-path`，如果成立，则调用 `o.doSlow(f)` 进入 `slow-path`，否则 `fast-path` 执行结束直接返回了。

这里简单解释下 `fast-path` 和 `slow-path`：

+ **`fast-path`**：一段针对常见操作或最佳情况进行优化的代码路径。在这条路径上，通常执行步骤最少、效率最高。所以 `fast path` 通常在设计上避免了昂贵的操作（如**加锁**、IO 操作等）以提高性能。
+ **`slow-path`**：用于处理较为罕见或复杂的情况，通常执行步骤较多、性能较低。这类路径通常在少数情况下才会被执行，比如当代码需要处理边缘情况或复杂的操作时。

所以说，`Do` 方法绝大多数情况下都会通过 `fast-path` 直接返回，只有第一次调用才会进入 `o.doSlow(f)` 逻辑。

在 `doSlow` 方法内部，先加锁，然后再一次检查了 `if o.done.Load() == 0`。很明显这是一个 `Double-Check Locking`，保证极端情况下的并发安全。我在[《Go 常见设计模式之单例模式》](https://jianghushinian.cn/2022/03/04/Golang-%E5%B8%B8%E8%A7%81%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%E4%B9%8B%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F/#%E5%8F%8C%E9%87%8D%E9%94%81%E5%AE%9A)中有讲解如何使用 `Double-Check Locking` 来实现单例模式，感兴趣的读者可以跳转过去查看。

现在，`sync.Once` 的源码就都解读完成了。

当然，细心的读者可能注意到，注释中其实写了为什么要将 `slow-path` 分离出来，单独定义一个函数，目的是为了对 `fast-path` 进行内联优化。

将 `slow-path` 逻辑放在单独的 `doSlow` 函数中可以使 `Do` 方法的快路径更简洁，这样还有助于 Go 编译器对 `fast-path` 进行**内联优化**（即直接嵌入到调用处），从而减少函数调用的开销，提高性能。

我们可以来验证一下内联是否生效，示例代码如下：

```go
package once

import "sync"

func main() {
	var once sync.Once
	once.Do(func() {
		println("Only once")
	})
}
```

执行 `go build` 时传入 `-gcflags='-m'` 构建参数可以查看内联情况：

```bash
$ go build -gcflags='-m' inlining/once/main.go  
# command-line-arguments
inlining/once/main.go:7:10: can inline main.func1
inlining/once/main.go:7:9: inlining call to sync.(*Once).Do
inlining/once/main.go:7:9: inlining call to atomic.(*Uint32).Load
inlining/once/main.go:6:6: moved to heap: once
inlining/once/main.go:7:10: func literal does not escape
```

打印日志中出现 `inlining` 关键字表示 `main` 中调用 `sync.Once.Do` 方法时确实存在内联优化。

作为对比，我们再来实现一个没有将 `slow-path` 分离出来的 `Once` 版本：

```go
package sync

import (
	"sync"
	"sync/atomic"
)

type Once struct {
	done atomic.Uint32
	m    sync.Mutex
}

func (o *Once) Do(f func()) {
	if o.done.Load() == 0 {
		o.m.Lock()
		defer o.m.Unlock()
		if o.done.Load() == 0 {
			defer o.done.Store(1)
			f()
		}
	}
}
```

使用示例如下：

```go
package once

import "github.com/jianghushinian/blog-go-example/sync/once/inlining/myonce/sync"

func main() {
	var once sync.Once
	once.Do(func() {
		println("Only once")
	})
}
```

执行 `go build` 查看内联情况：

```bash
$ go build -gcflags='-m' inlining/myonce/main.go
# command-line-arguments
inlining/myonce/main.go:5:6: can inline main
inlining/myonce/main.go:7:10: can inline main.func1
inlining/myonce/main.go:6:6: moved to heap: once
inlining/myonce/main.go:7:10: func literal does not escape
```

这一次编译器确实没有进行内联优化，可见 `doSlow` 函数的封装还是起了作用的。

`sync.Once` 就这么多功能，不过在 [Go 1.21](https://go.dev/doc/go1.21#syncpkgsync) 中 Go 官方又增加了三个 `sync.Once` 相关函数：`OnceFunc`、`OnceValue` 和 `OnceValues`，来增强 `sync.Once` 功能，接下来我们就依次介绍下。

### sync.OnceFunc

#### 源码解读

首先我们来看一下 `sync.OnceFunc` 源码实现：

> [https://github.com/golang/go/blob/go1.23.0/src/sync/oncefunc.go#L11](https://github.com/golang/go/blob/go1.23.0/src/sync/oncefunc.go#L11)
>

```go
// OnceFunc returns a function that invokes f only once. The returned function
// may be called concurrently.
//
// If f panics, the returned function will panic with the same value on every call.
func OnceFunc(f func()) func() {
	var (
		once  Once
		valid bool
		p     any
	)
	// Construct the inner closure just once to reduce costs on the fast path.
	g := func() {
		defer func() {
			p = recover()
			if !valid {
				// Re-panic immediately so on the first call the user gets a
				// complete stack trace into f.
				panic(p)
			}
		}()
		f()
		f = nil      // Do not keep f alive after invoking it.
		valid = true // Set only if f does not panic.
	}
	return func() {
		once.Do(g)
		if !valid {
			panic(p)
		}
	}
}
```

根据 `OnceFunc` 上方的代码注释可知：

+ `OnceFunc` 返回一个仅调用 `f` 一次的函数。这个返回的函数可以并发调用。
+ 如果函数 `f` 执行时出现 `panic`，则返回的函数将在每次调用时会产生同样的 `panic` 值。

可以发现，其实 `OnceFunc` 函数就是对 `once.Do` 的封装，不过显然它考虑了更多情况，使用 `defer` + `recover` 对 `panic` 进行捕获。用变量 `p` 暂存了 `panic` 信息，并且当多次调用 `OnceFunc` 返回函数时，都会重新 `panic`。

> NOTE:
>
> 如果你不熟悉 `defer`、`panic` 和 `recover`，我在[《Go 错误处理指北：Defer、Panic、Recover 三剑客》](https://jianghushinian.cn/2024/10/13/go-error-guidelines-defer-panic-recover/)一文中对这三者进行了详细讲解，供你参考。
>

#### 使用示例

`sync.OnceFunc` 使用示例如下：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	onceBody := sync.OnceFunc(func() {
		fmt.Println("Only once")
	})
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			onceBody()
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
}
```

执行示例代码，得到如下输出：

```bash
$ go run oncefunc/main.go                       
Only once
```

如果发生 `panic` 会怎样呢，我们可以尝试一下：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	onceBody := sync.OnceFunc(func() {
		panic("Only once")
	})

	for i := 0; i < 5; i++ {
		func() {
			defer func() {
				r := recover()
				fmt.Println("recover", r)
			}()
			onceBody()
		}()
	}
}
```

执行示例代码，得到如下输出：

```bash
$ go run oncefunc/panic/main.go
recover Only once
recover Only once
recover Only once
recover Only once
recover Only once
```

作为对比，如果 `sync.Once.Do` 遇到 `panic` 又会怎样呢？示例如下：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	onceBody := func() {
		panic("Only once")
	}

	for i := 0; i < 5; i++ {
		func() {
			defer func() {
				r := recover()
				fmt.Println("recover", r)
			}()
			once.Do(onceBody)
		}()
	}
}
```

执行示例代码，得到如下输出：

```bash
$ go run once/panic/main.go
recover Only once
recover <nil>
recover <nil>
recover <nil>
recover <nil>
```

由此可见，可以认为 `sync.OnceFunc` 是比 `sync.Once.Do` 更好用的接口，它帮我们考虑了函数 `f` 发生 `panic` 情况，所以可以考虑优先使用这个实现。

### sync.OnceValue

#### 源码解读

我们再来看下 `sync.OnceValue` 源码实现：

> [https://github.com/golang/go/blob/go1.23.0/src/sync/oncefunc.go#L43](https://github.com/golang/go/blob/go1.23.0/src/sync/oncefunc.go#L43)
>

```go
// OnceValue returns a function that invokes f only once and returns the value
// returned by f. The returned function may be called concurrently.
//
// If f panics, the returned function will panic with the same value on every call.
func OnceValue[T any](f func() T) func() T {
	var (
		once   Once
		valid  bool
		p      any
		result T
	)
	g := func() {
		defer func() {
			p = recover()
			if !valid {
				panic(p)
			}
		}()
		result = f()
		f = nil
		valid = true
	}
	return func() T {
		once.Do(g)
		if !valid {
			panic(p)
		}
		return result
	}
}
```

可以看到，与 `OnceFunc` 不同的是 `OnceValue` 使用了泛型，`OnceValue` 接收的函数 `f` 是带有返回值的，并且它返回的函数也带有返回值。

也就是说，相较于 `OnceFunc`，`OnceValue` 相当于是进化版，它接收的 `f` 函数签名不同，可以支持返回一个值，而其他的地方与 `OnceFunc` 实现并无区别，内部也只是多了一个使用 `result T` 记录返回值的逻辑。

#### 使用示例

`sync.OnceValue` 使用示例如下：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	once := sync.OnceValue(func() int {
		sum := 0
		for i := 0; i < 1000; i++ {
			sum += i
		}
		fmt.Println("Computed once:", sum)
		return sum
	})
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			const want = 499500
			got := once()
			if got != want {
				fmt.Println("want", want, "got", got)
			}
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
}
```

执行示例代码，得到如下输出：

```bash
$ go run oncevalue/main.go 
Computed once: 499500
```

### sync.OnceValues

#### 源码解读

最后，我们再来看下 `sync.OnceValues` 源码实现：

> [https://github.com/golang/go/blob/go1.23.0/src/sync/oncefunc.go#L74](https://github.com/golang/go/blob/go1.23.0/src/sync/oncefunc.go#L74)
>

```go
// OnceValues returns a function that invokes f only once and returns the values
// returned by f. The returned function may be called concurrently.
//
// If f panics, the returned function will panic with the same value on every call.
func OnceValues[T1, T2 any](f func() (T1, T2)) func() (T1, T2) {
	var (
		once  Once
		valid bool
		p     any
		r1    T1
		r2    T2
	)
	g := func() {
		defer func() {
			p = recover()
			if !valid {
				panic(p)
			}
		}()
		r1, r2 = f()
		f = nil
		valid = true
	}
	return func() (T1, T2) {
		once.Do(g)
		if !valid {
			panic(p)
		}
		return r1, r2
	}
}
```

`OnceValues` 与 `OnceValue` 的唯一区别就是它支持返回两个值，`f` 函数签名也变了 `f func() (T1, T2)`，所以我们可以想到最常见的使用方式，函数 `f` 返回一个 `value` 和一个 `error`，这也是 Go 函数惯用法。

#### 使用示例

`sync.OnceValues` 使用示例如下：

```go
package main

import (
	"fmt"
	"os"
	"sync"
)

func main() {
	once := sync.OnceValues(func() ([]byte, error) {
		fmt.Println("Reading file once")
		return os.ReadFile("oncevalues/example_test.go")
	})
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			data, err := once()
			if err != nil {
				fmt.Println("error:", err)
			}
			_ = data // Ignore the data for this example
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
}
```

执行示例代码，得到如下输出：

```bash
$ go run oncevalues/main.go
Reading file once
```

现在，`sync.Once` 以及它的三个相关函数我们就都讲解完成了。

### 总结

`sync.Once` 是一个非常实用的同步工具，它以简洁高效的方式，确保操作只执行一次，避免了重复初始化的开销。在多 goroutine 调用场景下，它能提供可靠的并发控制，是 Go 并发编程中不可或缺的工具。常用于单例模式、懒加载、并发安全的初始化等场景。

Go 1.21 还发布了几个 `sync.Once` 相关的函数 `sync.OnceFunc`、`sync.OnceValue` 和 `sync.OnceValues`，来增强 `sync.Once` 功能。

我们可以发现一些规律：

+ `sync.Once.Do` 是 `sync.Once` 暴露的唯一接口，对于参数 `f` 函数确保仅执行一次。
+ 而 `sync.OnceFunc`、`sync.OnceValue` 和 `sync.OnceValues`，三者是对 `sync.Once` 的封装，都能实现 `once` 的功能，并且这三者对 `f` 函数产生 `panic` 的情况进行了处理，保证多次调用它们都能产生同样的 `panic`。
+ `sync.Once.Do` 和 `sync.OnceFunc` 接收的参数 `f` 函数，无参数无返回值。
+ `sync.OnceValue` 接收的参数 `f` 函数有 1 个返回值。
+ `sync.OnceValues` 接收的参数 `f` 函数有 2 个返回值。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/sync/once) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ sync Documentation：[https://pkg.go.dev/sync@go1.23.0](https://pkg.go.dev/sync@go1.23.0)
+ Go 1.21 Release Notes：[https://go.dev/doc/go1.21#syncpkgsync](https://go.dev/doc/go1.21#syncpkgsync)
+ What does "hot path" mean in the context of sync.Once?：[https://stackoverflow.com/questions/59174176/what-does-hot-path-mean-in-the-context-of-sync-once](https://stackoverflow.com/questions/59174176/what-does-hot-path-mean-in-the-context-of-sync-once)
+ Go 语言中的结构体内存对齐你了解吗？：[https://jianghushinian.cn/2024/07/07/do-you-understand-the-memory-alignment-of-structs-in-the-go/#hot-path](https://jianghushinian.cn/2024/07/07/do-you-understand-the-memory-alignment-of-structs-in-the-go/#hot-path)
+ Go 中空结构体惯用法，我帮你总结全了！：[https://jianghushinian.cn/2024/06/02/i-have-summarized-all-the-usages-of-empty-struct-in-go-for-you/](https://jianghushinian.cn/2024/06/02/i-have-summarized-all-the-usages-of-empty-struct-in-go-for-you/)
+ Go 错误处理指北：Defer、Panic、Recover 三剑客：[https://jianghushinian.cn/2024/10/13/go-error-guidelines-defer-panic-recover/](https://jianghushinian.cn/2024/10/13/go-error-guidelines-defer-panic-recover/)
+ Go 常见设计模式之单例模式：[https://jianghushinian.cn/2022/03/04/Golang-常见设计模式之单例模式/](https://jianghushinian.cn/2022/03/04/Golang-常见设计模式之单例模式/)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/sync/once](https://github.com/jianghushinian/blog-go-example/tree/main/sync/once)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)

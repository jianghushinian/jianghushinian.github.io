---
title: 以 Go 语言为例解释什么是伪共享以及如何解决
date: 2024-07-04 11:04:34
tags:
- Go
- 翻译
categories:
- Go
---

> 本文翻译自：[What’s false sharing and how to solve it (using Golang as example)](https://medium.com/@genchilu/whats-false-sharing-and-how-to-solve-it-using-golang-as-example-ef978a305e10)

### 译文

在解释**伪共享（false sharing）**之前，有必要简要介绍一下 CPU 架构中缓存是如何工作的。

CPU 缓存中的最小单位是**缓存行（cache line）**（如今，CPU 中常见的缓存行大小为 64 字节）。因此，当 CPU 从内存读取一个变量时，它会同时读取该变量附近的所有变量。图 1 是一个简单的例子：

![图 1](figure1.png)

当 `core1` 从内存读取变量 `a` 时，它会同时将变量 `b` 读入缓存。（顺便说一句，我认为 CPU 从内存批量读取变量的主要原因是基于**空间局部性理论**：当 CPU 访问一个变量时，它可能很快就会读取旁边的变量。）

这种缓存架构存在一个问题：如果一个变量像图 2 那样存在于不同 CPU 核心的两个缓存行中：

![图 2](figure2.png)

当 `core1` 更新变量 `a` 时：

![图 3](figure3.png)

即使变量 `b` 没有被修改，它也会导致 `core2` 的缓存失效，因此 `core2` 将重新加载缓存行中的所有变量，如图 4 所示：

![图 4](figure4.png)

这就是所谓的**伪共享**：一个核心更新一个变量会迫使其他核心也更新缓存。我们都知道，CPU 从缓存读取变量比从内存读取要快得多。因此，当这个变量总是存在于多核中时，这将显著影响性能。

解决这个问题的常见方法是**缓存填充（cache padding）**：在变量之间填充一些无意义的变量。这将迫使一个变量单独占用一个核心的缓存行，所以当其他核心更新缓存变量时，不会使该核心从内存中重新加载变量。

让我们使用下面的 Go 代码片段简要介绍这个伪共享的概念。

这里是一个包含 3 个 `uint64` 的 Go 结构体，

```go
type NoPad struct {
	a uint64
	b uint64
	c uint64
}

func (myatomic *NoPad) IncreaseAllEles() {
	atomic.AddUint64(&myatomic.a, 1)
	atomic.AddUint64(&myatomic.b, 1)
	atomic.AddUint64(&myatomic.c, 1)
}
```

还有另一个我在变量之间添加了 `[8]uint64` 以进行填充的结构体：

```go
type Pad struct {
	a   uint64
	_p1 [8]uint64
	b   uint64
	_p2 [8]uint64
	c   uint64
	_p3 [8]uint64
}

func (myatomic *Pad) IncreaseAllEles() {
	atomic.AddUint64(&myatomic.a, 1)
	atomic.AddUint64(&myatomic.b, 1)
	atomic.AddUint64(&myatomic.c, 1)
}
```

然后我编写了一个简单的代码来运行基准测试：

```go
func testAtomicIncrease(myatomic MyAtomic) {
	paraNum := 1000
	addTimes := 1000
	var wg sync.WaitGroup
	wg.Add(paraNum)
	for i := 0; i < paraNum; i++ {
		go func() {
			for j := 0; j < addTimes; j++ {
				myatomic.IncreaseAllEles()
			}
			wg.Done()
		}()
	}
	wg.Wait()

}
func BenchmarkNoPad(b *testing.B) {
	myatomic := &NoPad{}
	b.ResetTimer()
	testAtomicIncrease(myatomic)
}

func BenchmarkPad(b *testing.B) {
	myatomic := &Pad{}
	b.ResetTimer()
	testAtomicIncrease(myatomic)
}
```

在 2014 年的 MBA 上运行的基准测试如下：

```bash
$> go test -bench=.
BenchmarkNoPad-4 2000000000 0.07 ns/op
BenchmarkPad-4 2000000000 0.02 ns/op
PASS
ok 1.777s
```

基准测试的结果显示，性能从 `0.07 ns/op` 提高到了 `0.02 ns/op`，这是一个很大的改进。

你也可以在其他语言（如 Java）中测试这一点，我相信你会得到相同的结果。

在你将其应用到生产环境之前，你应该了解两个关键点：

1. 确保了解你系统中 CPU 的缓存行大小：这与你使用的缓存填充大小有关。
2. 填充更多变量意味着你将消耗更多的内存资源。运行基准测试并确保你的投入是值得的。

我所有的示例代码都在 [GitHub](https://github.com/genchilu/concurrencyPractice/tree/master/golang/pad) 上。

### P.S.

之所以选择翻译此文，是因为我正在写关于 Go 结构体内存对齐的文章，需要介绍「伪共享」这个概念，受限于篇幅所限，就决定针对伪共享这个概念单独写一篇文章。我在查阅资料过程中发现此文讲解浅显易懂，于是想着把此文翻译下共读者查阅。

不过虽然这篇文章思路清晰易懂，但作者提供的基准测试代码并不够严谨，我在 2022 款 M2 芯片的 MBA 上测试得到如下结果：

```bash
$ go test -bench=. -v
goos: darwin
goarch: arm64
pkg: false-sharing
BenchmarkNoPad
BenchmarkNoPad-8        1000000000               0.09618 ns/op
BenchmarkPad
BenchmarkPad-8          1000000000               0.1065 ns/op
PASS
ok      false-sharing      2.368s
```

跟作者的结果完全相反，哈哈😄。

我们可以写一个简单的基准测试来验证使用 `cache padding` 来解决 `false sharing` 的效果：

```go
package main

import (
	"sync/atomic"
	"testing"
)

func BenchmarkPadding(b *testing.B) {
	b.Run("without_padding", func(b *testing.B) {
		nums := [128]atomic.Int64{}
		i := atomic.Int64{}
		b.RunParallel(func(pb *testing.PB) {
			id := i.Add(1)
			for pb.Next() {
				nums[id].Add(1)
			}
		})
	})
	b.Run("with_padding", func(b *testing.B) {
		type pad struct {
			val atomic.Int64
			_   [8]uint64
		}
		nums := [128]pad{}
		i := atomic.Int64{}
		b.RunParallel(func(pb *testing.PB) {
			id := i.Add(1)
			for pb.Next() {
				nums[id].val.Add(1)
			}
		})
	})
}
```

在 `without_padding` 场景中，由于 `nums` 数组的元素可能共享相同的缓存行，多个 `goroutine` 同时修改相邻元素时会导致缓存行失效，从而降低性能。

而在 `with_padding` 场景中，通过在高频访问的变量之间加入缓存填充 `_ [8]uint64`，使得每个元素都占据独立的缓存行，减少了这种缓存行的失效情况，预期能观察到性能的提升。

执行基准测试代码，输出如下：

```bash
$ go test -bench=. -v
goos: darwin
goarch: arm64
pkg: false-sharing
BenchmarkPadding
BenchmarkPadding/without_padding
BenchmarkPadding/without_padding-8              55441728                22.09 ns/op
BenchmarkPadding/with_padding
BenchmarkPadding/with_padding-8                 1000000000               1.075 ns/op
PASS
ok      false-sharing      4.255s
```

基准测试的结果显示，性能从 `22.09 ns/op` 提高到了 `1.075 ns/op`。

这段代码由原文评论区提供，你可以自行尝试验证。

**延伸阅读**

- 原文地址: https://medium.com/@genchilu/whats-false-sharing-and-how-to-solve-it-using-golang-as-example-ef978a305e10

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn

---
title: '万字长文：彻底掌握 Go 1.23 中的迭代器'
date: 2025-07-17 23:45:20
tags:
- Go
categories:
- Go
---

本文带大家一起来深入探究一下 Go 1.23 中发布的迭代器特性，这是一篇迟来的文章，再不写这篇文章 Go 1.25 就发布了 :)，Go 1.25 预计将于 2025 年 8 月发布。

<!-- more -->

### 何为迭代器

维基百科对迭代器的定义如下：

> 迭代器（英语：iterator），是使用户可在容器对象（container，例如链表或数组）上遍访的对象，设计人员使用此接口无需关心容器对象的内存分配的实现细节。其行为很像数据库技术中的光标（cursor），迭代器最早出现在1974年设计的CLU编程语言中。
>
> 在各种语言实现迭代器的方式皆不尽同，有些面向对象语言像Java、C#、Ruby、Python、Delphi都已将迭代器的特性内置语言当中，完美的跟语言集成，我们称之隐式迭代器。但像是C++语言本身就没有迭代器的特色，但STL仍利用模板实现了功能强大的迭代器。STL容器的数据的内存地址可能会重新分配（reallocate），与容器绑定的迭代器仍然可以定位到重新分配后的正确的内存地址。
>

虽然这段对迭代器的定义比较晦涩，但可以看出，许多主流编程语言都原生支持迭代器。而 Go 也终于在 1.23 中支持了迭代器特性，这是继泛型特性以来 Go 在语法层面上的又一次重大更新。

我们再来看看 Go 官方是如何定义迭代器的，Go 官方在 1.23 版本新增的 [iter 包文档](https://pkg.go.dev/iter@go1.23.0) 中对于迭代器定义如下：

> An iterator is a function that passes successive elements of a sequence to a callback function, conventionally named yield. The function stops either when the sequence is finished or when yield returns false, indicating to stop the iteration early. 
>

翻译如下：

> 迭代器是一个将序列的连续元素传递给回调函数（通常命名为 yield）的函数。该函数在序列结束或 yield 返回 false（表示提前停止迭代）时停止。
>

看到这个定义，你可能还不是很理解，不过没关系，你现在只需要记得迭代器是一个函数即可，接下来我们将通过此文彻底掌握 Go 迭代器。

### 为什么要引入迭代器

在 [discussions/56413](https://github.com/golang/go/discussions/56413) 中 Go 团队技术负责人 rsc 对 Go 中为什么要引入迭代器做了诠释：

> In the standard library alone, we have [archive/tar.Reader.Next](https://pkg.go.dev/archive/tar/#Reader.Next), [bufio.Reader.ReadByte](https://pkg.go.dev/bufio/#Reader.ReadByte), [bufio.Scanner.Scan](https://pkg.go.dev/bufio/#Scanner.Scan), [container/ring.Ring.Do](https://pkg.go.dev/container/ring/#Ring.Do), [database/sql.Rows](https://pkg.go.dev/database/sql#Rows), [expvar.Do](https://pkg.go.dev/expvar/#Do), [flag.Visit](https://pkg.go.dev/flag/#Visit), [go/token.FileSet.Iterate](https://pkg.go.dev/go/token/#FileSet.Iterate), [path/filepath.Walk](https://pkg.go.dev/path/filepath/#Walk), [go/token.FileSet.Iterate](https://pkg.go.dev/go/token/#FileSet.Iterate), [runtime.Frames.Next](https://pkg.go.dev/runtime/#Frames.Next), and [sync.Map.Range](https://pkg.go.dev/sync/#Map.Range), hardly any of which agree on the exact details of iteration. Even the functions that agree on the signature don’t always agree about the semantics. For example, most iteration functions that return (T, bool) follow the usual Go convention of having the bool indicate whether the T is valid. In contrast, the bool returned from [runtime.Frames.Next](https://pkg.go.dev/runtime/#Frames.Next) indicates whether the next call will return something valid.
>

大概意思是说，仅标准库中，就包含这么多迭代器函数，并且这些函数接口并未统一，各自为政。甚至有些迭代器即使在函数签名上达成了一致，在语义上却各不相同。这就使得我们学习 Go 代码的成本提高了，而且与 Go 语言面向工程的简洁特点相悖。

于是，这个由 rsc 牵头，社区中争议很大的特性在 Go 1.23 中落地了。

这里举几个例子，你可以先来感受一下 Go 语言在未提供迭代器特性时的现状：

```go
// bufio.Reader
r := bufio.NewReader(...)
for {
    line, _ , err := r.ReadLine()
    if err != nil {
        break
    }
    // do something
}


// bufio.Scanner
scanner := bufio.NewScanner(...)
for scanner.Scan() {
    line := scanner.Text()
    // do something
}


// database/sql.Rows
rows, _ := db.QueryContext(...)
for rows.Next() {
    if err := rows.Scan(...); err != nil {
        break
    }
    // do something
}
```

可以发现，Go 语言中迭代器函数的设计非常混乱，接口设计各不相同，这些设计可能都是当时的最佳实践，但是接口不统一的事实，确实存在问题，急需一个统一的标准来解决此问题。

所以，其实迭代器在 Go 语言中并不是什么新鲜的东西，它们一直存在，只不过各个迭代器函数实现接口并不统一。这个问题早期也许不明显，但随着 Go 语言标准库功能的增多以及泛型特性的引入，越来越多的泛型集合实现，也都需要设计迭代器接口。因此，语法层面的迭代器特性呼之欲出。

现在，你大概理解什么是迭代器了吗？你可以简单的，将能够作用于 `for` 循环，并不断产生下一个值的对象，称为迭代器。

### 迭代器示例

我们已经对迭代器有了初步的认识，为了加深你对 Go 中迭代器的理解，现在我们一起来自己实现一下迭代器。

#### 迭代器模式

Go 是现代语言中的新贵，虽然与主流面向对象语言在设计上有所出入，不过 Go 也支持通过设计模式中的迭代器模式来实现迭代器特性。

示例如下：

```go
package main

import "fmt"

type Iterator struct {
	data  []int
	index int
}

func NewIterator(data []int) *Iterator {
	return &Iterator{data: data, index: 0}
}

func (it *Iterator) HasNext() bool {
	return it.index < len(it.data)
}

func (it *Iterator) Next() int {
	if !it.HasNext() {
		panic("Stop iteration")
	}
	value := it.data[it.index]
	it.index++
	return value
}

func main() {
	it := NewIterator([]int{0, 1, 2, 3, 4})
	for it.HasNext() {
		fmt.Println(it.Next())
	}
}
```

这是 Go 语言实现的简化版本的迭代器模式，`NewIterator` 创建并返回一个迭代器对象，`it.HasNext()` 返回迭代器中是否还有下一个值，刚好可以作用于 `for` 循环对其进行遍历，`it.Next()` 返回迭代器中的下一个值。

#### 回调函数风格

此外，因为 Go 语言支持高阶函数，即函数可以作为另一个函数的参数，所以我们还可以通过回调函数的方式，来实现迭代器。

示例如下：

```go
package main

import (
	"container/ring"
	"fmt"
)

func main() {
	// 循环链表
	r := ring.New(5)
    // 初始化链表
	for i := 0; i < r.Len(); i++ {
		r.Value = i  // 为当前节点赋值
		r = r.Next() // 移动到下一个节点
	}
    // 迭代器
	r.Do(func(v any) {
		fmt.Println(v)
	})
}
```

这里引入了标准库中的 `container/ring` 包，这个包实现了循环链表，它定义了 `*Ring.Do` 方法来对链表 `Ring` 对象进行遍历操作。

`Do` 方法按正向顺序对链表中的每个元素调用函数 `f`，其实现如下：

```go
// Do calls function f on each element of the ring, in forward order.
// The behavior of Do is undefined if f changes *r.
func (r *Ring) Do(f func(any)) {
	if r != nil {
		f(r.Value)
		for p := r.Next(); p != r; p = p.next {
			f(p.Value)
		}
	}
}
```

可以看到，`Do` 方法内部有一个 `for` 循环，不停的调用 `r.Next()` 来获取链表中的下一个值，然后调用回调函数 `f` 并将下一个值传递给它。

#### Go 风格迭代器

以上两种方式实现的迭代器，在其他主流编程语言中也很容易复刻。现在我们再来使用 `channel` 数据结构实现一种 Go 语言独有风格的迭代器。

示例如下：

```go
package main

import "fmt"

func generator(n int) <-chan int {
	ch := make(chan int)
	go func() {
		for i := 0; i < n; i++ {
			ch <- i
		}
		close(ch)
	}()
	return ch
}

func main() {
	for n := range generator(5) {
		fmt.Println(n)
	}
}
```

这里我们定义了一个生成器函数 `generator`，它返回一个只读的 `<-chan int`。而 `channel` 类型恰好能够作用于 `for-range` 循环，生成器 `generator` 不断产生下一个值，`for-range` 循环就能接收到，这样我们就实现了 Go 风格的迭代器。

虽然这个版本的迭代器非常具有 Go 风格，但因为使用了 `channel` 所以性能较差，不建议作为实现迭代器的首选方案。

有了前面的铺垫，接下来我们就要真正进入到 Go 语言官方推出的迭代器的学习了。

### 何时引入迭代器

Go 引入迭代器的过程可谓一波多折，在 2020 年 8 月的 [issues/40605](https://github.com/golang/go/issues/40605) 中就有人为 Go 2 提出了迭代器提案，之后又经历了在 [issues/43557](https://github.com/golang/go/issues/43557)、[discussions/54245](https://github.com/golang/go/discussions/54245)、[discussions/56413](https://github.com/golang/go/discussions/56413)、[issues/61405](https://github.com/golang/go/issues/61405)、[issues/61897](https://github.com/golang/go/issues/61897) 中多次讨论。终于，迭代器特性以实验性质被加入到 [Go 1.22 版本](https://go.dev/doc/go1.22)中，通过 `GOEXPERIMENT=rangefunc` 编译参数可以在 Go 中启用 [range-over-function](https://go.dev/wiki/RangefuncExperiment) 迭代器特性。此外，rsc 还在 [issues/61898](https://github.com/golang/go/issues/61898) 中提出了 `golang.org/x/exp/xiter` 包的提案，用于适配迭代器，不过这个提案最终被撤销了。

2024 年 8 月，随着 [Go 1.23 版本](https://go.dev/doc/go1.23)的发布，Go 迭代器也终于正式落地了。同时引入了 [iter](https://pkg.go.dev/iter@go1.23.0) 包，来为用户自定义迭代器提供便利。

这便是 Go 引入迭代器的主要时间脉络。

### 改变

Go 语言发布之初，`for-range` 循环就能够支持遍历 `array`、`slice`、`string`、`map`、`channel` 几种内置类型。

在 Go 1.22 中，`for-range` 新增了对整数类型值（`integer value`）的支持，比如 `for i := range 10 {...}`。

而在 Go 1.23 中，Go 引入了迭代器特性，`for-range` 又新增了对如下三种特定函数类型的支持：

```go
// 无返回值迭代器，函数 f 签名为：func(func() bool)
for range f {...}

// 返回一个值迭代器，函数 f 签名为：func(func(V) bool)
for x := range f { ... }

// 返回两个值迭代器，函数 f 签名为：func(func(K, V) bool)
for x, y := range f { ... }
```

现在，`for-range` 支持遍历的所有类型如下：

> [https://go.dev/ref/spec#For_range](https://go.dev/ref/spec#For_range)
>

```go
Range expression                                       1st value                2nd value

array or slice      a  [n]E, *[n]E, or []E             index    i  int          a[i]       E
string              s  string type                     index    i  int          see below  rune
map                 m  map[K]V                         key      k  K            m[k]       V
channel             c  chan E, <-chan E                element  e  E
integer value       n  integer type, or untyped int    value    i  see below
function, 0 values  f  func(func() bool)
function, 1 value   f  func(func(V) bool)              value    v  V
function, 2 values  f  func(func(K, V) bool)           key      k  K            v          V
```

至此，Go 语言 `for-range` 语句支持的可迭代对象类型完整拼图已经形成，真正实现了“万物皆可遍历”的能力。

### 迭代器使用示例

基于以上的讲解，你可能对 Go 迭代器还是有些不明所以，那么现在，咱们一起从代码层面，切身感受一下迭代器的存在。我将通过几个示例代码，带你体验迭代器的用法。

#### 迭代器实现

##### 最简单的迭代器

我们先来实现一个最简单的迭代器。

示例如下：

```go
package main

import (
	"fmt"
)

func iterator(yield func() bool) {
	for i := 0; i < 5; i++ {
		if !yield() {
			return
		}
	}
}

func main() {
	i := 0
	for range iterator {
		fmt.Printf("i=%d\n", i)
		i++
	}
}
```

`iterator` 函数就是一个迭代器，它接收一个 `yield` 函数作为参数，并且函数签名符合 `func(func() bool)` 类型，那么 `iterator` 就能够作用于 `for-range` 循环。

在 `iterator` 函数内部，我们启动了一个 `for` 循环，这个循环将会迭代 5 次。并且在 `for` 循环代码块内部，每次迭代都会调用 `yield()` 函数，并判断其返回值，如果返回 `false`，则直接使用 `return` 提前终止循环。

执行示例代码，得到输出如下：

```bash
$ go run main.go
i=0
i=1
i=2
i=3
i=4
```

这个示例看起来有点傻，但这确实已经是 Go 中最简单的迭代器实现了。

你也许比较疑惑，这个示例代码的执行逻辑是什么？咱们暂且不去深究原理，挖一个坑留在这里，稍后再来填上。

现在，你只需要知道这段代码大概按照如下流程来执行：

+ 首先，符合函数签名 `func(func() bool)` 的函数，都可以作为迭代器，所以 `iterator` 是一个迭代器，可以应用于 `for-range` 循环。
+ 接着，当 `for-range` 开始迭代时，迭代器函数 `iterator` 会被调用，并执行其内部代码。这里你需要注意，`iterator` 函数只会被调用一次。
+ 现在重点来了，`iterator` 函数接收一个 `yield` 函数作为参数，那么这个 `yield` 函数是哪里来的呢？其实它是 Go 编译器帮我们自动生成的函数，并自动作为参数传递给 `iterator`。`yield` 函数内部包含了 `for-range` 循环体中的代码，你可以简单的将 `yield` 函数理解为如下伪代码：

```go
func yield() bool {
    fmt.Printf("i=%d\n", i)
    i++
    return true
}
```

+ 在 `iterator` 函数内部，有一个 `for` 循环，将执行 5 次。每一次循环都会调用 `yield` 函数，并根据其返回值决定是否停止迭代。
+ `for` 循环一旦结束，`iterator` 函数便执行完成，即 `for-range` 迭代完成。

现在，是不是对迭代器的认知更清晰了一些呢？

##### 控制迭代次数

刚刚的迭代器实现过于简单，甚至都无法从外部控制迭代次数，为了控制迭代次数，我们可以给 `iterator` 再嵌套一层函数。

示例如下：

```go
package main

import (
	"fmt"
)

func iterator(n int) func(yield func() bool) {
	return func(yield func() bool) {
		for i := 0; i < n; i++ {
			if !yield() {
				return
			}
		}
	}
}

func main() {
	i := 0
	for range iterator(3) {
		fmt.Printf("i=%d\n", i)
		i++
	}
}
```

这里为 `iterator` 又嵌套了一层函数，这样就可以通过参数的形式控制迭代次数了。

执行示例代码，得到输出如下：

```bash
$ go run main.go
i=0
i=1
i=2
```

值得注意的是，现在 `for-range` 遍历迭代器时，不再是直接使用函数名称 `for range iterator` 的形式进行迭代，而是要调用 `iterator` 函数以 `for range iterator(3)` 的形式来迭代。所以，实际上 `iterator(3)` 的返回值，才是真正的迭代器函数。

这个示例代码的套路是不是有点熟悉？用 Go 写过 Web 程序的读者应该能够体会得到，比如我们在用 [Gin](https://github.com/gin-gonic/gin) 框架开发项目时，编写的中间件程序就经常使用这种套路。如开源项目 [miniblog](https://github.com/onexstack/miniblog) 中的 [AuthnMiddleware](https://github.com/onexstack/miniblog/blob/master/internal/pkg/middleware/gin/authn.go#L29) 中间件实现。

##### 输出一个值

我们已经展示了两个迭代器示例，但是，它们都不产生值，在使用 `for-range` 进行迭代时，真的就是纯迭代，`for-range` 接收不到任何循环变量，所以感觉也没啥大用。

那么我们再来实现一个能够在每轮迭代时输出一个值的迭代器。

示例如下：

```go
package main

import (
	"fmt"
)

func iterator(n int) func(yield func(v int) bool) {
	return func(yield func(v int) bool) {
		for i := 0; i < n; i++ {
			if !yield(i) {
				return
			}
		}
	}
}

func main() {
	i := 0
	for v := range iterator(10) {
		if i >= 5 {
			break
		}
		fmt.Printf("%d => %d\n", i, v)
		i++
	}
}
```

这个版本的迭代器函数签名已经变了，不再是 `func(func() bool)`，而变成了 `func(func(V) bool)`。即 `yield` 函数支持接收一个参数 `v int`。

执行示例代码，得到输出如下：

```bash
$ go run main.go
0 => 0
1 => 1
2 => 2
3 => 3
4 => 4
```

现在这个示例程序终于看起来有点意义了，像那么回事了。

##### 输出两个值

讲到这里，想必你已经猜到接下来我们要实现的迭代器功能了。

没错，`for-range` 在迭代容器类型数据结构时，是支持接收两个循环变量的。比如迭代 `slice` 时会产生 `index` 和 `value` 两个循环变量，迭代 `map` 时会产生 `key`、`value` 两个循环变量。

我们先来实现支持迭代 `slice` 类型的迭代器。

示例如下：

```go
package main

import (
	"fmt"
)

func iterator(slice []int) func(yield func(i, v int) bool) {
	return func(yield func(i int, v int) bool) {
		for i, v := range slice {
			if !yield(i, v) {
				return
			}
		}
	}
}

func main() {
	s := []int{0, 1, 2, 3, 4}
	for i, v := range iterator(s) {
		if i == 2 {
			continue
		}
		fmt.Printf("%d => %d\n", i, v)
	}
}
```

这个迭代器函数签名为 `func(func(K, V) bool)`，即支持输出两个值，分别是 `slice` 对象索引（`i`）和元素（`v`）。

执行示例代码，得到输出如下：

```bash
$ go run main.go
0 => 0
1 => 1
3 => 3
4 => 4
```

根据输出，可以发现我们正确的得到了 `index` 和 `value`。并且，`continue` 语句也可以正常应用于迭代器。

##### 迭代 map

最后，我们再来来实现一下支持迭代 `map` 类型的迭代器。

示例如下：

```go
package main

import (
	"fmt"
)

func iterator(m map[string]int) func(yield func(k string, v int) bool) {
	return func(yield func(k string, v int) bool) {
		for k, v := range m {
			if !yield(k, v) {
				return
			}
		}
	}
}

func main() {
	m := map[string]int{
		"a": 0,
		"b": 1,
		"c": 2,
	}
	for k, v := range iterator(m) {
		fmt.Printf("%s: %d\n", k, v)
	}
}
```

可以发现，这里其实并没有什么新的内容，甚至迭代器函数签名都没变 `func(func(K, V) bool)`，只不过 `K`、`V` 的类型变了。只需简单改造，我们就已经实现了支持迭代 `map` 类型的迭代器。

执行示例代码，得到输出如下：

```bash
$ go run main.go
b: 1
c: 2
a: 0
```

本小结，我们一起实现了几个迭代器。这些迭代器看起来有点用，但好像也没那么有用。而且，仅从示例程序来看，我们好像把事情变得更加复杂了。

如你所见，目前来看的确如此。比如迭代 `slice` 对象，只需要用 `for i, v := range slice` 即可，为什么还需要构造一个迭代器呢？

咱们接着往下看，你会得到答案。

#### 泛型版本

初次接触 Go 迭代器时，你可能也和我一样，有点云里雾里，Get 不到其用意。

现在，我们使用泛型来重新实现一遍这几个迭代器函数，相信你会对迭代器的用途有更深入的理解。

##### 输出零个值

以下是泛型版本最简单的迭代器实现。

示例如下：

```go
package main

import (
	"fmt"
)

type Seq0 func(yield func() bool)

func iter0[Slice ~[]E, E any](s Slice) Seq0 {
	return func(yield func() bool) {
		for range s {
			if !yield() {
				return
			}
		}
	}
}

func main() {
	s1 := []int{1, 2, 3}
	i := 0
	for range iter0(s1) {
		fmt.Printf("i=%d\n", i)
		i++
	}

	fmt.Println("--------------")

	s2 := []string{"a", "b", "c"}
	i = 0
	for range iter0(s2) {
		fmt.Printf("i=%d\n", i)
		i++
	}
}
```

为了让代码可读性更好一些，我将迭代器函数签名定义为 `Seq0` 类型，命名中的 0 表示迭代器输出零个值。

为了体现泛型的作用，这里还分别演示了迭代 `int` 和 `string` 两种类型的 `slice` 对象 `s1` 和 `s2`。

执行示例代码，得到输出如下：

```bash
$ go run main.go
i=0
i=1
i=2
--------------
i=0
i=1
i=2
```

##### 输出一个值

同样是迭代 `slice` 对象，输出一个值的迭代器实现如下：

```go
package main

import (
	"fmt"
)

type Seq1[V any] func(yield func(V) bool)

func iter1[Slice ~[]E, E any](s Slice) Seq1[E] {
	return func(yield func(E) bool) {
		for _, v := range s {
			if !yield(v) {
				return
			}
		}
	}
}

func main() {
	s1 := []int{1, 2, 3}
	for v := range iter1(s1) {
		fmt.Printf("v=%d\n", v)
	}

	fmt.Println("--------------")

	s2 := []string{"a", "b", "c"}
	for v := range iter1(s2) {
		fmt.Printf("v=%s\n", v)
	}
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
v=1
v=2
v=3
--------------
v=a
v=b
v=c
```

##### 输出两个值

进一步，输出两个值的迭代器实现如下：

```go
package main

import (
	"fmt"
)

type Seq2[K, V any] func(yield func(K, V) bool)

func iter2[Slice ~[]E, E any](s Slice) Seq2[int, E] {
	return func(yield func(int, E) bool) {
		for i, v := range s {
			if !yield(i, v) {
				return
			}
		}
	}
}

func main() {
	s1 := []int{1, 2, 3}
	for i, v := range iter2(s1) {
		fmt.Printf("%d=%d\n", i, v)
	}

	fmt.Println("--------------")

	s2 := []string{"a", "b", "c"}
	for i, v := range iter2(s2) {
		fmt.Printf("%d=%s\n", i, v)
	}
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
0=1
1=2
2=3
--------------
0=a
1=b
2=c
```

看到这几个泛型版本迭代器的实现，是不是觉得 Go 的迭代器设计确实有点用。针对某种数据结构，我们只需要编写一个迭代器函数，就能迭代此数据结构类型的所有对象。

现在，你知道如何实现泛型版本的 `map` 迭代器了吗？这个实现就留给你自行去完成了。

本小结最后提醒一下，其实迭代器中的 `yield` 并非关键字，你可以随意命名，这是只是 Go 官方推荐的约定俗成的名字。

### iter 包

前文中，我提到过，Go 随着 1.23 迭代器的发布，新增了 [iter](https://pkg.go.dev/iter@go1.23.0) 包，而新的 `iter` 包为用户操作自定义迭代器提供了基础定义。

如下这两个函数类型，就是 `iter` 包提供的：

> [https://go.dev/doc/go1.23#iterators](https://go.dev/doc/go1.23#iterators)
>

```go
type Seq[V any] func(yield func(V) bool)

type Seq2[K, V any] func(yield func(K, V) bool)
```

也就是说，其实我们在前文中实现的迭代器，有更便捷的写法。我们无需自己定义 `Seq` 和 `Seq2` 类型，`iter` 包已经为我们定义好了。我们只需要拿过来用就行了。

示例如下：

```go
package main

import "iter"

func iter1[Slice ~[]E, E any](s Slice) iter.Seq[E] {
	return func(yield func(E) bool) {
		for _, v := range s {
			if !yield(v) {
				return
			}
		}
	}
}

func iter2[Slice ~[]E, E any](s Slice) iter.Seq2[int, E] {
	return func(yield func(int, E) bool) {
		for i, v := range s {
			if !yield(i, v) {
				return
			}
		}
	}
}
```

此外，`iter` 包还提供了两个函数 `Pull` 和 `Pull2`，这两个函数签名如下：

```go
func Pull[V any](seq Seq[V]) (next func() (V, bool), stop func())

func Pull2[K, V any](seq Seq2[K, V]) (next func() (K, V, bool), stop func())
```

现在我们还无需知道它们是干什么的，稍后会有专门的小节讲解。

Go 1.23 除了新增 `iter` 包，也对原有的 [slices](https://go.dev/pkg/slices) 包和 [maps](https://go.dev/pkg/maps) 进行了增强，为二者新增了很多函数方便与迭代器一起工作。

#### slices 包

[slices](https://go.dev/pkg/slices)  包添加了如下几个与迭代器一起工作的函数：

+ [All](https://go.dev/pkg/slices#All) 返回一个遍历切片**索引和值**的迭代器。
+ [Values](https://go.dev/pkg/slices#Values) 返回一个遍历切片**元素**的迭代器。
+ [Backward](https://go.dev/pkg/slices#Backward) 返回一个**反向遍历切片**的迭代器（从末尾向开头遍历）。
+ [Collect](https://go.dev/pkg/slices#Collect) 将迭代器中的值收集到一个**新切片**中。
+ [AppendSeq](https://go.dev/pkg/slices#AppendSeq) 将迭代器中的值追加到**现有切片**中。
+ [Sorted](https://go.dev/pkg/slices#Sorted) 将迭代器中的值收集到新切片，并**排序**该切片。
+ [SortedFunc](https://go.dev/pkg/slices#SortedFunc) 功能同 `Sorted`，但支持**自定义比较函数**。
+ [SortedStableFunc](https://go.dev/pkg/slices#SortedStableFunc) 功能同 `SortedFunc`，但使用**稳定排序算法**（保持相等元素的原始顺序）。
+ [Chunk](https://go.dev/pkg/slices#Chunk) 返回一个遍历切片中**连续子切片**的迭代器，每个子切片最多包含 `n` 个元素。

源码如下：

> [https://github.com/golang/go/blob/go1.23.0/src/slices/iter.go](https://github.com/golang/go/blob/go1.23.0/src/slices/iter.go)
>

```go
package slices

import (
	"cmp"
	"iter"
)

// All returns an iterator over index-value pairs in the slice
// in the usual order.
func All[Slice ~[]E, E any](s Slice) iter.Seq2[int, E] {
	return func(yield func(int, E) bool) {
		for i, v := range s {
			if !yield(i, v) {
				return
			}
		}
	}
}

// Backward returns an iterator over index-value pairs in the slice,
// traversing it backward with descending indices.
func Backward[Slice ~[]E, E any](s Slice) iter.Seq2[int, E] {
	return func(yield func(int, E) bool) {
		for i := len(s) - 1; i >= 0; i-- {
			if !yield(i, s[i]) {
				return
			}
		}
	}
}

// Values returns an iterator that yields the slice elements in order.
func Values[Slice ~[]E, E any](s Slice) iter.Seq[E] {
	return func(yield func(E) bool) {
		for _, v := range s {
			if !yield(v) {
				return
			}
		}
	}
}

...
```

可以发现，这里的 `All` 函数其实就是我们在前文中实现的 `iter2` 迭代器，`Values` 函数就是 `iter1` 迭代器。

这里只粘贴了部分源码，更多实现，就靠你自己去研究了。你还可以查看 [https://pkg.go.dev/slices@go1.23.0](https://pkg.go.dev/slices@go1.23.0) 文档，文档中有每一个迭代器使用示例。

#### maps 包

[maps](https://go.dev/pkg/maps) 包添加了如下几个与迭代器一起工作的函数：

+ [All](https://go.dev/pkg/maps#All) 返回一个遍历映射中**键值对**的迭代器。
+ [Keys](https://go.dev/pkg/maps#Keys) 返回一个遍历映射中**键**的迭代器。
+ [Values](https://go.dev/pkg/maps#Values) 返回一个遍历映射中**值**的迭代器。
+ [Insert](https://go.dev/pkg/maps#Insert) 将迭代器中的键值对添加到**现有 map** 中。
+ [Collect](https://go.dev/pkg/maps#Collect) 将迭代器中的键值对收集到一个**新的 map** 中并返回。

源码如下：

> [https://github.com/golang/go/blob/go1.23.0/src/maps/iter.go](https://github.com/golang/go/blob/go1.23.0/src/maps/iter.go)
>

```go
package maps

import "iter"

// All returns an iterator over key-value pairs from m.
// The iteration order is not specified and is not guaranteed
// to be the same from one call to the next.
func All[Map ~map[K]V, K comparable, V any](m Map) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		for k, v := range m {
			if !yield(k, v) {
				return
			}
		}
	}
}

// Keys returns an iterator over keys in m.
// The iteration order is not specified and is not guaranteed
// to be the same from one call to the next.
func Keys[Map ~map[K]V, K comparable, V any](m Map) iter.Seq[K] {
	return func(yield func(K) bool) {
		for k := range m {
			if !yield(k) {
				return
			}
		}
	}
}

// Values returns an iterator over values in m.
// The iteration order is not specified and is not guaranteed
// to be the same from one call to the next.
func Values[Map ~map[K]V, K comparable, V any](m Map) iter.Seq[V] {
	return func(yield func(V) bool) {
		for _, v := range m {
			if !yield(v) {
				return
			}
		}
	}
}

...
```

这里的 `All` 函数就是我在前文中让你自行去实现的能够支持迭代 `map` 类型的迭代器。

对于 `maps` 包更深入的学习，你还可以查看 [https://pkg.go.dev/maps@go1.23.0](https://pkg.go.dev/maps@go1.23.0) 文档，文档中有每一个迭代器使用示例。

现在，你对 Go 迭代器的理解是否又深入了一层呢？

有如下示例：

```go
package main

import (
	"fmt"
	"maps"
	"slices"
)

func main() {
	s := []string{"a", "b", "c"}
	for i, v := range slices.All(s) {
		fmt.Printf("%d => %s\n", i, v)
	}

	m := map[string]int{
		"a": 0,
		"b": 1,
		"c": 2,
	}
	for k, v := range maps.All(m) {
		fmt.Printf("%s: %d\n", k, v)
	}
}
```

看到这段代码，是不是感觉对迭代器的作用更加直观了。无论是迭代 `slice` 还是 `map` 类型，我们都只需要调用 `All` 函数，即可完成迭代。

Go 迭代器统一了这两种容器类型的迭代 API，后续如果再有容器类型需要迭代，都可以像这样实现一个 `All` 函数。这降就低了我们开发者学习不同数据结构的迭代 API 的心智负担。这也有助于形成共识，对用户培养相同的习惯，以后在对 Go 中任意类型的迭代，就成了一个很平常的操作，所有接口都符合直觉。

理想很丰满，现实有点骨感，让我们一起期待那一天的到来。

### 迭代器原理

现在，我们是时候来学习一下 Go 迭代器的原理了，让我们更进一步，探究迭代器的本质，以此来彻底掌握 Go 迭代器特性。

迭代器是一个高阶函数，它接收一个函数（`yield`）作为参数，其签名为以下三个函数类型之一：

```go
func(func() bool)
func(func(V) bool)
func(func(K, V) bool)
```

迭代器函数用于控制 `for-range` 时的迭代过程，`for-range` 可以启动一个迭代器，迭代器通过调用 `yield` 函数，将每个值（迭代器内部产生的值）传递给调用者（`for-range` 循环），`for-range` 内部的逻辑定义了 `yield` 函数内部应该如何处理每一个值，如果 `for-range` 代码块中存在 `break`、`continue`、`return` 等终止语句，那么 `yield` 函数就会返回 `false`，否则返回 `true`。

上面这句话实际上就是 Go 迭代器的大致迭代流程，但这段解释有点绕，你一定要多读几遍，加深理解。

我们以如下代码为例，讲解迭代器的底层原理：

```go
package main

import (
	"fmt"
)

func iterator(slice []int) func(yield func(i, v int) bool) {
	return func(yield func(i int, v int) bool) {
		for i, v := range slice {
			if !yield(i, v) {
				return
			}
		}
	}
}

func main() {
	s := []int{1, 2, 3, 4, 5}
	for i, v := range iterator(s) {
		if i == 3 {
			break
		}
		fmt.Printf("%d => %d\n", i, v)
	}
}
```

这是一个支持输出两个值的迭代器，我们在前文中其实已经见过了。

执行示例代码，得到输出如下：

```bash
$ go run main.go
0 => 1
1 => 2
2 => 3
```

这个结果符合预期。

Go 迭代器最让人迷惑的点就是，`iterator(s)` 返回的明明只是一个普通函数 `func(yield func(i int, v int) bool)`，它为什么就能过被迭代呢？

甚至我们把一个符合迭代器类型的空函数交给 `for-range` 迭代，Go 程序都不会报错。

示例如下：

```go
package main

import (
	"fmt"
)

func iterator(func() bool) {}

func main() {
	i := 0
	for range iterator {
		fmt.Printf("i=%d\n", i)
		i++
	}
}
```

其实，Go 编译器为我们隐藏了真相。当 `for-range` 要迭代的对象，是一个迭代器函数时，Go 编译器在编译 Go 代码时，会重写 `for-range` 语句。

也就是说，如下这段代码，在编译期间，Go 编译器会帮我们进行代码重写：

```go
for i, v := range iterator(s) {
    if i == 3 {
        break
    }
    fmt.Printf("%d => %d\n", i, v)
}
```

这段 `for-range` 代码片段，会被重写成如下代码：

```go
iterator(s)(func(i, v int) bool {
    if i == 3 {
        return false
    }
    fmt.Printf("%d => %d\n", i, v)
    return true
})
```

所以，最终 Go 编译器内部的代码应该长这样：

```go
package main

import (
	"fmt"
)

func iterator(slice []int) func(yield func(i, v int) bool) {
	return func(yield func(i int, v int) bool) {
		for i, v := range slice {
			if !yield(i, v) {
				return
			}
		}
	}
}

func main() {
	s := []int{1, 2, 3, 4, 5}
	iterator(s)(func(i, v int) bool {
		if i == 3 {
			return false
		}
		fmt.Printf("%d => %d\n", i, v)
		return true
	})
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
0 => 1
1 => 2
2 => 3
```

这个输出，与直接使用 `for-range` 对迭代器进行迭代时一致。

不过这段代码看起来稍微有点绕，咱们换种写法，就更清晰了：

```go
package main

import (
	"fmt"
)

func iterator(slice []int) func(yield func(i, v int) bool) {
	return func(yield func(i int, v int) bool) {
		for i, v := range slice {
			if !yield(i, v) {
				return
			}
		}
	}
}

func yield (i, v int) bool {
	if i == 3 {
		return false
	}
	fmt.Printf("%d => %d\n", i, v)
	return true
}

func main() {
	s := []int{1, 2, 3, 4, 5}
	iterator(s)(yield)
}
```

这里把 `for-range` 内部的逻辑迁移了出去，封装成了一个叫 `yield` 的函数。

而调用 `iterator(s)` 会返回一个新的函数 `func(yield func(i int, v int) bool)`。

所以 `iterator(s)(yield)` 这句代码，实际上是在调用 `iterator` 函数后，又直接调用了 `iterator(s)` 返回的函数。

而 `iterator(s)` 返回的这个函数 `func(yield func(i int, v int) bool)` 的参数刚好就是 `yield` 函数。

结合代码仔细阅读一下上面这几句话，确保能够完全理解，再接着向下阅读。

我在前文中说过，`for-range` 内部的逻辑定义了 `yield` 函数内部应该如何处理每一个值，如果 `for-range` 代码块中存在 `break`、`continue`、`return` 等终止语句，那么 `yield` 函数就会返回 `false`，否则返回 `true`。

这里也证实了，`for-range` 语句块中的代码逻辑，实际上是会被重写到 `yield` 函数中的，并且 `break` 会被重写成 `return false`。

而被重写后的代码执行结果不变，所以，迭代器仅仅是 Go 语言为我们提供的语法糖或者说障眼法罢了，Go 代码执行 `for-range iterator` 时，实际上还是函数调用。

那么，至此迭代器的逻辑是不是就彻底理通了呢？

且慢，事情并没有看起来这么简单。

其实我在上面展示的 Go 编译器重写后的代码并不足够准确，它只是一个精简版本，不过对我们理解 Go 迭代器底层原理来说，已经足够了。

我们重写后的 `yield` 函数，只考虑了 `break` 情况，但其实 `for-range` 语句块中的逻辑可能还会遇到 `continue`、`goto`、`return`、`defer`、`panic` 等，每种情况都需要考虑进去，所以 Go 编译器重写后的 `yield` 函数实际上是非常复杂的。

如果你想继续深入迭代器内部原理，可以参考 [https://github.com/golang/go/blob/go1.23.0/src/cmd/compile/internal/rangefunc/rewrite.go](https://github.com/golang/go/blob/go1.23.0/src/cmd/compile/internal/rangefunc/rewrite.go) 。

现在我们知道了，Go 迭代器看似设计比较复杂，但其实这已经是 Go 团队努里后的结果了，实际上 Go 编译器中的实现更加复杂。

你是否还能记起前文中介绍过标准库中的 `container/ring` 包，它所实现的回调类型的迭代器 `*Ring.Do(func(any))`，是不是就跟 Go 编译器重写后的迭代器逻辑非常类似。

### Push & Pull 迭代器

虽然我们已经学习了迭代器原理，但是关于 Go 迭代器的知识并没有学完，Go 的迭代器是分类型的。

截至目前，我们在前文中所提到的 Go 迭代器都被称为 Push 迭代器。这类迭代器的特点是由迭代器自身控制迭代的进度，迭代器负责迭代的逻辑，并会主动将元素推送给 `yield` 函数。

Go 中还有一种迭代器叫 Pull 迭代器。Pull 迭代器通过 `next()` 函数由调用方主动“拉取”元素，并可以通过 `stop()` 显式终止迭代。

接下来，我们再来学习一下 Go 中的 Pull 迭代器。

在介绍 [iter](https://github.com/golang/go/blob/go1.23.0/src/iter/iter.go) 包小节，我提到过 `iter` 包不仅提供了 `Seq` 和 `Seq2` 两个 Push 迭代器函数签名的定义。它还提供了 `Pull` 和 `Pull2` 两个函数的实现，用来支持 Pull 类型的迭代器。

函数签名如下：

> [https://github.com/golang/go/blob/go1.23.0/src/iter/iter.go](https://github.com/golang/go/blob/go1.23.0/src/iter/iter.go)
>

```go
func Pull[V any](seq Seq[V]) (next func() (V, bool), stop func()) {
    ...
}

func Pull2[K, V any](seq Seq2[K, V]) (next func() (K, V, bool), stop func()) {
    ...
}
```

这两个函数能够将 Push 迭代器转换成 Pull 迭代器，这也是目前 Go 中实现 Pull 迭代器的唯一途径。

顾名思义，`Pull` 函数用于转换 `Seq` 类型的 Push 迭代器，`Pull2` 函数则用于转换 `Seq2` 类型的 Push 迭代器。

这两个 Pull 迭代器函数返回的值都是两个函数 `next` 和 `stop`，`next` 用于获取 Pull 迭代器中的下一个值，`stop` 则用于主动停止迭代。当调用 `next` 函数时返回 `false`，说明迭代结束。当然，我们也可以主动调用 `stop`，来提前终止迭代器。

以 `Pull2` 为例，Pull 类型迭代器使用示例如下：

```go
package main

import (
	"fmt"
	"iter"
)

func iterator(slice []int) func(yield func(i, v int) bool) {
	return func(yield func(i int, v int) bool) {
		for i, v := range slice {
			if !yield(i, v) {
				return
			}
		}
	}
}

func main() {
	s := []int{1, 2, 3, 4, 5}
	next, stop := iter.Pull2(iterator(s))
	i, v, ok := next()
	fmt.Printf("i=%d v=%d ok=%t\n", i, v, ok)
	i, v, ok = next()
	fmt.Printf("i=%d v=%d ok=%t\n", i, v, ok)
	stop()
	i, v, ok = next()
	fmt.Printf("i=%d v=%d ok=%t\n", i, v, ok)
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
i=0 v=1 ok=true
i=1 v=2 ok=true
i=0 v=0 ok=false
```

通过示例，现在你能区分 Push 迭代器和 Pull 迭代器的使用场景了吗？

我在 `iter` 包源码注释中，发现了 Go 官方列举了一个 Pull 迭代器的使用示例，代码如下：

> [https://github.com/golang/go/blob/go1.23.0/src/iter/iter.go#L117-L140](https://github.com/golang/go/blob/go1.23.0/src/iter/iter.go#L117-L140)
>

```go
// Pairs returns an iterator over successive pairs of values from seq.
func Pairs[V any](seq iter.Seq[V]) iter.Seq2[V, V] {
    return func(yield func(V, V) bool) {
        next, stop := iter.Pull(seq)
        defer stop()
        for {
            v1, ok1 := next()
            if !ok1 {
                return
            }
            v2, ok2 := next()
            // If ok2 is false, v2 should be the
            // zero value; yield one last pair.
            if !yield(v1, v2) {
                return
            }
            if !ok2 {
                return
            }
        }
    }
}
```

`Pairs`函数是一个适配器函数，它将单值序列（`iter.Seq[V]`）转换为成一对值的序列（`iter.Seq2[V, V]`）。而这中间的转换就是使用 `iter.Pull` 函数来实现的。也就是说，`Pairs`函数将一个 Push 迭代器，通过 Pull 迭代器，转换成了另一个 Push 迭代器。

我们可以通过如下方式，来使用这个 `Pairs` 迭代器：

```go
package main

import (
	"fmt"
	"iter"
	"slices"
)

...

func main() {
	s := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
	// 将切片转换为单值序列
	singleSeq := slices.Values(s)

	// 生成值对序列
	pairSeq := Pairs(singleSeq)

	// 遍历值对
	for v1, v2 := range pairSeq {
		fmt.Printf("(%d, %d)\n", v1, v2)
	}
}
```

执行结果如下：

```bash
$ go run main.go
(0, 1)
(2, 3)
(4, 5)
(6, 7)
(8, 9)
```

虽然 Pull 迭代器功能很强大，但目前还并不能直接作用于 `for-range` 来对其进行迭代。还是需要将其转换成 Push 迭代器。所以，在 Go 语言中，Pull 迭代器应用并不多。比较适合用在你想要控制迭代进度的场景。

### Pull 迭代器原理

学习了 Pull 迭代器的使用，你是不是也很好奇，Pull 迭代器究竟是如何实现和运行的呢？接下来我们继续探索。

我们在前文中已经介绍了 `iter` 包所有导出的（`exported`）接口。其实在 `iter` 包源码中还定义了一个叫 `coro` 的东西。

源码如下：

> [https://github.com/golang/go/blob/go1.23.0/src/iter/iter.go](https://github.com/golang/go/blob/go1.23.0/src/iter/iter.go)
>

```go
type coro struct{}

//go:linkname newcoro runtime.newcoro
func newcoro(func(*coro)) *coro

//go:linkname coroswitch runtime.coroswitch
func coroswitch(*coro)
```

可以看到，`coro` 是一个空结构体，并且还有两个相关的函数定义，它们都通过 `//go:linkname` 指令链接到了 [runtime](https://github.com/golang/go/blob/go1.23.0/src/runtime/coro.go) 包。其实，这三者的具体实现都是放在 `runtime` 包中的，你可以在这里 [https://github.com/golang/go/blob/go1.23.0/src/runtime/coro.go](https://github.com/golang/go/blob/go1.23.0/src/runtime/coro.go) 看到。

> NOTE:
>
> 如果你对 `//go:linkname` 指令不熟悉，可以参考我的文章《[如何使用 go:linkname 指令访问 Go 包中的私有函数](https://mp.weixin.qq.com/s/nzbuLHfS4Nu2qtcd2bO6-w)》。
>

`runtime` 包中的源码比较多，并且也不是本文重点，我们就不跳转过去查看了，不影响我们对 Pull 迭代器的讲解。

关于 `coro` 我们只需要知道它实际上是比 goroutine 更加轻量的一种协程实现，被称为 coroutine。这有点类似于 Python 中的协程。`newcoro` 用于创建一个 coroutine，`coroswitch` 则可以主动切换 coroutine，使当前 coroutine 让出执行权，交给其他 coroutine 去执行。

我们在使用 goroutine 时，一般不会主动切换到其他 goroutine，而是 Go 调度器自动帮我们进行 goroutine 切换。这是 goroutine 与 coroutine 在用法上的最大区别，也是我们理解 coroutine 的关键。 

在网上关于 Go coroutine 的资料并不多，你可以在这 [https://research.swtch.com/coro](https://research.swtch.com/coro) 看到 rsc 发表的关于 Go coroutine 的讲解。

基于以上我对 coroutine 的介绍，我们已经具备了理解 Go Pull 迭代器的前提。下面，就开始正式进入 Pull 迭代器原理的学习了。

我把 `iter.Pull` 函数代码压缩了一下，只保留了最精简的逻辑，并且我还为其增加了非常详细的注释，来方便你理解。

源码如下：

> [https://github.com/golang/go/blob/go1.23.0/src/iter/iter.go](https://github.com/golang/go/blob/go1.23.0/src/iter/iter.go)
>

```go
// 精简版 Pull 函数伪代码（保留核心协作逻辑）
func Pull[V any](seq Seq[V]) (next func() (V, bool), stop func()) {
	// 状态变量
	var (
		v         V    // 迭代值
		ok        bool // 值有效性标志
		done      bool // 迭代结束标志
		yieldNext bool // 同步标记（防止乱序调用）
	)

	// 创建迭代协程 G（未立即运行）
	c := newcoro(func(c *coro) {
		// yield 函数：G 协程逻辑
		yield := func(v1 V) bool {
			if done { // 已终止则不再继续
				return false
			}
			if !yieldNext { // 确保执行流程的正确性
				panic("iter.Pull: yield called again before next")
			}
			yieldNext = false
			v, ok = v1, true // 存储当前迭代值
			coroswitch(c)    // 让出（yield）给主协程 F
			return !done     // 返回是否继续迭代
		}

		seq(yield) // 执行原始迭代器逻辑
		var v0 V
		v, ok = v0, false // v、ok 置零
		done = true       // 标记迭代结束
	})

	// next 函数：主协程 F 恢复 G 的执行
	next = func() (v1 V, ok1 bool) {
		if done { // 迭代已结束
			return
		}
		yieldNext = true // 允许 G 执行 yield
		coroswitch(c)    // 恢复（resume）G 的执行
		return v, ok // 返回 G 通过 yield 传递的值
	}

	// stop 函数：终止迭代
	stop = func() {
		if !done {
			done = true   // 标记终止
			coroswitch(c) // 恢复 G 执行清理
		}
	}
	return next, stop
}
```

> NOTE:
>
> 如果你了解 Python，那么其实这段代码还是比较好理解的，这非常像 Python 中的协程 `await/yield` 操作主动让出协程。
>

这里，我们先就 `Pull` 函数中的几个概念达成共识，方便理解。

在 `Pull` 函数中，通过 `newcoro` 创建的 coroutine 对象 `c`，我们把它称作迭代协程 `G`。`next` 函数称为主协程 `F`，因为 `next` 本身会在主协程中被调用。`stop` 函数也在主协程中被调用，它用于终止迭代。这里涉及两个协程和三个函数，它们之间通过 `coroswitch(c)` 实现协程切换。

接下来，我就上面的 `Pull` 函数代码，讲解一下 Pull 迭代器执行逻辑。

首先，主协程 `F` 开始执行，并启动 `G`，但 `G` 不会立即运行。相反，`F` 必须显式地恢复（`resume`）`G`，然后 `G` 才会开始运行。在任意时刻，`G` 可以反转并让出执行权（`yield`）给 `F`。这会暂停 `G` 并继续 `F` 的执行（从 `F` 的 `resume` 操作之后开始）。最终 `F` 再次调用 `resume`，这会暂停 `F` 并从 `G` 的 `yield` 处继续运行 `G`。如此反复交替，直到 `G` 代码执行完成返回。这会使 `G` 结束，并从 `F` 上次的 `resume` 处继续运行 `F`，并且 `G` 会给 `F` 一个信号，G 已经完成，`F` 不应再尝试恢复 `G`。

以上，就是 Pull 迭代器执行的大致逻辑。

在这种模式中，只有一个 goroutine 在运行，主协程 `F` 和迭代协程 `G` 通过 `coroswitch(c)` 实现一种定义良好、协调的方式轮流执行。

如果你完全没接触过 coroutine 的概念，上面的讲解对你来说可能会有点抽象。没关系，我画了一张流程图来演示 `Pull` 函数的整个执行流程。

如下：

![iter.Pull](pull.png)

希望结合此图，你能对 Go 中的 Pull 迭代器原理有更深入的理解。

### 更优雅的迭代器实现

我们已经一起见识了 Go 中的迭代器，最后再对标一下其他编程语言中迭代器的实现。

#### Go 迭代器

如下是 Go 中迭代器的实现：

```go
package main

import (
	"fmt"
	"iter"
)

func iterator(n int) iter.Seq[int] {
	return func(yield func(v int) bool) {
		for i := 0; i < n; i++ {
			if !yield(i) {
				return
			}
		}
	}
}

func main() {
	for v := range iterator(5) {
		fmt.Println(v)
	}
}
```

这个代码想必现在的你已经非常熟悉了。

#### Python 迭代器

我们再来看看 Python 如何实现迭代器：

```python
def generator(num: int):  
    for i in range(num):  
        yield i


for value in generator(5):
    print(value)
```

以上是 Python 中的生成器，是实现迭代器的一种方式。相较于 Go 迭代器来说，可谓简洁优雅。

Python 除了在语法层面原生支持这种生成器，其实也定义了迭代器协议，支持用户自定义迭代器，只需要为 `class` 实现 `__iter__` 和 `__netx__` 两个方法即可。

我们可以将 Go 中的 Push 迭代器对标 Python 中的生成器，Go 中的 Pull 迭代器对标 Python 中的迭代器。

#### JavaScript 迭代器

同 Python 一样，JavaScript 也从语法层面原生支持定义生成器函数：

```javascript
function* generator(num) {
    for (let i = 0; i < num; i++) {
        yield i;
    }
}

for (const v of generator(5)) {
    console.log(v);
}
```

可以说 JavaScript 迭代器就是照抄 Python 的，几乎一模一样。

其实总结下来可以发现，同样都是使用 `yield` 关键字来定义迭代器，Python 可谓优雅，JavaScript 是跟随者，Go 则是......大(一)道(言)至(难)简(尽) :)。

虽然 Go 官方做了很多努力，已经尽力将复杂交给了编译器，将简单留给用户。但我还是想吐槽一句，Go 的迭代器设计依然不够优雅。没有对比，就没有伤害。

### 迭代器到底在解决什么问题

我们从使用到原理剖析，一起深入学习了 Go 迭代器，同时也对比了解了 Python 和 JavaScript 迭代器，那么现在回过头来看，Go 迭代器到底在解决什么问题呢？

我认为 Go 的迭代器解决了如下两个问题：

+ 统一迭代接口：解决**生态碎片化**问题，比如可以统一 Go 标准库的现有的迭代器的接口实现。
+ 隐藏实现细节：迭代器实现了**解耦遍历逻辑与数据结构**，让开发者无需关心底层实现就能统一访问各种集合（数组、链表、树等）。

不过，Go 的迭代器不止在解决问题，同时也带来了新的问题。

Go 的迭代器代码看起来并不简单易懂，导致很多初学者无法写出合适的迭代器实现。这也导致社区中出现了许多反对的声音，就像 Go 泛型，虽然讨论了这么多年，设计了非常多版本的方案，但最终还是不够完美。同样，Go 迭代器设计也不够优雅，但这可能已经是当前时间节点的最佳方案了。

Go 1.23 迭代器的核心目标是想通过标准化接口，来实现将“复杂留给实现者，简洁留给使用者”。

迭代器虽然带来了一定复杂性，但是对于使用者来说，还是方便的。比如，Go 标准库和一些第三方包，实现了迭代器。那么对于我们开发者来说，只需要调用这些包中的迭代器即可。

但是，我们在编写业务代码的过程中，避免不了要实现自己的迭代器。所以说，往往迭代器的使用者和实现者，其实是同一个人，所以，迭代器的复杂性还是留给了我们 :(。

最后，我们一起来用迭代器解决一个真实的迭代场景。

准备一个 `demo/main.go` 文件。内容如下：

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello World!")
}

```

在 `demo/` 同级目录中编写 Go 程序入口文件 `main.go`。内容如下：

```go
package main

import (
	"bufio"
	"fmt"
	"iter"
	"os"
	"strings"
)

// NOTE: 以一个读取文件内容的函数为例

// 实现一：一次性加载整个文件，可能出现内存溢出

func ProcessFile1(filename string) {
	data, _ := os.ReadFile(filename)
	lines := strings.Split(string(data), "\n") // 按换行切分
	for i, line := range lines {
		fmt.Printf("line %d: %s\n", i, line)
	}
}

// 实现二：使用 bufio 迭代器实现

func ProcessFile2(filename string) {
	file, _ := os.Open(filename)
	defer file.Close()

	scanner := bufio.NewScanner(file)
	i := 0
	for scanner.Scan() {
		fmt.Printf("line %d: %s\n", i, scanner.Text())
		i++
	}
}

// 实现三：使用 Go 1.23 迭代器进行重构

// NOTE: 实现者

func ReadLines(filename string) iter.Seq2[int, string] {
	return func(yield func(int, string) bool) {
		file, _ := os.Open(filename)
		defer file.Close()

		scanner := bufio.NewScanner(file)
		i := 0
		for scanner.Scan() {
			if !yield(i, scanner.Text()) { // 按需生成
				return
			}
			i++
		}
	}
}

// NOTE: 使用者
// 把复杂留给实现着，把**标准**留个使用者

func ProcessFile3(filename string) {
	for i, line := range ReadLines(filename) {
		fmt.Printf("line %d: %s\n", i, line)
	}
}

func main() {
	filename := "demo/main.go"
	ProcessFile1(filename)
	fmt.Println("--------------")
	ProcessFile2(filename)
	fmt.Println("--------------")
	ProcessFile3(filename)
}
```

这个示例代码无需我过多讲解，根据代码中的注释，以及你所掌握的迭代器知识，非常容易理解。

执行示例代码，得到输出如下：

```bash
$ go run main.go
line 0: package main
line 1: 
line 2: import "fmt"
line 3: 
line 4: func main() {
line 5:         fmt.Println("Hello World!")
line 6: }
line 7: 
--------------
line 0: package main
line 1: 
line 2: import "fmt"
line 3: 
line 4: func main() {
line 5:         fmt.Println("Hello World!")
line 6: }
--------------
line 0: package main
line 1: 
line 2: import "fmt"
line 3: 
line 4: func main() {
line 5:         fmt.Println("Hello World!")
line 6: }
```

我们使用三种方式对 `demo/main.go` 文件内容进行迭代输出，都能得到正确结果。

根据输出，如果你仔细观察，还会发现，第一个迭代器有一个 `line 7` 的内容输出，即 `demo/main.go` 代码最后的空行，而另外两个迭代器都已经自行处理了空行。

现在，再回过头看一下 Go 中为什么要引入迭代器，你心中有答案了吗？

### 总结

本文我们一起深入探究了 Go 语言中的迭代器特性。

我们从迭代器定义，到 Go 中的迭代器发展历程和落地，再到迭代器的使用，最后到迭代器的原理。带你从头到尾的，深入研究了 Go 迭代器。

并且我们还对比了 Python 和 JavaScript 中的迭代器，让你了解不同语言迭代器实现的异同，认识到 Go 迭代器的优缺点。

其实，最终你会发现，Go 1.23 对迭代器的支持，也仅仅只是让 `for-range` 的迭代对象，新增支持了三种函数类型。但这足以改变 Go 社区，Go 迭代器的出现，让我们使用 `for-range` “迭代万物”成为可能。

虽然 Go 社区中存在很多对迭代器的反对声音，但我依然看好 Go 迭代器的未来。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/iterator) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ wiki/迭代器：[https://zh.wikipedia.org/wiki/迭代器](https://zh.wikipedia.org/wiki/%E8%BF%AD%E4%BB%A3%E5%99%A8)
+ Go Wiki: Rangefunc Experiment：[https://go.dev/wiki/RangefuncExperiment](https://go.dev/wiki/RangefuncExperiment)
+ Go Wiki: Range Clauses：[https://go.dev/wiki/Range](https://go.dev/wiki/Range)
+ The Go Programming Language Specification：For statements with range clause：[https://go.dev/ref/spec#For_range](https://go.dev/ref/spec#For_range)
+ Range over int：[https://groups.google.com/g/golang-nuts/c/7J8FY07dkW0](https://groups.google.com/g/golang-nuts/c/7J8FY07dkW0)
+ Range Over Function Types：[https://go.dev/blog/range-functions](https://go.dev/blog/range-functions)
+ Go 1.23 Release Notes：[https://go.dev/doc/go1.23](https://go.dev/doc/go1.23)
+ Go 1.22 Release Notes：[https://go.dev/doc/go1.22](https://go.dev/doc/go1.22)
+ iter Documentation：[https://pkg.go.dev/iter@go1.23.0](https://pkg.go.dev/iter@go1.23.0)
+ slices Documentation：[https://pkg.go.dev/slices@go1.23.0](https://pkg.go.dev/slices@go1.23.0)
+ map Documentation：[https://pkg.go.dev/maps@go1.23.0](https://pkg.go.dev/maps@go1.23.0)
+ Why People are Angry over Go 1.23 Iterators：[https://www.gingerbill.org/article/2024/06/17/go-iterator-design/](https://www.gingerbill.org/article/2024/06/17/go-iterator-design/)
+ proposal: Go 2: iterators #40605：[https://github.com/golang/go/issues/40605](https://github.com/golang/go/issues/40605)
+ proposal: Go 2: function values as iterators #43557：[https://github.com/golang/go/issues/43557](https://github.com/golang/go/issues/43557)
+ discussion: standard iterator interface #54245：[https://github.com/golang/go/discussions/54245](https://github.com/golang/go/discussions/54245)
+ user-defined iteration using range over func values #56413：[https://github.com/golang/go/discussions/56413](https://github.com/golang/go/discussions/56413)
+ spec: add range over int, range over func #61405：[https://github.com/golang/go/issues/61405](https://github.com/golang/go/issues/61405)
+ iter: new package for iterators #61897：[https://github.com/golang/go/issues/61897](https://github.com/golang/go/issues/61897)
+ proposal: x/exp/xiter: new package with iterator adapters #61898：[https://github.com/golang/go/issues/61898](https://github.com/golang/go/issues/61898)
+ src/cmd/compile/internal/rangefunc/rewrite.go：[https://github.com/golang/go/blob/go1.23.0/src/cmd/compile/internal/rangefunc/rewrite.go](https://github.com/golang/go/blob/go1.23.0/src/cmd/compile/internal/rangefunc/rewrite.go)
+ Coroutines for Go：[https://research.swtch.com/coro](https://research.swtch.com/coro)
+ Coroutines for Go：[https://news.ycombinator.com/item?id=36762682](https://news.ycombinator.com/item?id=36762682)
+ Go 1.23中的自定义迭代器与iter包：[https://tonybai.com/2024/06/24/range-over-func-and-package-iter-in-go-1-23/](https://tonybai.com/2024/06/24/range-over-func-and-package-iter-in-go-1-23/)
+ Python 迭代器：[https://docs.python.org/zh-cn/3.13/tutorial/classes.html#iterators](https://docs.python.org/zh-cn/3.13/tutorial/classes.html#iterators)
+ JavaScript 迭代器和生成器：[https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Iterators_and_generators](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Iterators_and_generators)
+ A look at iterators in Go：[https://medium.com/eureka-engineering/a-look-at-iterators-in-go-f8e86062937c](https://medium.com/eureka-engineering/a-look-at-iterators-in-go-f8e86062937c)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/iterator](https://github.com/jianghushinian/blog-go-example/tree/main/iterator)
+ 本文永久地址：[https://jianghushinian.cn/2025/07/17/go-iterator/](https://jianghushinian.cn/2025/07/17/go-iterator/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

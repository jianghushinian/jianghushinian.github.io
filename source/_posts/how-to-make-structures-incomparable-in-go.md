---
title: '在 Go 中如何让结构体不可比较？'
date: 2024-06-15 08:28:48
tags:
- Go
categories:
- Go
---

最近我在使用 Go 官方出品的结构化日志包 [slog](https://pkg.go.dev/golang.org/x/exp/slog) 时，看到 `slog.Value` 源码中有一个比较好玩的小 Tips，可以限制两个结构体之间的相等性比较，本文就来跟大家分享下。

<!-- more -->

### 在 Go 中结构体可以比较吗？

在 Go 中结构体可以比较吗？这其实是我曾经面试过的一个问题，我们来做一个实验：

定义如下结构体：

```go
type Normal struct {
	a string
	B int
}
```

使用这个结构体分别声明 3 个变量 `n1`、`n2`、`n3`，然后进行比较：

```go
n1 := Normal{
	a: "a",
	B: 10,
}
n2 := Normal{
	a: "a",
	B: 10,
}
n3 := Normal{
	a: "b",
	B: 20,
}

fmt.Println(n1 == n2)
fmt.Println(n1 == n3)
```

执行示例代码，输出结果如下：

```bash
$ go run main.go
true
false
```

可见 `Normal` 结构体是可以比较的。

### 如何让结构体不可比较？

那么所有结构体都可以比较吗？显然不是，如果都可以比较，那么 `reflect.DeepEqual()` 就没有存在的必要了。

定义如下结构体：

```go
type NoCompare struct {
	a string
	B map[string]int
}
```

使用这个结构体分别声明 2 个变量 `n1`、`n2`，然后进行比较：

```go
n1 := NoCompare{
	a: "a",
	B: map[string]int{
		"a": 10,
	},
}
n2 := NoCompare{
	a: "a",
	B: map[string]int{
		"a": 10,
	},
}

fmt.Println(n1 == n2)
```

执行示例代码，输出结果如下：

```bash
$ go run main.go
./main.go:59:15: invalid operation: n1 == n2 (struct containing map[string]int cannot be compared)
```

这里程序直接报错了，并提示结构体包含了 `map[string]int` 类型字段，不可比较。

所以小结一下：

结构体是否可以比较，不取决于字段是否可导出，而是取决于其是否包含不可比较字段。

如果全部字段都是可比较的，那么这个结构体就是可比较的。

如果其中有一个字段不可比较，那么这个结构体就是不可比较的。

不过虽然我们不可以使用 `==` 对 `n1`、`n2` 进行比较，但我们可以使用 `reflect.DeepEqual()` 对二者进行比较：

```go
fmt.Println(reflect.DeepEqual(n1, n2))
```

执行示例代码，输出结果如下：

```bash
$ go run main.go
true
```

### 更优雅的做法

最近我在使用 Go 官方出品的结构化日志包 [slog](https://pkg.go.dev/golang.org/x/exp/slog) 时，看到 `slog.Value` 源码：

```go
// A Value can represent any Go value, but unlike type any,
// it can represent most small values without an allocation.
// The zero Value corresponds to nil.
type Value struct {
	_ [0]func() // disallow ==
	// num holds the value for Kinds Int64, Uint64, Float64, Bool and Duration,
	// the string length for KindString, and nanoseconds since the epoch for KindTime.
	num uint64
	// If any is of type Kind, then the value is in num as described above.
	// If any is of type *time.Location, then the Kind is Time and time.Time value
	// can be constructed from the Unix nanos in num and the location (monotonic time
	// is not preserved).
	// If any is of type stringptr, then the Kind is String and the string value
	// consists of the length in num and the pointer in any.
	// Otherwise, the Kind is Any and any is the value.
	// (This implies that Attrs cannot store values of type Kind, *time.Location
	// or stringptr.)
	any any
}
```

可以发现，这里有一个匿名字段 `_ [0]func()`，并且注释写着 `// disallow ==`。

`_ [0]func()` 的目的显然是为了禁止比较。

我们来实验一下，`_ [0]func()` 是否能够实现禁止结构体相等性比较：

```go
v1 := Value{
	num: 1,
	any: 2,
}
v2 := Value{
	num: 1,
	any: 2,
}

fmt.Println(v1 == v2)
```

执行示例代码，输出结果如下：

```bash
$ go run main.go
./main.go:109:15: invalid operation: v1 == v2 (struct containing [0]func() cannot be compared)
```

可以发现，的确有效。因为 `func()` 是一个函数，而函数在 Go 中是不可比较的。

既然使用 `map[string]int` 和 `_ [0]func()` 都能实现禁止结构体相等性比较，那么我为什么说 `_ [0]func()` 是更优雅的做法呢？

`_ [0]func()` 有着比其他实现方式更优的特点：

它不占内存空间！

使用匿名字段 `_` 语义也更强。

而且，我们直接去 Go 源码里搜索，能够发现其实 Go 本身也在多处使用了这种用法：

![_ [0]func()](go-struct-not-comparable.png)

所以推荐使用 `_ [0]func()` 来实现禁用结构体相等性比较。

不过值得**注意**的是：当使用 `_ [0]func()` 时，**不要把它放在结构体最后一个字段，推荐放在第一个字段**。这与**结构体内存对齐**有关，我在[《Go 中空结构体惯用法，我帮你总结全了！》](https://jianghushinian.cn/2024/06/02/i-have-summarized-all-the-usages-of-empty-struct-in-go-for-you/#%E7%A9%BA%E7%BB%93%E6%9E%84%E4%BD%93%E5%BD%B1%E5%93%8D%E5%86%85%E5%AD%98%E5%AF%B9%E9%BD%90) 一文中也有提及。

> NOTE: 对于 `_ [0]func()` 不占用内存空间的验证，就交给你自己去实验了。
> 提示：可以使用 `fmt.Println(unsafe.Sizeof(v1), unsafe.Sizeof(v2))` 分别打印结构体 `Value` 的两个实例 `v1`、`v2` 的内存大小。你可以删掉 `_ [0]func()` 字段再试一试。

### 总结

好了，在 Go 中如何让结构体不可比较这个小 Tips 就分享给大家了，还是比较有意思的。

我在看到 `slog.Value` 源码使用 `_ [0]func()` 来禁用结构体相等性比较时，又搜索了 Go 的源码中多处在使用，我想这应该是社区推荐的做法了。然后就尝试去网上搜索了下，还真被我搜索到了一个叫 [Phuong Le](https://x.com/func25) 的人在 [X](https://x.com/) 上发布了 [Golang Tip #50: Make Structs Non-comparable.](https://x.com/func25/status/1768621711929311620) 专门来介绍这个 Tip，并且我在中文社区也找到了[鸟窝老师](https://colobu.com/about/)在[《Go语言编程技巧》](https://colobu.com/gotips/preface.html)中的译文 [Tip #50 使结构体不可比较](https://colobu.com/gotips/050.html)。

这也印证了我的猜测，`_ [0]func()` 在 Go 社区中是推荐用法。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/struct/non-comparable) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

- slog 源码: https://github.com/golang/go/blob/master/src/log/slog/value.go#L21
- Golang Tip #50: Make Structs Non-comparable.: https://x.com/func25/status/1768621711929311620
- Go语言编程技巧: https://colobu.com/gotips/050.html
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/struct/non-comparable

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn

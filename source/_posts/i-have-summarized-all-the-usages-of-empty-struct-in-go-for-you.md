---
title: 'Go 中空结构体惯用法，我帮你总结全了！'
date: 2024-06-02 09:11:14
tags:
- Go
categories:
- Go
---

在 Go 语言中，空结构体 `struct{}` 是一个非常特殊的类型，它**不包含任何字段**并且**不占用任何内存空间**。虽然听起来似乎没什么用，但空结构体在 Go 编程中实际上有着广泛的应用。本文将详细探讨空结构体的几种典型用法，并解释为何它们在特定场景下非常有用。

<!-- more -->

### 空结构体不占用内存空间

首先我们来验证下空结构体是否占用内存空间：

```go
type Empty struct{}

var s1 struct{}
s2 := Empty{}
s3 := struct{}{}

fmt.Printf("s1 addr: %p, size: %d\n", &s1, unsafe.Sizeof(s1))
fmt.Printf("s2 addr: %p, size: %d\n", &s2, unsafe.Sizeof(s2))
fmt.Printf("s3 addr: %p, size: %d\n", &s3, unsafe.Sizeof(s3))
fmt.Printf("s1 == s2 == s3: %t\n", s1 == s2 && s2 == s3)
```

> NOTE: 为了保持代码逻辑清晰，这里只展示了代码主逻辑。后文中所有示例代码都会如此，完整代码可以在文末给出的示例代码 [GitHub 链接](https://github.com/jianghushinian/blog-go-example/tree/main/struct/empty)中获取。

在 Go 语言中，我们可以使用 `unsafe.Sizeof` 计算一个对象占用的字节数。

执行以上示例代码，输出结果如下：

```bash
$ go run main.go
s1 addr: 0x1044ef4a0, size: 0
s2 addr: 0x1044ef4a0, size: 0
s3 addr: 0x1044ef4a0, size: 0
s1 == s2 == s3: true
```

根据输出结果可知：

1. 多个空结构体内存地址相同。
2. 空结构体占用字节数为 0，即不占用内存空间。
3. 多个空结构体值相等。

后面两个结论很好理解，第一个结论有点反常识。为什么不同变量实例化的空结构体内存地址会相同？

真的是这样吗？我们可以看下另一个示例：

```go
var (
	a struct{}
	b struct{}
	c struct{}
	d struct{}
)

println("&a:", &a)
println("&b:", &b)
println("&c:", &c)
println("&d:", &d)

println("&a == &b:", &a == &b)
x := &a
y := &b
println("x == y:", x == y)

fmt.Printf("&c(%p) == &d(%p): %t\n", &c, &d, &c == &d)
```

这段代码中定义了 4 个空结构体，依次打印它们的内存地址，然后又分别对比了 `a` 与 `b` 的内存地址和 `c` 与 `d` 的内存地址两两是否相等。

执行示例代码，输出结果如下：

```bash
$ go run -gcflags='-m -N -l' main.go
# command-line-arguments
./main.go:11:3: moved to heap: c
./main.go:12:3: moved to heap: d
./main.go:23:12: ... argument does not escape
./main.go:23:50: &c == &d escapes to heap
&a: 0x1400010ae84
&b: 0x1400010ae84
&c: 0x104ec74a0
&d: 0x104ec74a0
&a == &b: false
x == y: true
&c(0x104ec74a0) == &d(0x104ec74a0): true
```

在 Go 语言中使用 `go run` 命令时，可以通过 `-gcflags` 选项向 Go 编译器传递多个标志，这些标志会影响编译器的行为。

- `-m` 标志用于启动编译器的内存逃逸分析。
- `-N` 标志用于禁用编译器优化。
- `-l` 标志用于禁用函数内联。

根据输出可以发现，变量 `c` 和 `d` 发生了内存逃逸，并且最终二者的内存地址相同，相等比较结果为 `true`。

而 `a` 和 `b` 两个变量的输出结果就比较有意思了，两个变量没有发生内存逃逸，并且二者打印出来的内存地址相同，但内存地址相等比较结果却为 `false`。

所以，我们可以推翻之前的结论，新结论为：「多个空结构体内存地址**可能**相同」。

在 Go 官方的语言规范中 [Size and alignment guarantees](https://go.dev/ref/spec#Size_and_alignment_guarantees) 部分对关于空结构体内存地址进行了说明：

> A struct or array type has size zero if it contains no fields (or elements, respectively) that have a size greater than zero. Two distinct zero-size variables may have the same address in memory.

大概意思是说：如果一个结构体或数组类型不包含任何占用内存大小大于零的字段（或元素），那么它的大小为零。**两个不同的零大小变量可能在内存中具有相同的地址**。

注意⚠️，这里说的是**可能**：`may have the same`。所以前文所述「多个空结构体内存地址相同」的结论并不准确。

> NOTE: 本文示例执行结果基于 `Go 1.22.0` 版本，对于多个空结构体内存地址打印结果既存在相同情况，也存在不同情况，这跟 Go 编译器实现有关，后续实现可能会有变化。

另外，对于嵌套的空结构体，其表现结果与普通空结构体相同：

```go
type Empty struct{}

type MultiEmpty struct {
	A Empty
	B struct{}
}

s1 := Empty{}
s2 := MultiEmpty{}
fmt.Printf("s1 addr: %p, size: %d\n", &s1, unsafe.Sizeof(s1))
fmt.Printf("s2 addr: %p, size: %d\n", &s2, unsafe.Sizeof(s2))
```

执行示例代码，输出结果如下：

```bash
$ go run main.go
s1 addr: 0x1044ef4a0, size: 0
s2 addr: 0x1044ef4a0, size: 0
```

### 空结构体影响内存对齐

空结构体也并不是什么时候都不会占用内存空间，比如空结构体作为另一个结构体字段时，根据位置不同，可能因内存对齐原因，导致外层结构体大小不一样：

```go
type A struct {
	x int
	y string
	z struct{}
}

type B struct {
	x int
	z struct{}
	y string
}

type C struct {
	z struct{}
	x int
	y string
}

a := A{}
b := B{}
c := C{}
fmt.Printf("struct a size: %d\n", unsafe.Sizeof(a))
fmt.Printf("struct b size: %d\n", unsafe.Sizeof(b))
fmt.Printf("struct c size: %d\n", unsafe.Sizeof(c))
```

以上示例中，定义了三个结构体 `A`、`B`、`C`，并且都定义了三个字段，类型分别是 `int`、`string`、`struct{}`，空结构体字段分别放在最后、中间、最前面不同的位置。

执行示例代码，输出结果如下：

```bash
$ go run main.go
struct a size: 32
struct b size: 24
struct c size: 24
```

可以发现，当空结构体放在另一个结构体最后一个字段时，会触发内存对齐。

此时外层结构体会占用更多的内存空间，所以如果你的程序对内存要求比较严格，则在使用空结构体作为字段时需要考虑这一点。

> NOTE: 这里先挖个坑，我会再写一篇 Go 中结构体内存对齐的文章，分析下为什么 `struct{}` 放在结构体字段最后会出现内存对齐现象，敬请期待。防止迷路，可以关注下我的公众号：[Go编程世界](https://jianghushinian.cn/about)。

### 空结构体用法

根据前文的讲解，我们对 Go 中空结构体的特性和一些使用时注意事项已经有所了解，是时候探索空结构体的用处了。

#### 实现 Set

首先，空结构体最常用的地方，就是用来实现 `set(集合)` 类型了。

我们知道 Go 语言在语法层面没有提供 `set` 类型。不过我们可以很方便的使用 `map` + `struct{}` 来实现 `set` 类型，代码如下：

```go
// Set 基于空结构体实现 set
type Set map[string]struct{}

// Add 添加元素到 set
func (s Set) Add(element string) {
	s[element] = struct{}{}
}

// Remove 从 set 中移除元素
func (s Set) Remove(element string) {
	delete(s, element)
}

// Contains 检查 set 中是否包含指定元素
func (s Set) Contains(element string) bool {
	_, exists := s[element]
	return exists
}

// Size 返回 set 大小
func (s Set) Size() int {
	return len(s)
}

// String implements fmt.Stringer
func (s Set) String() string {
	format := "("
	for element := range s {
		format += element + " "
	}
	format = strings.TrimRight(format, " ") + ")"
	return format
}

s := make(Set)

s.Add("one")
s.Add("two")
s.Add("three")

fmt.Printf("set: %s\n", s)
fmt.Printf("set size: %d\n", s.Size())
fmt.Printf("set contains 'one': %t\n", s.Contains("one"))
fmt.Printf("set contains 'onex': %t\n", s.Contains("onex"))

s.Remove("one")

fmt.Printf("set: %s\n", s)
fmt.Printf("set size: %d\n", s.Size())
```

执行示例代码，输出结果如下：

```bash
$ go run main.go
set: (one two three)
set size: 3
set contains 'one': true
set contains 'onex': false
set: (three two)
set size: 2
```

使用 `map` 和空结构体非常容易实现 `set` 类型。`map` 的 `key` 实际上与 `set` 不重复的特性刚好一致，一个不需要关心 `value` 的 `map` 即为 `set`。

也正因为如此，空结构体类型最适合作为这个不需要关心的 `value` 的 `map` 了，因为它**不占空间，没有语义**。

也许有人会认为使用 `any` 作为 `map` 的 `value` 也可以实现 `set`。但其实 `any` 是会占用空间的。

示例如下：

```go
s := make(map[string]any)
s["t1"] = nil
s["t2"] = struct{}{}
fmt.Printf("set t1 value: %v, size: %d\n", s["t1"], unsafe.Sizeof(s["t1"]))
fmt.Printf("set t2 value: %v, size: %d\n", s["t2"], unsafe.Sizeof(s["t2"]))
```

执行示例代码，输出结果如下：

```bash
$ go run main.go
set t1 value: <nil>, size: 16
set t2 value: {}, size: 16
```

可以发现，`any` 类型的 `value` 是有大小的，所以并不合适。

日常开发中，我们还会用到一种 `set` 的惯用法：

```go
s := map[string]struct{}{
	"one":   {},
	"two":   {},
	"three": {},
}
for element := range s {
	fmt.Println(element)
}
```

这种用法也比较常见，无需声明一个 `set` 类型，直接通过字面量定义一个 `value` 为空结构体的 `map`，非常方便。

#### 申请超大容量 Array

基于空结构体不占内存空间的特性，我们可以考虑创建一个容量为 `100` 万的 `array`：

```go
var a [1000000]string
var b [1000000]struct{}

fmt.Printf("array a size: %d\n", unsafe.Sizeof(a))
fmt.Printf("array b size: %d\n", unsafe.Sizeof(b))
```

执行示例代码，输出结果如下：

```bash
$ go run main.go
array a size: 16000000
array b size: 0
```

使用空结构体创建的 `array` 其大小依然为 `0`。

#### 申请超大容量 Slice

我们还以考虑创建一个容量为 `100` 万的 `slice`：

```go
var a = make([]string, 1000000)
var b = make([]struct{}, 1000000)
fmt.Printf("slice a size: %d\n", unsafe.Sizeof(a))
fmt.Printf("slice b size: %d\n", unsafe.Sizeof(b))
```

执行示例代码，输出结果如下：

```bash
$ go run main.go
slice a size: 24
slice b size: 24
```

当然，可以发现，其实不管是否使用空结构体，`slice` 只占用 `header` 的空间。

#### 信号通知

空结构体另一个我经常使用的方法是与 `channel` 结合当作信号来使用，示例如下：

```go
done := make(chan struct{})

go func() {
    time.Sleep(1 * time.Second) // 执行一些操作...
    fmt.Printf("goroutine done\n")
    done <- struct{}{} // 发送完成信号
}()

fmt.Printf("waiting...\n")
<-done // 等待完成
fmt.Printf("main exit\n")
```

这段代码中声明了一个长度为 `0` 的 `channel`，其类型为 `chan struct{}`。

然后启动一个 `goroutine` 执行业务逻辑，主协程等待信号退出，二者使用 `channel` 进行通信。

执行示例代码，输出结果如下：

```bash
$ go run main.go
waiting...
goroutine done
main exit
```

主协程先输出 `waiting...`，然后等待 1s，`goroutine` 输出 `goroutine done`，接着主协程收到退出信号，输出 `main exit` 程序执行完成。

由于 `struct{}` 并不占用内存，所以实际上 `channel` 内部只需要将计数器加一即可，不涉及数据传输，故没有额外内存开销。

这段代码还有另一种实现：

```go
done := make(chan struct{})

go func() {
	time.Sleep(1 * time.Second) // 执行一些操作...
	fmt.Printf("goroutine done\n")
	close(done) // 不需要发送 struct{}{}，直接 close，发送完成信号
}()

fmt.Printf("waiting...\n")
<-done // 等待完成
fmt.Printf("main exit\n")
```

这里 `goroutine` 中都不需要发送空结构体，直接对 `channel` 进行 `close` 就行了，`struct{}` 在这里起到的作用更像是一个「占位符」的作用。

在 Go 语言 `context` 源码中也使用了 `struct{}` 作为完成信号：

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	// See https://blog.golang.org/pipelines for more examples of how to use
	// a Done channel for cancellation.
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

`context.Context` 的 `Done` 方法返回值即为 `chan struct{}`。

#### 无操作的方法接收器

有时候，我们需要“组合”一些方法，并且这些方法内部并不会用到方法`接收器`，这时就可以使用 `struct{}` 作为方法接收器。

```go
type NoOp struct{}

func (n NoOp) Perform() {
	fmt.Println("Performing no operation.")
}
```

方法中代码并没有引用 `n`，如果换成其他类型则会占用内存空间。

在实际开发过程中，有时候代码写到一半，为了编译通过，我们也会写出这种代码，先写出代码整体框架，再实现内部细节。

#### 作为接口实现

用 `struct{}` 作为方法接收器，还有另一个用途，就是作为接口的实现。常用于忽略不需要的输出，和单元测试。啥意思呢？往下看。

我们知道 Go 中有个 `io.Writer` 接口：

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

我们还知道，Go 的 `io` 包中有个 `io.Discard` 变量，它的主要作用是提供一个“黑洞”设备，任何写入到 `io.Discard` 的数据都会被消耗掉而不会有任何效果（这类似于 Unix 中的 `/dev/null` 设备）。

`io.Discard` 定义如下：

```go
// Discard is a [Writer] on which all Write calls succeed
// without doing anything.
var Discard Writer = discard{}

type discard struct{}

func (discard) Write(p []byte) (int, error) {
	return len(p), nil
}
```

`io.Discard` 代码定义极其简单，它实现了 `io.Writer` 接口，并且这个 `Writer` 方法的实现也极其简单，什么都没做直接返回。

根据注释也能发现，`Writer` 方法的目的就是啥都不做，所有调用都会成功，所以可以类比为 Unix 系统中的 `/dev/null`。

`io.Discard` 可以用于忽略日志：

```go
// 设置日志输出为 `io.Discard`，忽略所有日志
log.SetOutput(io.Discard)
// 这条日志不会在任何地方显示
log.Println("This log will not be shown anywhere")
```

此外，我曾写过一篇文章[《在 Go 语言单元测试中如何解决 MySQL 存储依赖问题》](https://jianghushinian.cn/2023/07/16/how-to-resolve-mysql-dependencies-in-go-testing/#Fake-%E6%B5%8B%E8%AF%95)。里面有这样一段示例代码：

```go
type UserStore interface {
	Create(user *User) error
	Get(id int) (*User, error)
}

...

type fakeUserStore struct{}

func (f *fakeUserStore) Create(user *store.User) error {
	return nil
}

func (f *fakeUserStore) Get(id int) (*store.User, error) {
	return &store.User{ID: id, Name: "test"}, nil
}
```

这就是空结构体作为接口实现的另一种用途，编写测试用 `fake object` 时非常有用。

即我们定义一个 `struct{}` 类型 `fakeUserStore`，然后实现 `UserStore` 接口，这样在单元测试代码中，就可以用 `fakeUserStore` 来替换真实的 `UserStore` 实例对象，以此来解决对象间的依赖问题。

#### 标识符

最后，我们再来介绍一种空结构体比较好玩的用法。

相信很多同学都直接或间接的使用过 Go 中的 `sync.Pool`，其定义如下：

```go
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer
	localSize uintptr

	victim     unsafe.Pointer
	victimSize uintptr

	New func() any
}
```

其中有一个 `noCopy` 属性，其定义如下： 

```go
type noCopy struct{}

func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

`noCopy` 即为一个空结构体，其实现也非常简单，仅定义了两个空方法。

而这个 `noCopy` 属性看似没什么用，实际上却有着大作用。这个字段的主要作用是阻止 `sync.Pool` 被意外复制。它是一种通过编译器静态分析来防止结构体被不当复制的技巧，以确保正确的使用和内存安全性。

可以通过 `go vet` 命令检测出 `sync.Pool` 是否被意外复制。 

在这里，`noCopy` 属性对当前结构体本身没有作用，但可以将其作为一个是否允许复制的标识符，有了这个标记，就代表结构体不能被复制，`go vet` 命令就可以检查出来。

我们自定义的 `struct` 也可以通过嵌入 `noCopy` 属性来实现禁止复制：

```go
package main

type noCopy struct{}

func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}

func main() {
	type A struct {
		noCopy noCopy
		a      string
	}

	type B struct {
		b string
	}

	a := A{a: "a"}
	b := B{b: "b"}

	_ = a
	_ = b
}
```

使用 `go vet` 命令检查是否存在意外的结构体复制：

```bash
$ go vet main.go

# command-line-arguments
# [command-line-arguments]
./main.go:21:6: assignment copies lock value to _: command-line-arguments.A contains command-line-arguments.noCopy
```

可以发现，`go vet` 已经检测出我们通过 `_ = a` 复制了 `noCopy` 结构体 `A`。

### 总结

空结构体 `struct{}` 在 Go 中虽小却有着巧妙的用途。

从节省内存的角度看，它是表示空概念的理想选择。从语义上考虑，使用 `struct{}` 语义更明确，就是不关注值。

由于内存对齐的影响，空结构体字段顺序可能影响外层结构体的大小，建议将空结构体放在外层结构体的第一个字段。

无论是使用空结构体实现集合、信号通知、方法载体还是占位符等，`struct{}` 都显示了其独特的价值。

你还知道空结构体还有哪些用途，可以分享出来大家一起交流学习。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/struct/empty) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

- The empty struct: https://dave.cheney.net/2014/03/25/the-empty-struct
- The Ingenious World of Empty Structs in Go: Zeroing in on Zero Memory: https://medium.com/@cosmicray001/the-ingenious-world-of-empty-structs-in-go-zeroing-in-on-zero-memory-a7050279fe18
- What is the use of empty struct in GoLang: https://www.pixelstech.net/article/1677371161-What-is-the-use-of-empty-struct-in-GoLang
- Using empty structs as context keys: https://gist.github.com/SammyOina/6eb54babd618ab6a850e8f1af4f4ac7d
- Size and alignment guarantees: https://go.dev/ref/spec#Size_and_alignment_guarantees
- What uses a type with empty struct has in Go?: https://stackoverflow.com/questions/47544156/what-uses-a-type-with-empty-struct-has-in-go
- runtime/malloc.go: https://github.com/golang/go/blob/master/src/runtime/malloc.go#L904
- 在 Go 语言单元测试中如何解决 MySQL 存储依赖问题: https://jianghushinian.cn/2023/07/16/how-to-resolve-mysql-dependencies-in-go-testing/#Fake-%E6%B5%8B%E8%AF%95
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/struct/empty

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn

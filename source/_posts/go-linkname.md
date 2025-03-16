---
title: '如何使用 go:linkname 指令访问 Go 包中的私有函数'
date: 2025-03-16 21:45:56
tags:
- Go
- 编译器指令
categories:
- Go
---

在 Go 语言的包设计中，函数和变量通过首字母大小写来严格区分导出（exported）与未导出（unexported）的可见性规则。这种机制是 Go 模块化设计的基石，但同时也为底层系统级开发带来了限制。`//go:linkname` 指令正是 Go 为突破这一限制预留的「后门」，它通过编译器的符号重定向能力，允许开发者直接链接任意包的未导出符号——无论是标准库的私有函数，还是第三方包的隐藏变量。

本文就来带大家一起体验下 `//go:linkname` 指令的魔力。

<!-- more -->

### go:linkname 指令简介

`//go:linkname` 是 Go 语言中的一个编译器指令，用于在编译阶段将当前包内的函数或变量与另一个包中函数或变量（即使是未导出的）进行链接。

语法格式如下：

```go
//go:linkname localname [importpath.name]
```

例如，我们要自定义一个 `FastRand` 函数并链接到 `runtime` 包的私有函数 `fastrand`，可以这样写：

```plain
//go:linkname FastRand runtime.fastrand
func FastRand() uint32
```

这样，我们只需要声明 `FastRand` 函数，而无需实现，当调用 `FastRand()` 时就会自动执行 `runtime.fastrand()` 的调用。

`//go:linkname` 指令支持 3 种模式，Pull 模式、Push 模式以及 Handshake 模式，这 3 种模式会在下文中依次讲解。

### 准备工作

我准备了如下目录结构，用于演示 `//go:linkname` 指令的功能。

```bash
$  tree linkname   
linkname
├── bar
│   └── bar.go
├── foo
│   ├── dummy.s
│   └── foo.go
├── go.mod
└── main.go

3 directories, 5 files
```

`main.go` 是程序入口，`foo/foo.go` 用来定义程序的导出函数，`bar/bar.go` 用来定义程序的未导出函数。所以 `main.go` 不会直接导入 `bar/bar.go`，而是通过导入 `foo/foo.go` 间接与 `bar/bar.go` 交互。

### Pull 模式

Pull（拉取）模式下 `bar` 包中的函数无需任何特殊处理，`foo` 包使用 `//go:linkname` 指令链接到 `bar` 包中的函数。示例如下：

首先在 `bar/bar.go` 中实现一个 `add` 函数：

> https://github.com/jianghushinian/blog-go-example/blob/main/directive/linkname/bar/bar.go
>

```go
package bar

func add(a, b int) int {
	return a + b
}
```

然后在 `foo/foo.go` 中使用 `//go:linkname` 指令链接到 `bar` 包中的 `add` 函数：

> https://github.com/jianghushinian/blog-go-example/blob/main/directive/linkname/foo/foo.go
>

```go
package foo

import (
	_ "unsafe"

	// 被拉取的包需要显式导入（除了 runtime 包）
	_ "github.com/jianghushinian/blog-go-example/directive/linkname/bar"
)

// Pull 模式（拉取外部实现）

//go:linkname Add github.com/jianghushinian/blog-go-example/directive/linkname/bar.add
func Add(a, b int) int
```

这里有两点需要注意：

1. 被拉取的包需要显式导入到当前包中（`runtime` 包除外），所以这里使用了匿名导入的方式导入 `bar` 包。
2. 要使用 `//go:linkname` 指令，必须要导入 `unsafe` 包，因为这是一个不安全的操作，所以这里同样使用匿名导入的方式导入 `unsafe` 包。

这里声明了 `Add` 函数但并未实现，并使用 `//go:linkname` 指令将 `Add` 函数链接到 `bar.add` 函数。

这就完成了 Pull 模式。

最后我们编写一个 `main` 函数来测试下效果：

> https://github.com/jianghushinian/blog-go-example/blob/main/directive/linkname/main.go
>

```go
package main

import (
	"fmt"

	"github.com/jianghushinian/blog-go-example/directive/linkname/foo"
)

func main() {
	fmt.Println("foo.Add(1, 2):", foo.Add(1, 2))

}
```

我们可以像正常函数一样导入并使用 `foo.Add` 函数。

执行示例代码，得到输出如下：

```bash
$ go run main.go                          
foo.Add(1, 2): 3
```

看起来一切正常。

不过，这种模式存在极大的安全隐患，在这种模式中，`bar` 包可能并不打算让 `foo` 使用其 `add` 函数，`bar` 包也不知道会有其他包链接它的函数，当维护 `bar` 包的人修改了 `add` 函数签名，则 `foo` 包就无法运行了。

由此我们可以尝试换一种方式进行链接，使用 Push 模式。

### Push 模式

Push（推送）模式下 `bar` 包使用 `//go:linkname` 指令将定义的未导出函数“重命名”为 `foo` 包中的导出函数，`foo` 包中只需要声明函数签名即可。示例如下：

首先在 `bar/bar.go` 中实现一个 `div` 函数：

> https://github.com/jianghushinian/blog-go-example/blob/main/directive/linkname/bar/bar.go
>

```go
package bar

import (
	_ "unsafe"
)

// Push 模式（导出本地实现）

//go:linkname div github.com/jianghushinian/blog-go-example/directive/linkname/foo.Div
func div(a, b int) int {
	return a / b
}
```

这里使用 `//go:linkname` 指令将未导出函数 `div` “重命名”为 `foo.Div`。

然后我们在 `foo/foo.go` 中声明 `Div` 函数：

> https://github.com/jianghushinian/blog-go-example/blob/main/directive/linkname/foo/foo.go
>

```go
package foo

import (
	// 被拉取的包需要显式导入（除了 runtime 包）
	_ "github.com/jianghushinian/blog-go-example/directive/linkname/bar"
)

func Div(a, b int) int
```

这里同样需要显式导入 `bar` 包，并且只声明 `Div` 函数而不做实现。

最后同样编写一个 `main` 函数来测试下效果：

> https://github.com/jianghushinian/blog-go-example/blob/main/directive/linkname/main.go
>

```go
package main

import (
	"fmt"

	"github.com/jianghushinian/blog-go-example/directive/linkname/foo"
)

func main() {
	fmt.Println("foo.Div(2, 1):", foo.Div(2, 1))
}
```

现在，我们来尝试执行下这个示例，得到输出如下：

```bash
$ go run main.go
# github.com/jianghushinian/blog-go-example/directive/linkname/foo
foo/foo.go:8:6: missing function body
```

这里得到了报错，提示 `foo.Div` 函数没有 `body` 体。

解决办法也很简单，我们只需要在 `foo.go` 同级目录下加上一个内容为空的 `dummy.s` 文件就好了。这是 Go 语言的约定，你知道就好，其实这个文件名叫什么无所谓，只要是表示汇编程序的 `.s` 结尾的文件就可以。

再次执行示例程序，得到输出如下：

```bash
$ go run main.go
foo.Div(2, 1): 2
```

这种模式下我们在 `bar` 包中主动暴露了未导出函数 `bar.div`，这样 `bar` 包就可以明确的知道自己会链接 `foo.Div` 函数。不过，这种模式带来了新的问题，就是需要引入一个空的 `dummy.s` 文件。

那么有没有更好的方式呢？当然，答案就是 Handshake 模式。

### Handshake 模式

Handshake（握手）模式故名思义，需要链接双方共同参与，携手实现最终目标。Handshake 模式下 `bar` 包使用 `//go:linkname` 指令将定义的未导出函数标记为允许被外部包链接，`foo` 包同样使用 `//go:linkname` 指令链接到 `bar` 包中的函数。示例如下：

首先在 `bar/bar.go` 中实现一个 `hello` 函数：

> https://github.com/jianghushinian/blog-go-example/blob/main/directive/linkname/bar/bar.go
>

```go
package bar

import (
	_ "unsafe"
)

// Handshake 模式（双方握手模式）

//go:linkname hello
func hello(name string) string {
	return "Hello " + name + "!"
}
```

这里使用 `//go:linkname hello` 指令标记为允许外部包进行链接。

接着在 `foo/foo.go` 中声明 `Hello` 函数：

> https://github.com/jianghushinian/blog-go-example/blob/main/directive/linkname/foo/foo.go
>

```go
package foo

import (
	_ "unsafe"

	// 被拉取的包需要显式导入（除了 runtime 包）
	_ "github.com/jianghushinian/blog-go-example/directive/linkname/bar"
)

// Handshake 模式（双方握手模式）

//go:linkname Hello github.com/jianghushinian/blog-go-example/directive/linkname/bar.hello
func Hello(name string) string
```

这里并没有新的写法，与 Pull 模式写法相同。不过需要注意的是，Handshake 模式中双方都需要引入 `unsafe` 包。

最后编写一个 `main` 函数来测试下效果：

> https://github.com/jianghushinian/blog-go-example/blob/main/directive/linkname/main.go
>

```go
package main

import (
	"fmt"

	"github.com/jianghushinian/blog-go-example/directive/linkname/foo"
)

func main() {
	fmt.Println(`foo.Hello("jianghushinian"):`, foo.Hello("jianghushinian"))
}
```

执行示例代码，得到输出如下：

```bash
$  go run main.go
foo.Hello("jianghushinian"): Hello jianghushinian!
```

可以发现，Handshake 模式其实就是 Pull 模式和 Push 模式的结合体。这种模式语义更加明确，且无需提供 `dummy.s` 文件，也是 Go 最推荐的写法。

### 内置包限制

事实上，从 Go 1.23 版本起，Go 已经不推荐 Pull 模式和 Push 模式了，仅推荐使用 Handshake 模式，具体原因，你可以在 [issues/67401](https://github.com/golang/go/issues/67401) 中查看。

由此，也引来的一个问题，那就是在 Go 1.23+ 版本中，如果使用 Pull 模式链接 Go 内置包，则默认情况下程序无法通过编译。

示例如下：

```go
package main

import (
	"fmt"
	_ "unsafe"
)

//go:linkname TooLarge fmt.tooLarge
func TooLarge(x int) bool

func main() {
	fmt.Println("TooLarge(1e6 + 1):", TooLarge(1e6+1))
}
```

这里使用 `//go:linkname` 指令将 `TooLarge` 函数链接到内部包函数 `fmt.tooLarge`。

执行示例代码，得到输出如下：

```bash
$ go run main.go
# command-line-arguments
link: main: invalid reference to fmt.tooLarge
```

要解决这个问题，需要提供编译新的编译指令 `-checklinkname=0`，用法如下：

```bash
$ go run -ldflags=-checklinkname=0 main.go
TooLarge(1e6 + 1): true
```

这样，程序就可以正常执行了。

当然，根据我的实测结果，如果把 `TooLarge` 函数的定义迁移到 `foo/foo.go` 中，然后在 `main.go` 中以 `foo.TooLarge(1e6+1)` 的方式使用，则没任何问题，无需提供编译指令 `-checklinkname=0`。

### 总结

`//go:linkname` 指令用于在编译阶段将当前包内的函数或变量与另一个包中函数或变量（即使是未导出的）进行链接，突破了 Go 中未导出标识在外部无法访问的限制。不过 `//go:linkname` 指令虽然强大，但需要慎用，毕竟它是“unsafe”的，除非你知道自己在干什么。

本文中演示了 `//go:linkname` 指令对函数的作用，关于对变量的应用读者可以自行尝试。

虽然 Go 中的 `//go:linkname` 指令支持 3 种模式：Pull 模式、Push 模式和 Handshake 模式，不过从 Go 1.23 版本起，已经只推荐使用 Handshake 模式了。至于具体原因，你可以访问 [issues/67401](https://github.com/golang/go/issues/67401) 中查看，我就不啰嗦了。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/directive/linkname) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ cmd/compile Documentation：[https://pkg.go.dev/cmd/compile@go1.24.1#hdr-Linkname_Directive](https://pkg.go.dev/cmd/compile@go1.24.1#hdr-Linkname_Directive)
+ cmd/link: lock down future uses of linkname #67401：[https://github.com/golang/go/issues/67401](https://github.com/golang/go/issues/67401)
+  the hall of shame：[https://github.com/golang/go/blob/go1.24.1/src/runtime/malloc.go#L996-L1010](https://github.com/golang/go/blob/go1.24.1/src/runtime/malloc.go#L996-L1010)
+ Go 1.23 Release Notes：[https://go.dev/doc/go1.23#linker](https://go.dev/doc/go1.23#linker)
+ Function declarations：[https://go.dev/ref/spec#Function_declarations](https://go.dev/ref/spec#Function_declarations)
+ 突破限制,访问其它Go package中的私有函数：[https://colobu.com/2017/05/12/call-private-functions-in-other-packages/](https://colobu.com/2017/05/12/call-private-functions-in-other-packages/)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/directive/linkname](https://github.com/jianghushinian/blog-go-example/tree/main/directive/linkname)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

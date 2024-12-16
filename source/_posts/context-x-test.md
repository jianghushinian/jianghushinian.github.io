---
title: 'Go 源码是如何解决测试代码循环依赖问题的？'
date: 2024-12-16 23:05:27
tags:
- Go
categories:
- Go
---

最近我写了一篇讲解 context 包源码的文章[《Go 并发控制：context 源码解读》](https://jianghushinian.cn/2024/12/09/context/)，在阅读源码的过程中，我在 context 包测试代码中发现了一个解决循环依赖的小技巧，在此分享给大家。

<!-- more -->

### x_test.go 解决循环依赖

context 包源码目录结构如下：

> [https://github.com/golang/go/tree/go1.23.0/src/context](https://github.com/golang/go/tree/go1.23.0/src/context)
>

```bash
$ tree context 
context
├── afterfunc_test.go
├── benchmark_test.go
├── context.go
├── context_test.go
├── example_test.go
├── net_test.go
└── x_test.go

1 directory, 7 files
```

`context.go` 文件是 context 包源码实现，其他都是测试文件。其中只有 `context_test.go` 的包名为 `context`，其他几个测试文件的包名则为 `context_test`。那么也就是说 `context_test.go` 是白盒测试，其他测试文件为黑盒测试。

不过，`context_test.go` 文件中并没有以 `Test` 开头的测试函数，而是定义了几个名称格式为 `XTestXxx` 的测试函数。以 `XTestCancelRemoves` 为例，其代码如下：

> [https://github.com/golang/go/blob/go1.23.0/src/context/context_test.go#L193](https://github.com/golang/go/blob/go1.23.0/src/context/context_test.go#L193)
>

```go
package context

// Tests in package context cannot depend directly on package testing due to an import cycle.
// If your test does requires access to unexported members of the context package,
// add your test below as `func XTestFoo(t testingT)` and add a `TestFoo` to x_test.go
// that calls it. Otherwise, write a regular test in a test.go file in package context_test.

import (
	"time"
)

type testingT interface {
	Deadline() (time.Time, bool)
	Error(args ...any)
	Errorf(format string, args ...any)
	Fail()
	FailNow()
	Failed() bool
	Fatal(args ...any)
	Fatalf(format string, args ...any)
	Helper()
	Log(args ...any)
	Logf(format string, args ...any)
	Name() string
	Parallel()
	Skip(args ...any)
	SkipNow()
	Skipf(format string, args ...any)
	Skipped() bool
}

...

func XTestCancelRemoves(t testingT) {
	checkChildren := func(when string, ctx Context, want int) {
		if got := len(ctx.(*cancelCtx).children); got != want {
			t.Errorf("%s: context has %d children, want %d", when, got, want)
		}
	}

	ctx, _ := WithCancel(Background())
	checkChildren("after creation", ctx, 0)
	_, cancel := WithCancel(ctx)
	checkChildren("with WithCancel child ", ctx, 1)
	cancel()
	checkChildren("after canceling WithCancel child", ctx, 0)

	ctx, _ = WithCancel(Background())
	checkChildren("after creation", ctx, 0)
	_, cancel = WithTimeout(ctx, 60*time.Minute)
	checkChildren("with WithTimeout child ", ctx, 1)
	cancel()
	checkChildren("after canceling WithTimeout child", ctx, 0)

	ctx, _ = WithCancel(Background())
	checkChildren("after creation", ctx, 0)
	stop := AfterFunc(ctx, func() {})
	checkChildren("with AfterFunc child ", ctx, 1)
	stop()
	checkChildren("after stopping AfterFunc child ", ctx, 0)
}
```

首先，`go test` 是不认识以 `XTest` 开头的函数的，其次，函数参数 `testingT` 是一个接口，并不是 `*testing.T` 结构体，所以 `XTestCancelRemoves` 不会被当作测试函数。

并且在文件开头的注释部分也说明了：

> `context` 包中的测试不能直接依赖 `testing` 包，因为会导致循环导入。如果你的测试需要访问 `context` 包中未导出的（unexported）成员，请将测试添加到下面，形式为 `func XTestFoo(t testingT)`，并在 `x_test.go` 文件中添加一个调用它的 `TestFoo` 方法。否则，请在 `context_test` 包中的 `test.go` 文件中编写常规测试。
>

所以，这种写法是为了解决循环导入的。

我在 testing 包源码中搜索了下，有两处直接导入 context 包，分别是 `deps.go` 文件和 `slogtest.go` 文件。

源码位置：

> [https://github.com/golang/go/blob/go1.23.0/src/testing/internal/testdeps/deps.go#L15](https://github.com/golang/go/blob/go1.23.0/src/testing/internal/testdeps/deps.go#L15)
>
> [https://github.com/golang/go/blob/go1.23.0/src/testing/slogtest/slogtest.go#L9](https://github.com/golang/go/blob/go1.23.0/src/testing/slogtest/slogtest.go#L9)
>

不过，实测下来这两处并不是导致循环导入的根本原因，因为它们都是 testing 的子包。如果没有用到，是不会被导入到 context 包的。

其实 testing 包源码中还有一处间接引用 context 包的地方，在 `testing.go` 中导入了 `runtime/trace` 包，而 `runtime/trace` 包内部则引入了 context 包。

源码位置：

> [https://github.com/golang/go/blob/go1.23.0/src/testing/testing.go#L385](https://github.com/golang/go/blob/go1.23.0/src/testing/testing.go#L385)
>

这个才是造成 context 包与 testing 包形成循环导入的根因。

那么为了解决这个问题，所以才抽象出 `testingT` 接口，这个接口就是照着 `*testing.T` 结构体实现的方法设计的，也就是说 `*testing.T` 结构体实现了这个接口。

但是因为 `go test` 是不认 `testingT` 接口的，所以如果将 `XTestCancelRemoves` 定义成以 `Test` 开头的单元测试函数 `TestCancelRemoves`，就会编译报错。为了解决这个问题，前面加一个 `X`，就得到了 `XTestCancelRemoves`。而 `XTestCancelRemoves` 不过是一个**普通函数**，并不是单元测试函数。所以使用 `go test` 命令执行测试代码的时候，不会执行 `XTestCancelRemoves` 函数。

那么现在这个问题就好解决了。在 `x_test.go` 中定义 `TestCancelRemoves` 单元测试函数，并且其内部调用了 `XTestCancelRemoves`，实现代码如下：

> [https://github.com/golang/go/blob/go1.23.0/src/context/x_test.go#L26](https://github.com/golang/go/blob/go1.23.0/src/context/x_test.go#L26)
>

```go
package context_test

import (
	. "context"
	"errors"
	"fmt"
	"math/rand"
	"runtime"
	"strings"
	"sync"
	"testing"
	"time"
)

// Each XTestFoo in context_test.go must be called from a TestFoo here to run.
...

func TestCancelRemoves(t *testing.T) {
	XTestCancelRemoves(t) // uses unexported context types
}
```

注意这里使用 `import . "context"` 的方式导入了 `context` 包，因为这两个文件不在同一个包，当前黑盒测试代码包名为 `context_test`，并且这里就可以导入 `testing` 包了，这样就解决了循环依赖问题。

`TestCancelRemoves` 函数以 `Test` 开头，并且参数为 `*testing.T`，这是一个标志的单元测试代码，能够被 `go test` 识别。

现在，`context`、`testing`、`context_test` 三个包的依赖情况如下：

![包依赖关系](context-test.png)

`context_test` 包导入了 `context`、`testing` 两个包，而 `context`、`testing` 两个包并没有互相导入，这也就通过抽象出一个更高的层级依赖两个下层包的方式，解决了循环导入。这也是我们平时开发时，避免循环导入的小技巧。

### export_test.go 测试后门

在分析 `x_test.go` 机制时，让我想起了 Go 语言“圣经”[《Go程序设计语言》](https://book.douban.com/subject/27044219/)一书中讲到的测试“后门”。既然都讲到这里，那么我再顺便分享一下使用 `export_test.go` 作为测试“后门”的小技巧。

![《Go程序设计语言》](go-programming-language.png)

> NOTE:
>
> 身为一名 Gopher，如果你还没读过这本 Go 语言“圣经”，那么强烈建议你读一下。
>

在《Go程序设计语言》一书 `11.2.4 外部测试包` 这一小节中也有提到使用黑盒测试解决循环引用问题。不过，如果有些包变量是 unexported 的，则可以通过编写测试“后门”来解决。

比如 `fmt` 包中有一个 unexported 的函数 `isSpace`，在黑盒测试中需要被使用。解决方案非常简单，在包名为 `fmt` 的白盒测试文件中，声明一个 exported 的新变量 `IsSpace`，并将 `isSpace` 赋值给它：

> [https://github.com/golang/go/blob/go1.23.0/src/fmt/export_test.go](https://github.com/golang/go/blob/go1.23.0/src/fmt/export_test.go)
>

```go
package fmt

var IsSpace = isSpace
var Parsenum = parsenum
```

然后就可以在黑盒测试中使用 exported 变量 `IsSpace` 了：

> [https://github.com/golang/go/blob/go1.23.0/src/fmt/fmt_test.go#L1789](https://github.com/golang/go/blob/go1.23.0/src/fmt/fmt_test.go#L1789)
>

```go
package fmt_test

import (
	"bytes"
	. "fmt"
	"internal/race"
	"io"
	"math"
	"reflect"
	"runtime"
	"strings"
	"testing"
	"time"
	"unicode"
)

...

func TestIsSpace(t *testing.T) {
	// This tests the internal isSpace function.
	// IsSpace = isSpace is defined in export_test.go.
	for i := rune(0); i <= unicode.MaxRune; i++ {
		if IsSpace(i) != unicode.IsSpace(i) {
			t.Errorf("isSpace(%U) = %v, want %v", i, IsSpace(i), unicode.IsSpace(i))
		}
	}
}
```

测试函数 `TestIsSpace` 的代码注释中也说明了 `IsSpace = isSpace` 是在 `export_test.go` 中定义的。

为测试编写“后门”原来如此简单。并且，由于以 `_test.go` 结尾的文件只在编译测试的时候才会被使用，那么正常编译 exported 变量 `IsSpace` 是不会被使用的，所以无需担心 `isSpace` 被乱用的问题。

### 总结

本文介绍了两个单元测试的小技巧，我们可以使用 `XTest` 来解决循环依赖问题，使用测试“后门”来解决黑盒测试引用白盒测试 unexported 变量问题。

另外，《Go程序设计语言》非常值得一读，推荐给大家。

希望此文能对你有所启发。

**延伸阅读**

+ go1.23.0/src/context：[https://github.com/golang/go/tree/go1.23.0/src/context](https://github.com/golang/go/tree/go1.23.0/src/context)
+ go1.23.0/src/fmt：[https://github.com/golang/go/blob/go1.23.0/src/fmt/](https://github.com/golang/go/blob/go1.23.0/src/fmt/)
+ 《Go程序设计语言》：[https://book.douban.com/subject/27044219/](https://book.douban.com/subject/27044219/)
+ Go 并发控制：context 源码解读：[https://jianghushinian.cn/2024/12/09/context/](https://jianghushinian.cn/2024/12/09/context/)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/context](https://github.com/jianghushinian/blog-go-example/tree/main/context)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

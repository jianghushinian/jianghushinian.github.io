---
title: '在 Go 中如果获取 goroutine 的 id？'
date: 2024-12-04 09:13:43
tags:
- Go
categories:
- Go
---

如果你使用过如 Python、Java 等主流支持并发的编程语言，那么通常都能够比较容易的获得进程和线程的 id。但是在 Go 语言，没有直接提供对多进程和多线程的支持，而是提供了 goroutine 来支持并发编程。不过在 Go 中，获取 goroutine 的 id 并不像其他编程语言那样容易，但依然有办法，本文就来介绍下如何实现。

<!-- more -->

### 获取当前进程的 id

首先，虽然 Go 没有提供多进程编程，但启动 Go 程序还是会有一个进程存在的，Go 标准库提供了 `os.Getpid` 函数，可以方便的获取当前进程的 id：

> [https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/id/pid/main.go](https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/id/pid/main.go)

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	// 获取当前进程的 id
	pid := os.Getpid()
	fmt.Println("process id:", pid)
}
```

调用 `os.Getpid()` 会返回当前进程的 pid（进程 id）。

### 获取当前 goroutine 的 id

Go 并没有直接提供获取 goroutine id 的方法，因为 goroutine 的管理是由 Go 运行时（Go runtime）负责的，它并不暴露每个 goroutine 的 id。然而，有一些方法可以间接获取到与 goroutine 相关的信息。

#### 使用 runtime 包获取 goroutine id

虽然不能直接获取每个 goroutine 的 id，但我们可以变相的通过 `runtime.Stack` 函数来获取。

实现代码如下：

> [https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/id/main.go](https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/id/main.go)

```go
package main

import (
	"fmt"
	"runtime"
	"strconv"
	"strings"
	"sync"
)

func GoId() int {
	buf := make([]byte, 32)
	n := runtime.Stack(buf, false)
	idField := strings.Fields(strings.TrimPrefix(string(buf[:n]), "goroutine "))[0]
	id, err := strconv.Atoi(idField)
	if err != nil {
		panic(fmt.Sprintf("cannot get goroutine id: %v", err))
	}
	return id
}

func main() {
	fmt.Println("main", GoId())
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		i := i
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println(i, GoId())
		}()
	}
	wg.Wait()
}
```

这段代码通过自定义的 `GoId` 函数来获取当前 goroutine 的 id，并在 `main` 函数的主 goroutine 和子 goroutines 中打印它们的 id。

我们来解释下 `GoId` 函数主要逻辑的实现：

1. `runtime.Stack(buf, false)`：
    - `runtime.Stack` 是 `runtime` 包提供的公开函数，用于获取当前 goroutine 的堆栈信息。
    - 参数 `buf` 是一个字节数组，用来存储调用 `runtime.Stack()` 返回的堆栈信息。
    - 第二个参数是 `false`，表示我们只获取当前 goroutine 的栈信息，如果为 `true` 则是获取所有 goroutines 的栈信息。
    - `runtime.Stack()` 会将当前 goroutine 的栈信息写入 `buf` 中，`n` 是返回的字节数，表示堆栈信息的长度。
2. `strings.TrimPrefix(string(buf[:n]), "goroutine ")`：
    - 将堆栈信息转为字符串并去掉前缀 `"goroutine "`。
    - 堆栈信息的第一行格式通常如下：

    ```plain
    goroutine 1 [running]:
    ```

    - 这里通过 `TrimPrefix` 去除前缀 `"goroutine "` 后，剩下的内容就是 goroutine 的 id 及其状态信息。
3. `idField := strings.Fields(...)[0]`：
    - `strings.Fields` 将经过 `TrimPrefix` 处理后的字符串按空格切割成一个字符串切片。
    - 从切片中获取第一个字段，这就是 goroutine 的 id，如 `1`。

如果一切顺利，`GoId` 函数最终返回当前 goroutine 的 id。

`main` 函数实现则比较简单，先调用 `GoId()` 打印主 goroutine 的 id，然后启动了 10 个子 goroutine 并分别打印它们的 id。

执行示例代码，得到如下输出：

```bash
$ go run main.go
main 1
9 29
0 20
5 25
6 26
7 27
8 28
2 22
1 21
4 24
3 23
```

这样，我们就变相的通过先获取堆栈信息，然后再从堆栈信息中进行解析的方式拿到了 goroutine 的 id。想必你也能够发现，这种实现方式性能不高，所以不到万不得已，不要轻易获取 goroutine 的 id。

那么有没有更高效的方式呢？

很遗憾，Go 官方没有提供。不过有第三方库帮我们实现了。

#### 使用第三方库获取 goroutine id

一个比较常用的库是 [https://github.com/petermattis/goid](https://github.com/petermattis/goid)，可以用来获取当前 goroutine 的 id。

安装方式：

```bash
$ go get github.com/petermattis/goid
```

使用示例：

> [https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/id/goid/main.go](https://github.com/jianghushinian/blog-go-example/blob/main/goroutine/id/goid/main.go)

```go
package main

import (
	"fmt"
	"sync"

	"github.com/petermattis/goid"
)

func main() {
	fmt.Println("main", goid.Get())
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		i := i
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println(i, goid.Get())
		}()
	}
	wg.Wait()
}
```

执行示例代码，得到如下输出：

```bash
$ go run goid/main.go
main 1
9 43
4 38
5 39
6 40
7 41
8 42
1 35
0 34
2 36
3 37
```

我们仅需要调用 `goid.Get()` 即可获取当前 goroutine 的 id。

`goid` 库使用了 C 和 汇编来获取 goroutine id，所以性能更好。并且 `goid` 为所有的 Go 版本都做了兼容，从项目文件名可以看出，不同 Go 版本有着不同的实现：

```bash
$ tree goid
goid
├── LICENSE
├── README.md
├── go.mod
├── goid.go
├── goid_gccgo.go
├── goid_go1.3.c
├── goid_go1.3.go
├── goid_go1.4.go
├── goid_go1.4.s
├── goid_go1.5.go
├── goid_go1.5.s
├── goid_slow.go
├── goid_test.go
├── runtime_gccgo_go1.8.go
├── runtime_go1.23.go
├── runtime_go1.5.go
├── runtime_go1.6.go
└── runtime_go1.9.go

1 directory, 18 files
```

在 `goid_go1.3.c` 中可以看到 C 语言版本实现如下：

> [https://github.com/petermattis/goid/blob/master/goid_go1.3.c](https://github.com/petermattis/goid/blob/master/goid_go1.3.c)

```c
// +build !go1.4

#include <runtime.h>

void ·Get(int64 ret) {
    ret = g->goid;
    USED(&ret);
}
```

在 `goid_go1.4.s` 中可以看到汇编语言版本实现如下：

> [https://github.com/petermattis/goid/blob/master/goid_go1.4.s](https://github.com/petermattis/goid/blob/master/goid_go1.4.s)

```plain
// +build amd64 amd64p32 arm 386
// +build go1.4,!go1.5

#include "textflag.h"

#ifdef GOARCH_arm
#define JMP B
#endif

TEXT ·getg(SB),NOSPLIT,$0-0
	JMP	runtime·getg(SB)
```

此外，为了保证兼容性，在 `goid.go` 中还有一个 Go 语言版本实现：

> [https://github.com/petermattis/goid/blob/master/goid.go](https://github.com/petermattis/goid/blob/master/goid.go)

```go
package goid

import (
	"bytes"
	"runtime"
	"strconv"
)

func ExtractGID(s []byte) int64 {
	s = s[len("goroutine "):]
	s = s[:bytes.IndexByte(s, ' ')]
	gid, _ := strconv.ParseInt(string(s), 10, 64)
	return gid
}

// Parse the goid from runtime.Stack() output. Slow, but it works.
func getSlow() int64 {
	var buf [64]byte
	return ExtractGID(buf[:runtime.Stack(buf[:], false)])
}
```

这里 Go 版本的实现同样使用 `runtime.Stack()`，并且注释也标明了这个实现比较慢。

所以，如果我们真的需要获取 goroutine 的 id，那么推荐使用 [goid](https://github.com/petermattis/goid/)。

### 总结

在 Go 中获取当前进程的 id 可以使用 `os.Getpid()` 函数。如果要获取当前 goroutine 的 id 则要困难一些，Go 标准库没有直接提供该功能，不过我们可以变相的从 `runtime.Stack()` 返回的堆栈信息中获取，也可以使用第三方库 [goid](https://github.com/petermattis/goid/) 来获取。

不过，你有没有想过，为什么 Go 不提供获取当前 goroutine 的 id 的功能呢？

这个问题早有讨论，其实在早期的 Go 版本中提供了这样的方法，只不过后来被取消了。讨论过程你可以在这里 [https://groups.google.com/g/golang-nuts/c/Nt0hVV_nqHE](https://groups.google.com/g/golang-nuts/c/Nt0hVV_nqHE) 看到。

总结起来就是 Go 设计故意不让用户获取 goroutine 的 id，避免滥用。所以，要不要在你的代码中使用 goroutine 的 id，你需要自定判断。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/goroutine/id) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Re: [go-nuts] How to get current goroutine id?：[https://groups.google.com/g/golang-nuts/c/Nt0hVV_nqHE](https://groups.google.com/g/golang-nuts/c/Nt0hVV_nqHE)
+ <font style="color:rgb(33, 33, 33);">How can I get a goroutine's runtime ID?：</font>[https://stackoverflow.com/questions/75361134/how-can-i-get-a-goroutines-runtime-id](https://stackoverflow.com/questions/75361134/how-can-i-get-a-goroutines-runtime-id)
+ <font style="color:rgb(33, 33, 33);">如何得到goroutine 的 id?：</font>[https://colobu.com/2016/04/01/how-to-get-goroutine-id/](https://colobu.com/2016/04/01/how-to-get-goroutine-id/)
+ <font style="color:rgb(33, 33, 33);">goid GitHub 源码：</font>[https://github.com/petermattis/goid/](https://github.com/petermattis/goid/)
+ <font style="color:rgb(33, 33, 33);">runtime.Stack Documentation：</font>[https://pkg.go.dev/runtime@go1.23.0#Stack](https://pkg.go.dev/runtime@go1.23.0#Stack)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/goroutine/id](https://github.com/jianghushinian/blog-go-example/tree/main/goroutine/id)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

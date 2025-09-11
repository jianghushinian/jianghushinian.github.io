---
title: 'Go 程序如何实现优雅退出？来看看 K8s 是怎么做的'
date: 2024-08-22 11:28:13
tags:
- Go
- Kubernetes
categories:
- Go
---

在写 Go 程序时，优雅退出是一个老生常谈的问题，也是我们在微服务开发过程中的标配，本文就来介绍下工作中常见的几种优雅退出场景，以及带大家一起来看一下 K8s 中的优雅退出是怎么实现的。

<!-- more -->

### 优雅退出

我们一般可以通过如下方式执行一个 Go 程序：

```bash
$ go build -o main main.go
$ ./main
```

如果要停止正在运行的程序，通常可以这样做：

- 在正在运行程序的终端执行 `Ctrl + C`。
- 在正在运行程序的终端执行 `Ctrl + \`。
- 在终端执行 `kill` 命令，如 `kill pid` 或 `kill -9 pid`。

以上是几种比较常见的终止程序的方式。

这几种操作本身没什么问题，不过它们的默认行为都比较“暴力”。它们会直接强制关闭进程，这就有可能导致出现数据不一致的问题。

比如，一个 HTTP Server 程序正在处理用户下单请求，用户付款操作已经完成，但订单状态还没来得及从「待支付」变更为「已支付」，进程就被杀死退出了。

这种情况肯定是要避免的，于是就有了优雅退出的概念。

所谓的优雅退出，其实就是在关闭进程的时候，不能“暴力”关闭，而是要等待进程中的逻辑（比如一次完整的 HTTP 请求）处理完成后，才关闭进程。

### os/signal 信号机制

其实上面介绍的几种终止程序的方式，都是通过向正在执行的进程发送信号来实现的。

- 在终端执行 `Ctrl + C` 发送的是 `SIGINT` 信号，这个信号表示中断，默认行为就是终止程序。
- 在终端执行 `Ctrl + \` 发送的是 `SIGQUIT` 信号，这个信号其实跟 `SIGINT` 信号差不多，不过它会生成 core 文件，并在终端会打印很多日志内容，不如 `Ctrl + C` 常用。
- `kill` 命令与上面两个快捷键相比，更常用于**结束以后台模式启动的进程**，`kill pid` 发送的是 `SIGTERM` 信号，而 `kill -9 pid` 则发送 `SIGKILL` 信号。

以上几种方式中我们见到了 4 种终止进程的信号：`SIGINT`、`SIGQUIT`、`SIGTERM` 和 `SIGKILL`。

**这其中，前 3 种信号是可以被 Go 进程内部捕获并处理的，而 `SIGKILL` 信号则无法捕获，它会强制杀死进程，没有回旋余地**。

在写 Go 代码时，默认情况下，我们没有关注任何信号，Go 程序会自行处理接收到的信号。对于 `SIGINT`、`SIGTERM`、`SIGQUIT` 这几个信号，Go 的处理方式是直接强制终止进程。

这在 `os/signal` 包的 [官方文档](https://pkg.go.dev/os/signal@go1.22.0#hdr-Default_behavior_of_signals_in_Go_programs) 中有提及：

> The signals SIGKILL and SIGSTOP may not be caught by a program, and therefore cannot be affected by this package.
>
> By default, a synchronous signal is converted into a run-time panic. A SIGHUP, SIGINT, or SIGTERM signal causes the program to exit. A SIGQUIT, SIGILL, SIGTRAP, SIGABRT, SIGSTKFLT, SIGEMT, or SIGSYS signal causes the program to exit with a stack dump. A SIGTSTP, SIGTTIN, or SIGTTOU signal gets the system default behavior (these signals are used by the shell for job control). The SIGPROF signal is handled directly by the Go runtime to implement runtime.CPUProfile. Other signals will be caught but no action will be taken.

译文如下：

> SIGKILL 和 SIGSTOP 两个信号可能不会被程序捕获，因此不会受到此包的影响。
>
> 默认情况下，一个同步信号会被转换为运行时恐慌（panic）。在收到 SIGHUP、SIGINT 或 SIGTERM 信号时，程序将退出。收到 SIGQUIT、SIGILL、SIGTRAP、SIGABRT、SIGSTKFLT、SIGEMT 或 SIGSYS 信号时，程序会在退出时生成堆栈转储（stack dump）。SIGTSTP、SIGTTIN 或 SIGTTOU 信号将按照系统的默认行为处理（这些信号通常由 shell 用于作业控制）。SIGPROF 信号由 Go 运行时直接处理，用于实现 runtime.CPUProfile。其他信号将被捕获但不会采取任何行动。

从这段描述中，我们可以发现，**Go 程序在收到 `SIGINT` 和 `SIGTERM` 两种信号时，程序会直接退出，在收到 `SIGQUIT` 信号时，程序退出并生成 `stack dump`，即退出后控制台打印的那些日志。**

我们可以写一个简单的小程序，来实验一下 `Ctrl + C` 终止 Go 程序的效果。

示例代码如下：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fmt.Println("main enter")

	time.Sleep(time.Second)

	fmt.Println("main exit")
}
```

按照如下方式执行示例程序：

```bash
$ go build -o main main.go && ./main
main enter
^C
$ echo $?
130
```

这里先启动 Go 程序，然后在 Go 程序执行到 `time.Sleep` 的时，按下 `Ctrl + C`，程序会立即终止。

并且通过 `echo $?` 命令可以看到程序退出码为 `130`，表示异常退出，程序正常退出码通常为 `0`。

> NOTE: 
> 这里之所以使用 `go build` 命令先将 Go 程序编译成二进制文件然后再执行，而不是直接使用 `go run` 命令执行程序。是因为不管程序执行结果如何，`go run` 命令返回的程序退出状态码始终为 `1`。只有先将 Go 程序编译成二进制文件以后，再执行二进制文件才能获得（可以使用 `echo $?` 命令）正常的进程退出码。

**如果我们在 Go 代码中自行处理收到的 `Ctrl + C` 传来的信号 `SIGINT`，我们就能够控制程序的退出行为，这也是实现优雅退出的机会所在。**

现在我们就来一起学习下 Go 为我们提供的信号处理包 `os/signal`。

Go 为我们提供了 `os/signal` 内置包用来处理信号，`os/signal` 包提供了如下 [6 个函数](https://pkg.go.dev/os/signal@go1.22.0#pkg-index) 供我们使用：

```go
// 忽略一个或多个指定的信号
func Ignore(sig ...os.Signal)
// 判断指定的信号是否被忽略了
func Ignored(sig os.Signal) bool
// 注册需要关注的某些信号，信号会被传递给函数的第一个参数（channel 类型的参数 c）
func Notify(c chan<- os.Signal, sig ...os.Signal)
// 带有 Context 版本的 Notify
func NotifyContext(parent context.Context, signals ...os.Signal) (ctx context.Context, stop context.CancelFunc)
// 取消关注指定的信号（之前通过调用 Notify 所关注的信号）
func Reset(sig ...os.Signal)
// 停止向 channel 发送所有关注的信号
func Stop(c chan<- os.Signal)
```

**此包中的函数允许程序更改 Go 程序处理信号的默认方式。**

这里我们最需要关注的就是 `Notify` 函数，它可以用来注册我们需要关注的某些信号，这会禁用给定信号的默认行为，转而通过一个或多个已注册的通道（`channel`）传送它们。

我们写一个代码示例程序来看一下 `os/signal` 如何使用：

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	fmt.Println("main enter")

	quit := make(chan os.Signal, 1)
	// 注册需要关注的信号：SIGINT、SIGTERM、SIGQUIT
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	// 阻塞当前 goroutine 等待信号
	sig := <-quit
	fmt.Printf("received signal: %d-%s\n", sig, sig)

	fmt.Println("main exit")
}
```

如你所见，`os/signal` 包使用起来非常简单。

首先我们定义了一个名为 `quit` 的 `channel`，类型为 `os.Signal`，长度为 `1`，用来接收关注的信号。

调用 `signal.Notify` 函数对我们要关注的信号进行注册。

然后调用 `sig := <-quit` 将会阻塞当前 `main` 函数所在的主 goroutine，直到进程接收到 `SIGINT`、`SIGTERM`、`SIGQUIT` 中的任意一个信号。

当我们关注了这几个信号以后，Go 不会自行处理这几个信号，需要我们自己来处理。

程序中打印了几条日志用来观察效果。

这里需要特别注意的是：**我们通过 `signal.Notify(c chan<- os.Signal, sig ...os.Signal)` 函数注册所关注的信号，`signal` 包在收到对应信号时，会向 `c` 这个 `channel` 发送信号，但是发送信号时不会阻塞。也就是说，如果 `signal` 包发送信号到 `c` 时，由于 `c` 满了而导致阻塞，`signal` 包会直接丢弃信号。**

在 `signal.Notify` 函数签名上方的注释中有详细说明：

> https://github.com/golang/go/blob/release-branch.go1.22/src/os/signal/signal.go#L121

```go
// Notify causes package signal to relay incoming signals to c.
// If no signals are provided, all incoming signals will be relayed to c.
// Otherwise, just the provided signals will.
//
// Package signal will not block sending to c: the caller must ensure
// that c has sufficient buffer space to keep up with the expected
// signal rate. For a channel used for notification of just one signal value,
// a buffer of size 1 is sufficient.
//
// It is allowed to call Notify multiple times with the same channel:
// each call expands the set of signals sent to that channel.
// The only way to remove signals from the set is to call Stop.
//
// It is allowed to call Notify multiple times with different channels
// and the same signals: each channel receives copies of incoming
// signals independently.
func Notify(c chan<- os.Signal, sig ...os.Signal) {
    ...
}
```

译文如下：

> Notify 会让 signal 包将收到的信号转发到通道 c 中。如果没有提供具体的信号类型，则所有收到的信号都会被转发到 c。否则，只会将指定的信号转发到 c。
signal 包向 c 发送信号时不会阻塞：调用者必须确保 c 具有足够的缓冲区空间以应对预期的信号速率。如果一个通道仅用于通知单个信号值，那么一个大小为 1 的缓冲区就足够了。
允许使用同一个通道多次调用 Notify：每次调用都会扩展发送到该通道的信号集。要移除信号集中的信号，唯一的方法是调用 Stop。
允许使用不同的通道和相同的信号多次调用 Notify：每个通道都会独立接收传入信号的副本。

在 `signal.Notify` 函数内部通过 `go watchSignalLoop()` 方式启动了一个新的 goroutine，用来监控信号：

```go
var (
	// watchSignalLoopOnce guards calling the conditionally
	// initialized watchSignalLoop. If watchSignalLoop is non-nil,
	// it will be run in a goroutine lazily once Notify is invoked.
	// See Issue 21576.
	watchSignalLoopOnce sync.Once
	watchSignalLoop     func()
)

...

func Notify(c chan<- os.Signal, sig ...os.Signal) {
	...
	add := func(n int) {
		if n < 0 {
			return
		}
		if !h.want(n) {
			h.set(n)
			if handlers.ref[n] == 0 {
				enableSignal(n)

				// The runtime requires that we enable a
				// signal before starting the watcher.
				watchSignalLoopOnce.Do(func() {
					if watchSignalLoop != nil {
						// 监控信号循环
						go watchSignalLoop()
					}
				})
			}
			handlers.ref[n]++
		}
	}
...
}
```

可以发现 `watchSignalLoop` 函数只会执行一次，并且采用 goroutine 的方式执行。

`watchSignalLoop` 函数的定义可以在 `os/signal/signal_unix.go` 中找到：

> https://github.com/golang/go/blob/release-branch.go1.22/src/os/signal/signal_unix.go

```go
func loop() {
	for {
		process(syscall.Signal(signal_recv()))
	}
}

func init() {
	watchSignalLoop = loop
}
```

可以看到，这里开启了一个无限循环，来执行 `process` 函数：

> https://github.com/golang/go/blob/release-branch.go1.22/src/os/signal/signal.go#L232

```go
func process(sig os.Signal) {
	n := signum(sig)
	if n < 0 {
		return
	}

	handlers.Lock()
	defer handlers.Unlock()

	for c, h := range handlers.m {
		if h.want(n) {
			// send but do not block for it
			select {
			case c <- sig:
			default: // 当向 c 发送信号遇到阻塞时，default 逻辑直接丢弃了 sig 信号，没做任何处理
			}
		}
	}

	// Avoid the race mentioned in Stop.
	for _, d := range handlers.stopping {
		if d.h.want(n) {
			select {
			case d.c <- sig:
			default:
			}
		}
	}
}
```

这个 `process` 函数就是 `os/signal` 包向我们注册的 `channel` 发送信号的核心逻辑。

`os/signal` 包在收到我们使用 `signal.Notify` 注册的信号时，会通过 `c <- sig` 向通道 `c` 发送信号。如果向 `c` 发送信号遇到阻塞，`default` 逻辑会直接丢弃 `sig` 信号，不做任何处理。这也就是为什么我们在创建 `quit := make(chan os.Signal, 1)` 时一定要给 `channel` 分配至少 `1` 个缓冲区。

我们可以尝试执行这个示例程序，得到如下输出：

```bash
$ go build -o main main.go && ./main
main enter
^Creceived signal: 2-interrupt
main exit
$ echo $?
0
```

首先使用 `go build -o main main.go && ./main` 命令编译并执行程序。

然后程序会打印 `main enter` 日志并阻塞在那里。

此时我们按下 `Ctrl + C`，控制台会打印日志 `^Creceived signal: 2-interrupt`，然后输出 `main exit` 并退出。

这里第二行日志开头的 `^C` 就表示我们按下了 `Ctrl + C`，收到的信号值为为 `2`，字符串表示形式为 `interrupt`。

程序退出码为 `0`，因为信号被我们捕获并处理，然后程序正常退出，我们改变了 Go 程序对 `SIGINT` 信号处理的默认行为。

其实每一个信号在 `os/signal` 包中都被定义为一个常量：

```go
// A Signal is a number describing a process signal.
// It implements the os.Signal interface.
type Signal int

// Signals
const (
	SIGINT    = Signal(0x2)
	SIGQUIT   = Signal(0x3)
	SIGTERM   = Signal(0xf)
	...
)
```

对应的字符串表示形式为：

```go

// Signal table
var signals = [...]string{
	2:  "interrupt",
	3:  "quit",
	15: "terminated",
	...
}
```

> NOTE:
> 示例程序中，我们是在 `main` 函数中调用 `signal.Notify` 注册关注的信号，即在主的 goroutine 中调用。其实将其放在子 goroutine 中调用也是可以的，并不会影响程序效果，你可以自行尝试。

现在我们已经知道了在 Go 中如何使用 `os/signal` 来接收并处理进程退出信号，那么接下来要关注的就是在进程退出前，如何保证主逻辑操作完成，以实现 Go 程序的优雅退出。

### net/http 的优雅退出

讲解完了前置知识，终于可以进入讲解优雅退出的环节了。

首先我们就以一个 HTTP Server 为例，讲解下在 Go 程序中如何实现优雅退出。

#### HTTP Server 示例程序

这是一个简单的 HTTP Server 示例程序：

```go
package main

import (
	"log"
	"net/http"
	"time"
)

func main() {
	srv := &http.Server{
		Addr: ":8000",
	}

	http.HandleFunc("/sleep", func(w http.ResponseWriter, r *http.Request) {
		duration, err := time.ParseDuration(r.FormValue("duration"))
		if err != nil {
			http.Error(w, err.Error(), 400)
			return
		}
		time.Sleep(duration)
		_, _ = w.Write([]byte("Hello World!"))
	})

	if err := srv.ListenAndServe(); err != nil {
		// Error starting or closing listener:
		log.Fatalf("HTTP server ListenAndServe: %v", err)
	}
	log.Println("Stopped serving new connections")
}
```

> NOTE:
> 注意示例程序中在处理 `srv.ListenAndServe()` 返回的错误时，使用了 `log.Fatalf` 来打印日志并退出程序，这会调用 `os.Exit(1)`，如果函数中有 `defer` 语句则不会被执行，所以 `log.Fatalf` 仅建议在测试程序中使用，生产环境中谨慎使用。

示例中的 HTTP Server 监听了 `8000` 端口，并提供一个 `/sleep` 接口，根据用户传入的 `duration` 参数 sleep 对应的时间，然后返回响应。

执行示例程序：

```bash
$ go build -o main main.go && ./main
```

然后新打开另外一个终端访问这个 HTTP Server：

```bash
$ curl "http://localhost:8000/sleep?duration=0s"
Hello World!
```

传递 `duration=0s` 参数表示不进行 sleep，我们立即得到了正确的响应。

现在我们增加一点 sleep 时间进行请求：

```bash
$ curl "http://localhost:8000/sleep?duration=5s"
```

在 `5s` 以内回到运行 HTTP Server 的终端，并用 `Ctrl + C` 终止程序：

```bash
$ go build -o main main.go && ./main
^C
```

这次我们的客户端请求没有得到正确的响应：

```bash
$ curl "http://localhost:8000/sleep?duration=5s"
curl: (52) Empty reply from server
```

这就是没有实现优雅退出所带来的后果。

当一个客户端请求正在进行中，此时终止 HTTP Server 进程，请求还没有来得及完成，连接就被断开了，客户端无法得到正确的响应结果。

为了改变这一局面，就需要进行优雅退出操作。

#### HTTP Server 优雅退出

我们来分析下一个 HTTP Server 要进行优雅退出，需要做哪些事情：

1. 首先，我们要关闭 HTTP Server 监听的端口，即通过 `net.Listen` 所开启的 `Listener`，以免新的请求进来。
2. 接着，我们要关闭所有空闲的 HTTP 连接。
3. 然后，我们还需要等待所有正在处理请求的 HTTP 连接变为空闲状态之后关闭它们。这里应该可以进行无限期等待，也可以设置一个超时时间，超过一定时间后强制断开连接，以免程序永远无法退出。
4. 最后，正常退出进程。

幸运的是针对以上 HTTP Server 的优雅退出流程，`net/http` 包已经帮我们实现好了。

在 Go 1.8 版本之前，我们需要自己实现以上流程，或者有一些流行的第三方包也能帮我们做到。而从 Go 1.8 版本开始，`net/http` 包自身为我们提供了 `http.Server.Shutdown` 方法可以实现优雅退出的完整流程。

可以在 Go 仓库 [issues/4674](https://github.com/golang/go/issues/4674) 中看到对 `net/http` 包加入优雅退出功能的讨论，这个问题最早在 2013 年就被提出了，不过却从 Go 1.1 版本拖到了 Go 1.8 版本才得以支持。

我们可以在 [net/http 文档](https://pkg.go.dev/net/http@go1.22.0#Server.Shutdown) 中找到关于 `http.Server.Shutdown` 方法的说明：

> Shutdown gracefully shuts down the server without interrupting any active connections. Shutdown works by first closing all open listeners, then closing all idle connections, and then waiting indefinitely for connections to return to idle and then shut down. If the provided context expires before the shutdown is complete, Shutdown returns the context's error, otherwise it returns any error returned from closing the Server's underlying Listener(s).

译文如下：

> Shutdown 会优雅地关闭服务器，而不会中断任何活动的连接。它的工作原理是先关闭所有已打开的监听器（listeners），然后关闭所有空闲的连接，并无限期地等待所有连接变为空闲状态后再关闭服务器。如果在关闭完成之前，传入的上下文（context）过期，Shutdown 会返回上下文的错误，否则它将返回关闭服务器底层监听器时所产生的任何错误。

现在，结合前文介绍的 `os/signal` 包以及 `net/http` 包提供的 `http.Server.Shutdown` 方法，我们可以写出如下优雅退出 HTTP Server 代码：

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	srv := &http.Server{
		Addr: ":8000",
	}

	http.HandleFunc("/sleep", func(w http.ResponseWriter, r *http.Request) {
		duration, err := time.ParseDuration(r.FormValue("duration"))
		if err != nil {
			http.Error(w, err.Error(), 400)
			return
		}
		time.Sleep(duration)
		_, _ = w.Write([]byte("Hello World!"))
	})

	go func() {
		quit := make(chan os.Signal, 1)
		signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
		<-quit
		log.Println("Shutdown Server...")

		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()

		// We received an SIGINT/SIGTERM/SIGQUIT signal, shut down.
		if err := srv.Shutdown(ctx); err != nil {
			// Error from closing listeners, or context timeout:
			log.Printf("HTTP server Shutdown: %v", err)
		}
		log.Println("HTTP server graceful shutdown completed")
	}()

	if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
		// Error starting or closing listener:
		log.Fatalf("HTTP server ListenAndServe: %v", err)
	}
	log.Println("Stopped serving new connections")
}
```

示例中，为了让主 goroutine 不被阻塞，我们开启了一个新的 goroutine 来支持优雅退出。

`quit` 用来接收关注的信号，我们关注了 `SIGINT`、`SIGTERM`、`SIGQUIT` 这 3 个程序退出信号。

`<-quit` 收到退出信号以后，程序将进入优雅退出环节。

调用 `srv.Shutdown(ctx)` 进行优雅退出时，传递了一个 `10s` 超时的 `Context`，这是为了超过一定时间后强制退出，以免程序无限期等待下去，永远无法退出。

并且我们修改了调用 `srv.ListenAndServe()` 时的错误处理判断代码，因为调用 `srv.Shutdown(ctx)` 后，`srv.ListenAndServe()` **立即返回 `http.ErrServerClosed` 错误**，这是符合预期的错误，所以我们将其排除在错误处理流程之外。

执行示例程序：

```bash
$ go build -o main main.go && ./main
```

打开新的终端，访问 HTTP Server：

```bash
$ curl "http://localhost:8000/sleep?duration=5s"
```

`5s` 以内回到 HTTP Server 启动终端，按下 `Ctrl + C`：

```bash
$ go build -o main main.go && ./main
^C2024/08/22 09:15:20 Shutdown Server...
2024/08/22 09:15:20 Stopped serving new connections
```

可以发现程序进入了优雅退出流程 `Shutdown Server...`，并最终打印 `Stopped serving new connections` 日志后退出。

遗憾的是，客户端请求并没有接收到成功的响应信息：

```bash
$ curl "http://localhost:8000/sleep?duration=5s"
curl: (52) Empty reply from server
```

看来，我们的优雅退出实现并没有生效。

其实仔细观察你会发现，我们的程序少打印了一行 `HTTP server graceful shutdown completed` 日志，说明实现优雅退出的 goroutine 并没有执行完成。

根据 [net/http 文档](https://pkg.go.dev/net/http@go1.22.0#Server.Shutdown) 的描述：

> When Shutdown is called, Serve, ListenAndServe, and ListenAndServeTLS immediately return ErrServerClosed. Make sure the program doesn't exit and waits instead for Shutdown to return.

即当调用 `srv.Shutdown(ctx)` 进入优雅退出流程后，`Serve`、`ListenAndServe` 和 `ListenAndServeTLS` 这三个方法会**立即返回 `ErrServerClosed` 错误，我们要确保程序没有退出，而是等待 `Shutdown` 方法执行完成并返回**。

因为进入优雅退出流程后，`srv.ListenAndServe()` 会立即返回，主 goroutine 会马上执行最后一行代码 `log.Println("Stopped serving new connections")` 并退出。

此时子 goroutine 中运行的优雅退出逻辑 `srv.Shutdown(ctx)` 还没来得及处理完成，就跟随主 goroutine 一同退出了。

这是一个有坑的实现。

**我们必须保证程序主 goroutine 等待 `Shutdown` 方法执行完成并返回后才退出。**

修改后的代码如下所示：

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	srv := &http.Server{
		Addr: ":8000",
	}

	http.HandleFunc("/sleep", func(w http.ResponseWriter, r *http.Request) {
		duration, err := time.ParseDuration(r.FormValue("duration"))
		if err != nil {
			http.Error(w, err.Error(), 400)
			return
		}

		time.Sleep(duration)
		_, _ = w.Write([]byte("Welcome HTTP Server"))
	})

	go func() {
		if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
			// Error starting or closing listener:
			log.Fatalf("HTTP server ListenAndServe: %v", err)
		}
		log.Println("Stopped serving new connections")
	}()

	// 可以注册一些 hook 函数，比如从注册中心下线逻辑
	srv.RegisterOnShutdown(func() {
		log.Println("Register Shutdown 1")
	})
	srv.RegisterOnShutdown(func() {
		log.Println("Register Shutdown 2")
	})

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	<-quit
	log.Println("Shutdown Server...")

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// We received an SIGINT/SIGTERM/SIGQUIT signal, shut down.
	if err := srv.Shutdown(ctx); err != nil {
		// Error from closing listeners, or context timeout:
		log.Printf("HTTP server Shutdown: %v", err)
	}
	log.Println("HTTP server graceful shutdown completed")
}
```

这一次我们将优雅退出逻辑 `srv.Shutdown(ctx)` 和服务监听逻辑 `srv.ListenAndServe()` 代码位置进行了互换。

将 `srv.Shutdown(ctx)` 放在主 goroutine 中，而 `srv.ListenAndServe()` 放在了子 goroutine 中。这样的目的显而易见，主 goroutine 只有等待 `srv.Shutdown(ctx)` 执行完成才会退出，所以也就保证了优雅退出流程能过执行完成。

此外，我还顺便使用 `srv.RegisterOnShutdown()` 注册了两个函数到优雅退出流程中，`srv.Shutdown(ctx)` 内部会执行这里注册的函数。所以这里可以注册一些带有清理功能的函数，比如从注册中心下线逻辑等。

现在再次执行示例程序：

```bash
$ go build -o main main.go && ./main
^C2024/08/22 09:16:21 Shutdown Server...
2024/08/22 09:16:21 Stopped serving new connections
2024/08/22 09:16:21 Register Shutdown 1
2024/08/22 09:16:21 Register Shutdown 2
2024/08/22 09:16:24 HTTP server graceful shutdown completed
```

```bash
$ curl "http://localhost:8000/sleep?duration=5s"
Welcome HTTP Server
```

根据这两段日志来看，这一次的优雅退出流程一切正常。

并且可以观察到，我们使用 `srv.RegisterOnShutdown` 注册的 2 个 hook 函数是按照注册顺序依次执行的。

我们还可以测试下超时退出的逻辑，先启动 HTTP Server，然后使用 `curl` 命令请求服务时设置一个 `20s` 超时，这样在通过 `Ctrl + C` 进行优雅退出操作时，`srv.Shutdown(ctx)` 就会因为等待超过 `10s` 没有处理完请求而强制退出。

执行日志如下：

```bash
$ go build -o main main.go && ./main
^C2024/08/22 09:17:09 Shutdown Server...
2024/08/22 09:17:09 Stopped serving new connections
2024/08/22 09:17:09 Register Shutdown 1
2024/08/22 09:17:09 Register Shutdown 2
2024/08/22 09:17:19 HTTP server Shutdown: context deadline exceeded
2024/08/22 09:17:19 HTTP server graceful shutdown completed
```

```bash
$ curl "http://localhost:8000/sleep?duration=20s"
curl: (52) Empty reply from server
```

日志输出结果符合预期。

> NOTE:
> 我们当前示例的实现方式有些情况下存在一个小问题，当 `srv.ListenAndServe()` 返回后，如果子 goroutine 出现了 `panic`，由于我们没有使用 `recover` 语句捕获 `panic`，则 `main` 函数中的 `defer` 语句不会执行，这一点在生产环境下你要小心。
> 其实 `net/http` 包提供了另一种实现优雅退出的 [示例代码](https://pkg.go.dev/net/http@go1.22.0#example-Server.Shutdown)，示例中还是将 `srv.ListenAndServe()` 放在主 goroutine 中，`srv.Shutdown(ctx)` 放在子 goroutine 中，利用一个新的 `channel` 阻塞主 goroutine 的方式来实现。感兴趣的读者可以点击进去学习。

#### HTTP Handler 中有 goroutine 的情况

我们在 HTTP Handler 函数中加上一段异步代码，修改后的程序如下：

```go
http.HandleFunc("/sleep", func(w http.ResponseWriter, r *http.Request) {
	duration, err := time.ParseDuration(r.FormValue("duration"))
	if err != nil {
		http.Error(w, err.Error(), 400)
		return
	}

	time.Sleep(duration)

	// 模拟需要异步执行的代码，比如注册接口异步发送邮件、发送 Kafka 消息等
	go func() {
		log.Println("Goroutine enter")
		time.Sleep(time.Second * 5)
		log.Println("Goroutine exit")
	}()

	_, _ = w.Write([]byte("Welcome HTTP Server"))
})
```

这里新启动了一个 goroutine 模拟需要异步执行的代码，比如注册接口异步发送邮件、发送 Kafka 消息等。这在实际工作中非常常见。

执行示例程序，再次测试优雅退出流程：

```bash
$ go build -o main main.go && ./main
^C2024/08/22 09:18:53 Shutdown Server...
2024/08/22 09:18:53 Stopped serving new connections
2024/08/22 09:18:53 Register Shutdown 1
2024/08/22 09:18:53 Register Shutdown 2
2024/08/22 09:18:56 Goroutine enter
2024/08/22 09:18:56 HTTP server graceful shutdown completed
```

```bash
$ curl "http://localhost:8000/sleep?duration=5s" 
Welcome HTTP Server
```

客户端请求被正确处理了，但是根据日志输出可以发现，处理函数 Handler 中的子 goroutine 并没有正常执行完成就退出了，`Goroutine enter` 日志有被打印，`Goroutine exit` 日志并没有被打印。

出现这种情况的原因，同样是因为主 goroutine 已经退出，子 goroutine 还没来得及处理完成，就跟随主 goroutine 一同退出了。

这会导致数据不一致问题。

为了解决这一问题，我们对示例程序做如下修改：

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

type Service struct {
	wg sync.WaitGroup
}

func (s *Service) FakeSendEmail() {
	s.wg.Add(1)

	go func() {
		defer s.wg.Done()
		defer func() {
			if err := recover(); err != nil {
				log.Printf("Recovered panic: %v\n", err)
			}
		}()

		log.Println("Goroutine enter")
		time.Sleep(time.Second * 5)
		log.Println("Goroutine exit")
	}()
}

func (s *Service) GracefulStop(ctx context.Context) {
	log.Println("Waiting for service to finish")
	quit := make(chan struct{})
	go func() {
		s.wg.Wait()
		close(quit)
	}()
	select {
	case <-ctx.Done():
		log.Println("context was marked as done earlier, than user service has stopped")
	case <-quit:
		log.Println("Service finished")
	}
}

func (s *Service) Handler(w http.ResponseWriter, r *http.Request) {
	duration, err := time.ParseDuration(r.FormValue("duration"))
	if err != nil {
		http.Error(w, err.Error(), 400)
		return
	}

	time.Sleep(duration)

	// 模拟需要异步执行的代码，比如注册接口异步发送邮件、发送 Kafka 消息等
	s.FakeSendEmail()

	_, _ = w.Write([]byte("Welcome HTTP Server"))
}

func main() {
	srv := &http.Server{
		Addr: ":8000",
	}

	svc := &Service{}
	http.HandleFunc("/sleep", svc.Handler)

	go func() {
		if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
			// Error starting or closing listener:
			log.Fatalf("HTTP server ListenAndServe: %v", err)
		}
		log.Println("Stopped serving new connections")
	}()

	// 错误写法
	// srv.RegisterOnShutdown(func() {
	// 	svc.GracefulStop(ctx)
	// })

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	<-quit
	log.Println("Shutdown Server...")

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// We received an SIGINT/SIGTERM/SIGQUIT signal, shut down.
	if err := srv.Shutdown(ctx); err != nil {
		// Error from closing listeners, or context timeout:
		log.Printf("HTTP server Shutdown: %v", err)
	}

	// 优雅退出 service
	svc.GracefulStop(ctx)
	log.Println("HTTP server graceful shutdown completed")
}
```

这里对示例程序进行了重构，定义一个 `Service` 结构体用来承载业务逻辑，它包含一个 `sync.WaitGroup` 对象用来控制异步程序执行。

Handler 中的异步代码被移到了 `s.FakeSendEmail()` 方法中，`s.FakeSendEmail()` 方法内部会启动一个新的 goroutine 模拟异步发送邮件。

并且我们还为 `Service` 提供了一个优雅退出方法 `GracefulStop`，`GracefulStop` 使用 `s.wg.Wait()` 方式，来等待它关联的所有已开启的 goroutine 执行完成再退出。

在 `main` 函数中，执行 `srv.Shutdown(ctx)` 完成后，再调用 `svc.GracefulStop(ctx)` 实现优雅退出。

现在，执行示例程序，再次测试优雅退出流程：

```bash
$ go build -o main main.go && ./main
^C2024/08/22 09:20:03 Shutdown Server...
2024/08/22 09:20:03 Stopped serving new connections
2024/08/22 09:20:06 Goroutine enter
2024/08/22 09:20:06 Waiting for service to finish
2024/08/22 09:20:11 Goroutine exit
2024/08/22 09:20:11 Service finished
2024/08/22 09:20:11 HTTP server graceful shutdown completed
```

```bash
$ curl "http://localhost:8000/sleep?duration=5s"
Welcome HTTP Server
```

这一次 `Goroutine exit` 日志被正确打印出来，说明优雅退出生效了。

这也提醒我们，在开发过程中，不要随意创建一个不知道何时退出的 goroutine，我们要主动关注 goroutine 的生命周期，以免程序失控。

细心的读者应该已经发现，我在示例程序中注释了一段错误写法的代码：

```go
// 错误写法
// srv.RegisterOnShutdown(func() {
// 	svc.GracefulStop(ctx)
// })
```

对于 `Service` 优雅退出子 goroutine 的场景的确不适用于将其注册到 `srv.RegisterOnShutdown` 中。

这是因为 `svc.Handler` 中的代码执行到 `time.Sleep(duration)` 时，程序还没开始执行 `svc.FakeSendEmail()`，这时如果我们按 `Ctrl + C` 退出程序，`srv.Shutdown(ctx)` 内部会先执行 `srv.RegisterOnShutdown` 注册的函数，`svc.GracefulStop` 会立即执行完成并退出，之后等待几秒，`svc.Handler` 中的逻辑才会走到 `svc.FakeSendEmail()`，此时就已经无法实现优雅退出 goroutine 了。

至此，HTTP Server 中基本的常见优雅退出场景及方案我们就介绍完了，接下来我再带你一起深入了解一下 `Shutdown` 的源码是如何实现的。

#### Shutdown 源码

`Shutdown` 方法源码如下：

> https://github.com/golang/go/blob/release-branch.go1.22/src/net/http/server.go#L2990

```go
// Shutdown gracefully shuts down the server without interrupting any
// active connections. Shutdown works by first closing all open
// listeners, then closing all idle connections, and then waiting
// indefinitely for connections to return to idle and then shut down.
// If the provided context expires before the shutdown is complete,
// Shutdown returns the context's error, otherwise it returns any
// error returned from closing the [Server]'s underlying Listener(s).
//
// When Shutdown is called, [Serve], [ListenAndServe], and
// [ListenAndServeTLS] immediately return [ErrServerClosed]. Make sure the
// program doesn't exit and waits instead for Shutdown to return.
//
// Shutdown does not attempt to close nor wait for hijacked
// connections such as WebSockets. The caller of Shutdown should
// separately notify such long-lived connections of shutdown and wait
// for them to close, if desired. See [Server.RegisterOnShutdown] for a way to
// register shutdown notification functions.
//
// Once Shutdown has been called on a server, it may not be reused;
// future calls to methods such as Serve will return ErrServerClosed.
func (srv *Server) Shutdown(ctx context.Context) error {
	srv.inShutdown.Store(true)

	srv.mu.Lock()
	lnerr := srv.closeListenersLocked()
	for _, f := range srv.onShutdown {
		go f()
	}
	srv.mu.Unlock()
	srv.listenerGroup.Wait()

	pollIntervalBase := time.Millisecond
	nextPollInterval := func() time.Duration {
		// Add 10% jitter.
		interval := pollIntervalBase + time.Duration(rand.Intn(int(pollIntervalBase/10)))
		// Double and clamp for next time.
		pollIntervalBase *= 2
		if pollIntervalBase > shutdownPollIntervalMax {
			pollIntervalBase = shutdownPollIntervalMax
		}
		return interval
	}

	timer := time.NewTimer(nextPollInterval())
	defer timer.Stop()
	for {
		if srv.closeIdleConns() {
			return lnerr
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-timer.C:
			timer.Reset(nextPollInterval())
		}
	}
}
```

首先 `Shutdown` 方法注释写的非常清晰：

> Shutdown 会优雅地关闭服务器，而不会中断任何活动的连接。它的工作原理是先关闭所有已打开的监听器（listeners），然后关闭所有空闲的连接，并无限期地等待所有连接变为空闲状态后再关闭服务器。如果在关闭完成之前，传入的上下文（context）过期，Shutdown 会返回上下文的错误，否则它将返回关闭服务器底层监听器时所产生的任何错误。
>
> 当调用 Shutdown 时，[Serve]、[ListenAndServe] 和 [ListenAndServeTLS] 会立即返回 [ErrServerClosed] 错误。请确保程序不会直接退出，而是等待 Shutdown 返回后再退出。
>
> Shutdown 不会尝试关闭或等待被劫持的连接（例如 WebSocket）。Shutdown 的调用者应单独通知这些长时间存在的连接关于关闭的信息，并根据需要等待它们关闭。可以参考 [Server.RegisterOnShutdown] 来注册关闭通知函数。
>
> 一旦在服务器上调用了 Shutdown，它将无法再次使用；之后对 Serve 等方法的调用将返回 ErrServerClosed 错误。

通过这段注释，我们就能对 `Shutdown` 方法执行流程有个大概理解。

接着我们来从上到下依次分析下 `Shutdown` 源码。

第一行代码如下：

```go
srv.inShutdown.Store(true)
```

`Shutdown` 首先将 `inShutdown` 标记为 `true`，`inShutdown` 是 `atomic.Bool` 类型，它用来标记服务器是否正在关闭。

这里使用了 `atomic` 来保证操作的原子性，以免其他方法读取到错误的 `inShutdown` 标志位，发生错误。避免 HTTP Server 进程已经开始处理结束逻辑，还会有新的请求进入到 `srv.Serve` 方法。

接着 `Shutdown` 会关闭监听的端口：

```go
srv.mu.Lock()
lnerr := srv.closeListenersLocked()
for _, f := range srv.onShutdown {
    go f()
}
srv.mu.Unlock()
```

代码中的 `srv.closeListenersLocked()` 就是在关闭所有的监听器（`listeners`）。

方法定义如下：

```go
func (s *Server) closeListenersLocked() error {
	var err error
	for ln := range s.listeners {
		if cerr := (*ln).Close(); cerr != nil && err == nil {
			err = cerr
		}
	}
	return err
}
```

这一操作，就对应了在前文中讲解的 HTTP Server 优雅退出流程中的第 1 步，关闭所有开启的 `net.Listener` 对象。

接下来循环遍历 `srv.onShutdown` 中的函数，并依次启动新的 goroutine 对其进行调用。

`onShutdown` 是 `[]func()` 类型，其切片内容正是在我们调用 `srv.RegisterOnShutdown` 的时候注册进来的。

`srv.RegisterOnShutdown` 定义如下：

```go
// RegisterOnShutdown registers a function to call on [Server.Shutdown].
// This can be used to gracefully shutdown connections that have
// undergone ALPN protocol upgrade or that have been hijacked.
// This function should start protocol-specific graceful shutdown,
// but should not wait for shutdown to complete.
func (srv *Server) RegisterOnShutdown(f func()) {
	srv.mu.Lock()
	srv.onShutdown = append(srv.onShutdown, f)
	srv.mu.Unlock()
}
```

这是我们在前文中的使用示例：

```go
// 可以注册一些 hook 函数，比如从注册中心下线逻辑
srv.RegisterOnShutdown(func() {
    log.Println("Register Shutdown 1")
})
srv.RegisterOnShutdown(func() {
    log.Println("Register Shutdown 2")
})
```

接着代码执行到这一步：

```go
srv.listenerGroup.Wait()
```

根据这个操作的属性名和方法名可以猜到，`listenerGroup` 明显是 `sync.WaitGroup` 类型。

既然有 `Wait()`，那就应该会有 `Add(1)` 操作。在源码中搜索 `listenerGroup.Add(1)` 关键字，可以搜到如下方法：

```go
// trackListener adds or removes a net.Listener to the set of tracked
// listeners.
//
// We store a pointer to interface in the map set, in case the
// net.Listener is not comparable. This is safe because we only call
// trackListener via Serve and can track+defer untrack the same
// pointer to local variable there. We never need to compare a
// Listener from another caller.
//
// It reports whether the server is still up (not Shutdown or Closed).
func (s *Server) trackListener(ln *net.Listener, add bool) bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.listeners == nil {
		s.listeners = make(map[*net.Listener]struct{})
	}
	if add {
		if s.shuttingDown() {
			return false
		}
		s.listeners[ln] = struct{}{}
		s.listenerGroup.Add(1)
	} else {
		delete(s.listeners, ln)
		s.listenerGroup.Done()
	}
	return true
}
```

`trackListener` 用于添加或移除一个 `net.Listener` 到已跟踪的监听器集合中。

这个方法会被 `Serve` 方法调用，而实际上我们执行 `srv.ListenAndServe()` 的方法内部，也是在调用 `Serve` 方法。

`Serve` 方法定义如下：

```go
// Serve accepts incoming connections on the Listener l, creating a
// new service goroutine for each. The service goroutines read requests and
// then call srv.Handler to reply to them.
//
// HTTP/2 support is only enabled if the Listener returns [*tls.Conn]
// connections and they were configured with "h2" in the TLS
// Config.NextProtos.
//
// Serve always returns a non-nil error and closes l.
// After [Server.Shutdown] or [Server.Close], the returned error is [ErrServerClosed].
func (srv *Server) Serve(l net.Listener) error {
	if fn := testHookServerServe; fn != nil {
		fn(srv, l) // call hook with unwrapped listener
	}

	origListener := l
	l = &onceCloseListener{Listener: l}
	defer l.Close()

	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}

	// 将 `net.Listener` 添加到已跟踪的监听器集合中
	// 内部会通过调用 s.shuttingDown() 判断是否正在进行退出操作，如果是，则返回 ErrServerClosed
	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
	// Serve 函数退出时，将 `net.Listener` 从已跟踪的监听器集合中移除
	defer srv.trackListener(&l, false)

	baseCtx := context.Background()
	if srv.BaseContext != nil {
		baseCtx = srv.BaseContext(origListener)
		if baseCtx == nil {
			panic("BaseContext returned a nil context")
		}
	}

	var tempDelay time.Duration // how long to sleep on accept failure

	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, err := l.Accept()
		if err != nil {
			// 每次新的请求进来，先判断当前服务是否已经被标记为正在关闭，如果是，则直接返回 ErrServerClosed
			if srv.shuttingDown() {
				return ErrServerClosed
			}
			if ne, ok := err.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return err
		}
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
		go c.serve(connCtx)
	}
}
```

在 `Serve` 方法内部，我们先重点关注如下代码段：

```go
if !srv.trackListener(&l, true) {
    return ErrServerClosed
}
defer srv.trackListener(&l, false)
```

这说明 `Serve` 在启动的时候会将一个新的监听器（`net.Listener`）加入到 `listeners` 集合中。

`Serve` 函数退出时，会对其进行移除。

并且，`srv.trackListener` 内部又调用了 `s.shuttingDown()` 判断当前服务是否正在进行退出操作，如果是，则返回 `ErrServerClosed`。


```go
// 标记为关闭状态，就不会有请求进来，直接返回错误
if srv.shuttingDown() {
    return ErrServerClosed
}
```

`shuttingDown` 定义如下：

```go
func (s *Server) shuttingDown() bool {
	return s.inShutdown.Load()
}
```

同理，在 `for` 循环中，每次 `rw, err := l.Accept()` 收到新的请求，都会先判断当前服务是否已经被标记为正在关闭，如果是，则直接返回 `ErrServerClosed`。

这里其实就是在跟 `Shutdown` 方法中的 `srv.inShutdown.Store(true)` 进行配合操作。

`Shutdown` 收到优雅退出请求，就将 `inShutdown` 标记为 `true`。此时 `Serve` 方法内部为了不再接收新的请求进来，每次都会调用 `s.shuttingDown()` 进行判断。保证不会再有新的请求进来，导致 `Shutdown` 无法退出。

这跟前文讲解完全吻合，在 `Shutdown` 方法还没执行完成的时候，`Serve` 方法其实已经退出了。也是我们为什么将 `srv.ListenAndServe()` 代码放到子 goroutine 中的原因。

`Shutdown` 接着往下执行：

```go
pollIntervalBase := time.Millisecond
nextPollInterval := func() time.Duration {
    // Add 10% jitter.
    interval := pollIntervalBase + time.Duration(rand.Intn(int(pollIntervalBase/10)))
    // Double and clamp for next time.
    pollIntervalBase *= 2
    if pollIntervalBase > shutdownPollIntervalMax {
        pollIntervalBase = shutdownPollIntervalMax
    }
    return interval
}

timer := time.NewTimer(nextPollInterval())
defer timer.Stop()
for {
    if srv.closeIdleConns() {
        return lnerr
    }
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-timer.C:
        timer.Reset(nextPollInterval())
    }
}
```

这一段逻辑比较多，并且 `nextPollInterval` 函数看起来比较迷惑，不过没关系，我们一点点来分析。

我们把这段代码中的 `nextPollInterval` 函数单独拿出来跑一下，就能大概知道它的意图了：

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	const shutdownPollIntervalMax = 500 * time.Millisecond
	pollIntervalBase := time.Millisecond
	nextPollInterval := func() time.Duration {
		// Add 10% jitter.
		interval := pollIntervalBase + time.Duration(rand.Intn(int(pollIntervalBase/10)))
		// Double and clamp for next time.
		pollIntervalBase *= 2
		if pollIntervalBase > shutdownPollIntervalMax {
			pollIntervalBase = shutdownPollIntervalMax
		}
		return interval
	}

	for i := 0; i < 20; i++ {
		fmt.Println(nextPollInterval())
	}
}
```

执行这段程序，输入结果如下：

```bash
$ go run main.go
1.078014ms
2.007835ms
4.151327ms
8.474296ms
17.487625ms
34.403371ms
64.613106ms
136.696655ms
273.873977ms
516.290814ms
502.815326ms
516.160214ms
523.34143ms
537.808701ms
518.913897ms
526.711692ms
518.421559ms
527.229427ms
526.904891ms
502.738764ms
```

我们可以把随机数再去掉。

把这行代码：

```go
interval := pollIntervalBase + time.Duration(rand.Intn(int(pollIntervalBase/10)))
```

改成这样：

```go
interval := pollIntervalBase + time.Duration(pollIntervalBase/10)
```

重新执行这段程序，输入结果如下：

```bash
$ go run main.go
1.1ms
2.2ms
4.4ms
8.8ms
17.6ms
35.2ms
70.4ms
140.8ms
281.6ms
550ms
550ms
550ms
550ms
550ms
550ms
550ms
550ms
550ms
550ms
550ms
```

根据输出结果，我们可以清晰的看出，`nextPollInterval` 函数执行返回值，开始时按照 2 倍方式增长，最终固定在 `550ms`。

`pollIntervalBase` 最终值等于 `shutdownPollIntervalMax`。

根据公式计算 `interval := pollIntervalBase + time.Duration(pollIntervalBase/10)`，即 `interval = 500 + 500 / 10 = 550ms`，计算结果与输出结果相吻合。

说白了，这段代码写这么复杂，其核心目的就是为 `Shutdown` 方法中等待空闲连接关闭的轮询操作设计一个动态的、带有抖动（jitter）的时间间隔。这种设计确保服务器在执行优雅退出时，能够有效地处理剩余的空闲连接，同时避免不必要的资源浪费。

现在来看 `for` 循环这段代码，就非常好理解了：

```go
timer := time.NewTimer(nextPollInterval())
defer timer.Stop()
for {
	if srv.closeIdleConns() {
		return lnerr
	}
	select {
	case <-ctx.Done():
		return ctx.Err()
	case <-timer.C:
		timer.Reset(nextPollInterval())
	}
}
```

这就是 Go 常用的定时器惯用法。

根据 `nextPollInterval()` 返回值大小，每次定时循环调用 `srv.closeIdleConns()` 方法。

并且这里有一个 case 执行了 `case <-ctx.Done()`，这正是我们调用 `srv.Shutdown(ctx)` 时，用来控制超时时间传递进来的 `Context`。

另外，值得一提的是，在 [Go 1.15](https://github.com/golang/go/blob/release-branch.go1.15/src/net/http/server.go#L2699) 及以前的版本的 `Shutdown` 代码中这段定时器代码并不是这样实现的。

旧版本代码实现如下：

> https://github.com/golang/go/blob/release-branch.go1.15/src/net/http/server.go

```go
var shutdownPollInterval = 500 * time.Millisecond
...

ticker := time.NewTicker(shutdownPollInterval)
defer ticker.Stop()
for {
    if srv.closeIdleConns() && srv.numListeners() == 0 {
        return lnerr
    }
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-ticker.C:
    }
}
```

旧版本代码实现更加简单，并没有使用 `time.NewTimer`，而是使用了 `time.NewTicker`。这样实现的好处是代码简单，逻辑清晰，没花哨的功能。

其实我们在工作中写代码也是一样的道理，先让代码 `run` 起来，后期再考虑优化的问题。`Shutdown` 方法在 Go 1.8 版本被加入，直到 Go 1.16 版本这段代码才发生改变。

旧版本代码使用 `time.NewTicker` 是因为每次定时循环的周期都是固定值，不需要改变。

新版本代码使用 `time.NewTimer` 是为了在每次循环周期中调用 `timer.Reset` 重置间隔时间。

这也是一个值得学习的小技巧。我们在工作中经常会遇到类似的需求：每隔一段时间，执行一次操作。最简单的方式就是使用 `time.Sleep` 来做间隔时长，然后就是 `time.NewTicker` 和 `time.NewTimer` 这两种方式。这 3 种方式其实都能实现每隔一段时间执行一次操作，但它们适用场景又有所不同。

`time.Sleep` 是用阻塞当前 goroutine 的方式来实现的，它需要调度器先唤醒当前 goroutine，然后才能执行后续代码逻辑。

`time.Ticker` 创建了一个底层数据结构定时器 `runtimeTimer`，并且监听 `runtimeTimer` 计时结束后产生的信号。因为 Go 为其进行了优化，所以它的 CPU 消耗比 `time.Sleep` 小很多。

`time.Timer` 底层也是定时器 `runtimeTimer`，只不过我们可以方便的使用 `timer.Reset` 重置间隔时间。

所以这 3 者都有各自适用的场景。

现在我们需要继续跟踪的代码就剩下 `srv.closeIdleConns()` 了，根据方法命名我们也能大概猜测到它的用途就是为了关闭空闲连接。

`closeIdleConns` 方法定义如下：

```go
// closeIdleConns closes all idle connections and reports whether the
// server is quiescent.
func (s *Server) closeIdleConns() bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	quiescent := true
	for c := range s.activeConn {
		st, unixSec := c.getState()
		// Issue 22682: treat StateNew connections as if
		// they're idle if we haven't read the first request's
		// header in over 5 seconds.
		//  这里预留 5s，防止在第一次读取连接头部信息时超过 5s
		if st == StateNew && unixSec < time.Now().Unix()-5 {
			st = StateIdle
		}
		if st != StateIdle || unixSec == 0 {
			// Assume unixSec == 0 means it's a very new
			// connection, without state set yet.
			// // unixSec == 0 代表这个连接是非常新的连接，则标志位被置为 false
			quiescent = false
			continue
		}
		c.rwc.Close()
		delete(s.activeConn, c)
	}
	return quiescent
}
```

这个方法比较核心，所以整个操作做了加锁处理。

使用 `for` 循环遍历所有连接，`activeConn` 是一个集合，类型为 `map[*conn]struct{}`，里面记录了所有存活的连接。

`c.getState()` 能够获取连接的当前状态，对应的还有一个 `setState` 方法能够设置状态，`setState` 方法会在 `Serve` 方法中被调用。这其实就形成闭环了，每次有新的请求进来，都会设置连接状态（`Serve` 会根据当前处理请求的进度，将连接状态设置成 `StateNew`、`StateActive`、`StateIdle`、`StateClosed` 等），而在 `Shutdown` 方法中获取连接状态。

接着，代码中会判断连接中的请求是否已经完成操作（即：是否处于空闲状态 `StateIdle`），如果是，就直接将连接关闭，并从连接集合中移除，否则，跳过此次循环，等待下次循环周期。

这里调用 `c.rwc.Close()` 关闭连接，调用 `delete(s.activeConn, c)` 将当前连接从集合中移除，直到集合为空，表示全部连接已经被关闭释放，循环退出。

`closeIdleConns` 方法最终返回的 `quiescent` 标志位，是用来标记是否所有的连接都已经关闭。如果是，返回 `true`，否则，返回 `false`。

这个方法的逻辑，其实就对应了前文讲解优雅退出流程中的第 2、3 两步。

至此，`Shutdown` 的源码就分析完成了。

`Shutdown` 方法的整个流程也完全是按照我们前文中讲解的优雅退出流程来的。

> NOTE:
> 除了使用 `Shutdown` 进行优雅退出，`net/http` 包还为我们提供了 `Close` 方法用来强制退出，你可以自行尝试。

### Gin 的优雅退出

在工作中我们开发 Go Web 程序时，往往不会直接使用 `net/http` 包，而是引入一个第三方库或框架。其中 Gin 作为 Go 生态中最为流行的 Web 库，我觉得有必要讲解一下在 Gin 中如何进行优雅退出。

不过，学习了 `net/http` 的优雅退出，实际上 Gin 框架的优雅退出是一样的，因为 Gin 只是一个路由库，提供 HTTP Server 能力的还是 `net/http`。

Gin 框架中的优雅退出示例代码如下：

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()

	router.GET("/sleep", func(c *gin.Context) {
		duration, err := time.ParseDuration(c.Query("duration"))
		if err != nil {
			c.String(http.StatusBadRequest, err.Error())
			return
		}

		time.Sleep(duration)
		c.String(http.StatusOK, "Welcome Gin Server")
	})

	srv := &http.Server{
		Addr:    ":8000",
		Handler: router,
	}

	go func() {
		// 服务连接
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			// Error starting or closing listener:
			log.Fatalf("HTTP server ListenAndServe: %v", err)
		}
		log.Println("Stopped serving new connections")
	}()

	// 等待中断信号以优雅地关闭服务器（设置 5 秒的超时时间）
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	<-quit
	log.Println("Shutdown Server...")

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// We received an SIGINT/SIGTERM signal, shut down.
	if err := srv.Shutdown(ctx); err != nil {
		// Error from closing listeners, or context timeout:
		log.Printf("HTTP server Shutdown: %v", err)
	}
	log.Println("HTTP server graceful shutdown completed")
}
```

可以发现，在 Gin 框架中实现优雅退出代码与我们在 `net/http` 包中的实现没什么不同。

只不过我们在实例化 `http.Server` 对象时，将 `*gin.Engine` 作为了 Handler 复制给 `Handler` 属性：

```go
srv := &http.Server{
    Addr:    ":8000",
    Handler: router,
}
```

执行示例程序测试优雅退出，得到如下输出：

```bash
$ go build -o main main.go && ./main
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /sleep                    --> main.main.func1 (3 handlers)
^C2024/08/22 09:23:36 Shutdown Server...
2024/08/22 09:23:36 Stopped serving new connections
[GIN] 2024/08/22 - 09:23:39 | 200 |  5.001282167s |       127.0.0.1 | GET      "/sleep?duration=5s"
2024/08/22 09:23:39 HTTP server graceful shutdown completed
```

```bash
$ curl "http://localhost:8000/sleep?duration=5s"
Welcome Gin Server
```

这里同样在处理请求的过程中，按下 `Ctrl + C`，根据日志可以发现，Gin 示例代码中的优雅退出没有问题。

Gin [框架文档](https://gin-gonic.com/zh-cn/docs/examples/graceful-restart-or-stop/) 也提到了在 Go 1.8 版本之前可以使用如下几个第三方库实现的优雅退出替代方案：

- [manners](https://github.com/braintree/manners)：可以优雅关机的 Go Http 服务器。
- [graceful](https://github.com/tylerstillwater/graceful)：Graceful 是一个 Go 扩展包，可以优雅地关闭 http.Handler 服务器。
- [grace](https://github.com/facebookarchive/grace)：Go 服务器平滑重启和零停机时间部署。

当然我们使用的是 Go 1.8 以上版本，可以不需要这些库。因为 `net/http` 已经提供了原生的优雅退出方案，所以几乎用不到它们，感兴趣的可以自行研究下。

> NOTE:
> 可以参阅 Gin 完整的 [graceful-shutdown](https://github.com/gin-gonic/examples/tree/master/graceful-shutdown) 示例。

### K8s 的优雅退出

现在，我们已经掌握了 Go 中 HTTP Server 程序如何实现优雅退出，是时候看一看 K8s 中提供的一种更为优雅的优雅退出退出方案了😄。

这要从 K8s API Server 启动入口说起：

> https://github.com/kubernetes/kubernetes/blob/release-1.31/cmd/kube-apiserver/apiserver.go

```go
func main() {
	command := app.NewAPIServerCommand()
	code := cli.Run(command)
	os.Exit(code)
}
```

K8s API Server 启动入口代码非常简单，我们可以进入 `app.NewAPIServerCommand()` 查看更多细节：

> https://github.com/kubernetes/kubernetes/blob/release-1.31/cmd/kube-apiserver/app/server.go#L122

```go
// NewAPIServerCommand creates a *cobra.Command object with default parameters
func NewAPIServerCommand() *cobra.Command {
	...
	cmd := &cobra.Command{
		...
		RunE: func(cmd *cobra.Command, args []string) error {
			...
			return Run(cmd.Context(), completedOptions)
		},
		...
	}
	cmd.SetContext(genericapiserver.SetupSignalContext())

	...
	return cmd
}
```

在 `NewAPIServerCommand` 函数中，我们要关注的核心代码只有两行：

一行是 `cmd.SetContext(genericapiserver.SetupSignalContext())`，这是在为 `cmd` 对象设置 `ctx` 属性。

另一行是 `RunE` 属性中最后一行代码 `Run(cmd.Context(), completedOptions)`，这里是启动程序，并使用了 `cmd` 对象的 `ctx` 属性。

很明显，K8s 使用了 Go 语言中流行的 Cobra 命令行框架作为程序的启动框架，Cobra 提供了如下两个方法可以设置和获取 `Context`：

```go
func (c *Command) Context() context.Context {
	return c.ctx
}

func (c *Command) SetContext(ctx context.Context) {
	c.ctx = ctx
}
```

> NOTE:
> 如果你对 Cobra 不太熟悉，可以参考我的另一篇文章[《Go 语言现代命令行框架 Cobra 详解》](https://jianghushinian.cn/2023/05/08/go-modern-command-line-framework-cobra-details/)。

这里的 `ctx` 就是串联起 K8s 实现优雅退出的核心对象。

首先通过 `genericapiserver.SetupSignalContext()` 获取到一个 `context.Context` 对象，根据函数名称可以猜测到它可能跟信号有关。

对于 `Run(cmd.Context(), completedOptions)` 方法的调用，由于嵌套层级比较深，逻辑比较复杂，我就不把整个代码调用链都贴出来讲了。总之，这个启动过程最终可以定位到 [preparedGenericAPIServer.RunWithContext](https://github.com/kubernetes/kubernetes/blob/release-1.31/staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go#L503) 这个方法的执行。在 `RunWithContext` 方法内部的第一行代码 `stopCh := ctx.Done()` 是重点，它拿到了一个控制程序退出时机的 `channel`（这跟我们前文讲解的优雅退出示例中 `quit := make(chan os.Signal, 1)` 变量作用相同），而这个 `ctx` 实际上就是 `genericapiserver.SetupSignalContext()` 的返回值，如果你感兴趣可以详细研究下这个 `stopCh` 的使用过程。

我们直接去分析 `genericapiserver.SetupSignalContext()` 的实现：

> https://github.com/kubernetes/apiserver/blob/release-1.31/pkg/server/signal.go

```go
package server

import (
	"context"
	"os"
	"os/signal"
)

var onlyOneSignalHandler = make(chan struct{})
var shutdownHandler chan os.Signal

// SetupSignalHandler registered for SIGTERM and SIGINT. A stop channel is returned
// which is closed on one of these signals. If a second signal is caught, the program
// is terminated with exit code 1.
// Only one of SetupSignalContext and SetupSignalHandler should be called, and only can
// be called once.
func SetupSignalHandler() <-chan struct{} {
	return SetupSignalContext().Done()
}

// SetupSignalContext is same as SetupSignalHandler, but a context.Context is returned.
// Only one of SetupSignalContext and SetupSignalHandler should be called, and only can
// be called once.
func SetupSignalContext() context.Context {
	close(onlyOneSignalHandler) // panics when called twice

	shutdownHandler = make(chan os.Signal, 2)

	ctx, cancel := context.WithCancel(context.Background())
	signal.Notify(shutdownHandler, shutdownSignals...)
	go func() {
		<-shutdownHandler
		cancel()
		<-shutdownHandler
		os.Exit(1) // second signal. Exit directly.
	}()

	return ctx
}

// RequestShutdown emulates a received event that is considered as shutdown signal (SIGTERM/SIGINT)
// This returns whether a handler was notified
func RequestShutdown() bool {
	if shutdownHandler != nil {
		select {
		case shutdownHandler <- shutdownSignals[0]:
			return true
		default:
		}
	}

	return false
}
```

这里代码不多，但却相当精妙，可以一窥 K8s 设计之优雅。

我们从 `SetupSignalContext` 函数开始分析。

`SetupSignalContext` 函数第一行代码，通过调用 `close(onlyOneSignalHandler)` 来确保在整个程序中只调用一次 `SetupSignalContext` 函数，调用多次则直接 `panic`。这能强制调用方写出正确的代码，避免出现意料之外的情况。

`shutdownHandler` 是一个包含了两个缓冲区的 `channel`，而不像我们定义的 `quit := make(chan os.Signal, 1)` 那样只有一个缓冲区大小。

我们前文讲过，通过 `signal.Notify(c chan<- os.Signal, sig ...os.Signal)` 函数注册所关注的信号后，`signal` 包在给 `c` 发送信号时不会阻塞。因为我们要接收两次退出信号，所以 `shutdownHandler` 缓冲区大小为 `2`。

这也是 `SetupSignalContext` 函数的精髓所在，它实现了收到一次 `SIGINT/SIGTERM` 信号，程序优雅退出，收到两次 `SIGINT/SIGTERM` 信号，程序强制退出的功能。

代码片段如下：

```go
ctx, cancel := context.WithCancel(context.Background())
signal.Notify(shutdownHandler, shutdownSignals...)
go func() {
	<-shutdownHandler
	cancel()
	<-shutdownHandler
	os.Exit(1) // second signal. Exit directly.
}()
```

这里使用一个带有取消功能的 `Context`，当第一次收到信号时，就调用 `cancel()` 取消这个 `ctx`。而这个 `ctx` 会作为函数返回值返给调用方，调用方拿到它，就可以在需要的地方调用 `<-ctx.Done()` 来等待退出信号了。这就是 `preparedGenericAPIServer.RunWithContext` 方法中调用 `stopCh := ctx.Done()` 拿到 `channel`，然后等待 `<-stopCh` 退出信号的逻辑了。

这里用到的 `shutdownSignals` 变量，定义在 `signal_posix.go` 文件中：

> https://github.com/kubernetes/apiserver/blob/release-1.31/pkg/server/signal_posix.go

```go
package server

import (
	"os"
	"syscall"
)

var shutdownSignals = []os.Signal{os.Interrupt, syscall.SIGTERM}
```

`shutdownSignals` 是一个保存了两个信号的切片对象。

`os.Interrupt` 实际上是一个变量，它的值等于 `syscall.SIGINT`。

```go
// The only signal values guaranteed to be present in the os package on all
// systems are os.Interrupt (send the process an interrupt) and os.Kill (force
// the process to exit). On Windows, sending os.Interrupt to a process with
// os.Process.Signal is not implemented; it will return an error instead of
// sending a signal.
var (
	Interrupt Signal = syscall.SIGINT
	Kill      Signal = syscall.SIGKILL
)
```

这里为实现优雅退出，监控了两个信号 `SIGINT` 和 `SIGTERM`，并没有监控 `SIGQUIT` 信号。不过这已经足够用了，根据我的经验，绝大多数情况下我们都会使用 `Ctrl + C` 终止程序，而非使用 `Ctrl + \`。

`SetupSignalHandler` 函数内部调用了 `SetupSignalContext` 函数，它唯一的作用就是直接返回给调用方 `ctx.Done()` 所返回的 `channel`，以此来方便调用方。

`RequestShutdown` 函数可以主动触发退出事件信号（`SIGTERM/SIGINT`），返回值表示是否触发成功。

现在将 K8s 优雅退出方案集成进我们的 `net/http` 优雅退出示例程序中：

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"time"

	genericapiserver "k8s.io/apiserver/pkg/server"
)

func main() {
	srv := &http.Server{
		Addr: ":8000",
	}

	http.HandleFunc("/sleep", func(w http.ResponseWriter, r *http.Request) {
		duration, err := time.ParseDuration(r.FormValue("duration"))
		if err != nil {
			http.Error(w, err.Error(), 400)
			return
		}

		time.Sleep(duration)
		_, _ = w.Write([]byte("Welcome HTTP Server"))
	})

	go func() {
		if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
			log.Fatalf("HTTP server error: %v", err)
		}
		log.Println("Stopped serving new connections")
	}()

	// NOTE: 只需要替换这 3 行代码，Gin 版本同理
	// quit := make(chan os.Signal, 1)
	// signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	// <-quit

	// 可以直接丢弃，context.Context.Done() 返回的就是普通空结构体
	<-genericapiserver.SetupSignalHandler()

	log.Println("Shutdown Server...")

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// We received an SIGINT/SIGTERM signal, shut down.
	if err := srv.Shutdown(ctx); err != nil {
		// Error from closing listeners, or context timeout:
		log.Printf("HTTP server Shutdown: %v", err)
	}
	log.Println("HTTP server graceful shutdown completed")
}
```

我们只需要将如下 3 行代码：

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit
```

替换成 K8s 提供的 `SetupSignalHandler` 函数调用即可：

```go
<-genericapiserver.SetupSignalHandler()
```

其他代码都不用修改。

执行示例程序，按一次 `Ctrl + C` 测试优雅退出：

```bash
$ go build -o main main.go && ./main
^C2024/08/22 09:24:46 Shutdown Server...
2024/08/22 09:24:46 Stopped serving new connections
2024/08/22 09:24:49 HTTP server graceful shutdown completed
$ echo $?
0
```

```bash
$ curl "http://localhost:8000/sleep?duration=5s"
Welcome HTTP Server
```

执行示例程序，按两次 `Ctrl + C` 测试强制退出：

```bash
$ go build -o main main.go && ./main
^C2024/08/22 09:25:28 Shutdown Server...
2024/08/22 09:25:28 Stopped serving new connections
^C
$ echo $?                                       
1
```

```bash
$ curl "http://localhost:8000/sleep?duration=5s"
curl: (52) Empty reply from server
```

完美，K8s 为我们提供了优雅退出的新思路。这样在开发环境，为了方便调试，我们可以无需等待优雅退出，只要连续发送两次 `SIGTERM/SIGINT` 即可强制退出程序。在生产环境发送一次 `SIGTERM/SIGINT` 信号等待优雅退出。

使用 Gin 框架开发的 Web 程序也可以这样修改，你可以自行尝试。

### gRPC 的优雅退出

#### gRPC Server 优雅退出

接下来我们一起看下 gRPC Server 程序如何实现优雅退出。

示例程序目录结构如下：

```bash
$ tree grpc
grpc
├── Makefile
├── client
│   └── main.go
├── pb
│   ├── helloworld.pb.go
│   ├── helloworld.proto
│   └── helloworld_grpc.pb.go
└── server
    └── main.go
```

`helloworld.proto` 中定义了 gRPC Server 支持的服务接口：

```protobuf
syntax = "proto3";

option go_package = ".;pb";

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
  string duration = 2;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

`server/main.go` 中 Server 端代码如下：

```go
// Package main implements a server for Greeter service.
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"net"
	"time"

	"google.golang.org/grpc"
	genericapiserver "k8s.io/apiserver/pkg/server"

	"github.com/jianghushinian/blog-go-example/gracefulstop/grpc/pb"
)

var (
	port = flag.Int("port", 50051, "The server port")
)

// server is used to implement helloworld.GreeterServer.
type server struct {
	pb.UnimplementedGreeterServer
}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())

	duration, _ := time.ParseDuration(in.GetDuration())
	time.Sleep(duration)

	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())

	go func() {
		if err := s.Serve(lis); err != nil {
			log.Fatalf("failed to serve: %v", err)
		}
	}()

	<-genericapiserver.SetupSignalHandler()
	log.Printf("Shutdown Server...")
	s.GracefulStop()
	log.Println("gRPC server graceful shutdown completed")
}
```

这与 HTTP Server 的优雅退出逻辑基本相同，同样由 `grpc` 包提供了优雅退出方法 `GracefulStop`。

在接收到退出信号以后，调用 `s.GracefulStop()` 方法即可实现优雅退出。可以发现，这其实是一个优雅退出的套路。

`client/main.go` 中 Client 端代码如下：

```go
// Package main implements a client for Greeter service.
package main

import (
	"context"
	"flag"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	"github.com/jianghushinian/blog-go-example/gracefulstop/grpc/pb"
)

const (
	defaultName = "world"
)

var (
	addr = flag.String("addr", "localhost:50051", "the address to connect to")
	name = flag.String("name", defaultName, "Name to greet")
)

func main() {
	flag.Parse()
	// Set up a connection to the server.
	conn, err := grpc.NewClient(*addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*60)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name, Duration: "10s"})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.GetMessage())
}
```

执行示例程序，测试优雅退出逻辑：

```bash
# 执行服务端代码
$ go build -o main main.go && ./main
2024/08/22 09:26:17 server listening at [::]:50051
2024/08/22 09:26:24 Received: world
^C2024/08/22 09:26:26 Shutdown Server...
2024/08/22 09:26:34 gRPC server graceful shutdown completed
$ echo $?
0
```

```bash
# 执行客户端代码
$ go build -o main main.go && ./main
2024/08/22 09:26:34 Greeting: Hello world
```

优雅退出生效。

既然 gRPC Server 中的优雅退出方案已经介绍完了，同讲解 HTTP Server 优雅退出一样，接下来我再带你一起深入了解一下 `GracefulStop` 的源码是如何实现的。

#### GracefulStop 源码

`GracefulStop` 方法源码如下：

> https://github.com/grpc/grpc-go/blob/v1.65.0/server.go#L1882

```go
// GracefulStop stops the gRPC server gracefully. It stops the server from
// accepting new connections and RPCs and blocks until all the pending RPCs are
// finished.
func (s *Server) GracefulStop() {
	s.stop(true)
}

func (s *Server) stop(graceful bool) {
	s.quit.Fire()
	defer s.done.Fire()

	s.channelzRemoveOnce.Do(func() { channelz.RemoveEntry(s.channelz.ID) })
	s.mu.Lock()
	s.closeListenersLocked()
	// Wait for serving threads to be ready to exit.  Only then can we be sure no
	// new conns will be created.
	s.mu.Unlock()
	s.serveWG.Wait()

	s.mu.Lock()
	defer s.mu.Unlock()

	if graceful {
		s.drainAllServerTransportsLocked()
	} else {
		s.closeServerTransportsLocked()
	}

	for len(s.conns) != 0 {
		s.cv.Wait()
	}
	s.conns = nil

	if s.opts.numServerWorkers > 0 {
		// Closing the channel (only once, via grpcsync.OnceFunc) after all the
		// connections have been closed above ensures that there are no
		// goroutines executing the callback passed to st.HandleStreams (where
		// the channel is written to).
		s.serverWorkerChannelClose()
	}

	if graceful || s.opts.waitForHandlers {
		s.handlersWG.Wait()
	}

	if s.events != nil {
		s.events.Finish()
		s.events = nil
	}
}
```

`GracefulStop` 方法直接调用了 `s.stop(true)` 方法。

`stop` 方法的 `graceful` 参数用来决定是否启用优雅退出，传递 `true` 表示优雅退出，传递 `false` 表示强制退出。

`stop` 方法第一段代码逻辑如下：

```go
s.quit.Fire()
defer s.done.Fire()
```

`Server` 对象的 `quit` 和 `done` 属性类型都为 `*grpcsync.Event`，前者用来标记 gRPC Server 正在执行退出流程，后者标记退出完成。

`Event` 定义如下：

> https://github.com/grpc/grpc-go/blob/v1.65.0/internal/grpcsync/event.go

```go
// Event represents a one-time event that may occur in the future.
type Event struct {
	fired int32
	c     chan struct{}
	o     sync.Once
}

// Fire causes e to complete.  It is safe to call multiple times, and
// concurrently.  It returns true iff this call to Fire caused the signaling
// channel returned by Done to close.
func (e *Event) Fire() bool {
	ret := false
	e.o.Do(func() {
		atomic.StoreInt32(&e.fired, 1)
		close(e.c)
		ret = true
	})
	return ret
}

// Done returns a channel that will be closed when Fire is called.
func (e *Event) Done() <-chan struct{} {
	return e.c
}

// HasFired returns true if Fire has been called.
func (e *Event) HasFired() bool {
	return atomic.LoadInt32(&e.fired) == 1
}

// NewEvent returns a new, ready-to-use Event.
func NewEvent() *Event {
	return &Event{c: make(chan struct{})}
}
```

可以发现，`*Event` 对象的 `Fire` 方法就是将 `fired` 字段值置为 `1`，并且关闭类型为 `channel` 的字段 `c`，所以其实只要调用了 `Fire`，那么调用 `Done` 方法将立即返回，调用 `HasFired` 方法就是在判断 `fired` 字段值是否为 `1`（即是否调用过 `Fire` 方法）。

`s.quit.Fire()` 代码被调用以后，`Serve` 方法就能够感知到当前服务正在退出，接下来就不会再接收新的请求进来了。

`Serve` 方法源码如下：

```go
// Serve accepts incoming connections on the listener lis, creating a new
// ServerTransport and service goroutine for each. The service goroutines
// read gRPC requests and then call the registered handlers to reply to them.
// Serve returns when lis.Accept fails with fatal errors.  lis will be closed when
// this method returns.
// Serve will return a non-nil error unless Stop or GracefulStop is called.
//
// Note: All supported releases of Go (as of December 2023) override the OS
// defaults for TCP keepalive time and interval to 15s. To enable TCP keepalive
// with OS defaults for keepalive time and interval, callers need to do the
// following two things:
//   - pass a net.Listener created by calling the Listen method on a
//     net.ListenConfig with the `KeepAlive` field set to a negative value. This
//     will result in the Go standard library not overriding OS defaults for TCP
//     keepalive interval and time. But this will also result in the Go standard
//     library not enabling TCP keepalives by default.
//   - override the Accept method on the passed in net.Listener and set the
//     SO_KEEPALIVE socket option to enable TCP keepalives, with OS defaults.
func (s *Server) Serve(lis net.Listener) error {
	s.mu.Lock()
	s.printf("serving")
	s.serve = true
	if s.lis == nil {
		// Serve called after Stop or GracefulStop.
		s.mu.Unlock()
		lis.Close()
		return ErrServerStopped
	}

	s.serveWG.Add(1)
	defer func() {
		s.serveWG.Done()
		// 判断当前服务是否正在退出
		if s.quit.HasFired() {
			// Stop or GracefulStop called; block until done and return nil.
			<-s.done.Done()
		}
	}()

	ls := &listenSocket{
		Listener: lis,
		channelz: channelz.RegisterSocket(&channelz.Socket{
			SocketType:    channelz.SocketTypeListen,
			Parent:        s.channelz,
			RefName:       lis.Addr().String(),
			LocalAddr:     lis.Addr(),
			SocketOptions: channelz.GetSocketOption(lis)},
		),
	}
	s.lis[ls] = true

	defer func() {
		s.mu.Lock()
		if s.lis != nil && s.lis[ls] {
			ls.Close()
			delete(s.lis, ls)
		}
		s.mu.Unlock()
	}()

	s.mu.Unlock()
	channelz.Info(logger, ls.channelz, "ListenSocket created")

	var tempDelay time.Duration // how long to sleep on accept failure
	for {
		rawConn, err := lis.Accept()
		if err != nil {
			if ne, ok := err.(interface {
				Temporary() bool
			}); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				s.mu.Lock()
				s.printf("Accept error: %v; retrying in %v", err, tempDelay)
				s.mu.Unlock()
				timer := time.NewTimer(tempDelay)
				select {
				case <-timer.C:
				// 判断当前服务是否正在退出
				case <-s.quit.Done():
					timer.Stop()
					return nil
				}
				continue
			}
			s.mu.Lock()
			s.printf("done serving; Accept = %v", err)
			s.mu.Unlock()

			// 判断当前服务是否正在退出
			if s.quit.HasFired() {
				return nil
			}
			return err
		}
		tempDelay = 0
		// Start a new goroutine to deal with rawConn so we don't stall this Accept
		// loop goroutine.
		//
		// Make sure we account for the goroutine so GracefulStop doesn't nil out
		// s.conns before this conn can be added.
		s.serveWG.Add(1)
		go func() {
			s.handleRawConn(lis.Addr().String(), rawConn)
			s.serveWG.Done()
		}()
	}
}
```

我们可以直接跳到 `for` 循环部分的代码段，每次通过 `rawConn, err := lis.Accept()` 接收到一个新的请求进来，都会使用 `if s.quit.HasFired()` 来判断当前服务是否正在退出，如果返回结果为 `true`，则 `Serve` 方法直接退出。

此时 `Serve` 方法的 `defer` 语句开始执行，这里会再次使用 `if s.quit.HasFired()` 判断当前服务是否正在退出（之所以判断两次，因为 `Serve` 方法也可能由于其他原因导致退出，进入 `defer` 逻辑），如果是，则调用 `<-s.done.Done()` 阻塞在这里，直到 `GracefulStop` 优雅退出逻辑执行完成。 

此外，`for` 循环内部的 `select` 语句中，有一个 case 调用了 `<-s.quit.Done()`，也是在判断当前服务是否正在退出，如果是，则调用 `timer.Stop()` 清理定时器后，`Serve` 方法直接退出。

我们接着往下看：

```go
s.channelzRemoveOnce.Do(func() { channelz.RemoveEntry(s.channelz.ID) })
s.mu.Lock()
s.closeListenersLocked()
// Wait for serving threads to be ready to exit.  Only then can we be sure no
// new conns will be created.
s.mu.Unlock()
s.serveWG.Wait()
```

这里的 `channelz` 是 gRPC 的一个监控工具，用于跟踪 gRPC 的内部状态，`RemoveEntry` 会移除该服务器的监控数据。

`s.closeListenersLocked()` 方法是不是很熟悉，这与 `net/http` 包中的 `Shutdown` 方法命名都一样，作用也就不言而喻了。

定义如下：

```go
// s.mu must be held by the caller.
func (s *Server) closeListenersLocked() {
	for lis := range s.lis {
		lis.Close()
	}
	s.lis = nil
}
```

对于 `s.serveWG.Wait()` 这行代码，根据这个操作的属性名和方法名可以猜到，`serveWG` 明显是 `sync.WaitGroup` 类型。

既然有 `Wait()`，那就应该会有 `Add(1)` 操作。其正在前文中贴出 `Serve` 代码中。

刚进入 `Serve` 方法时，就会调用 `s.serveWG.Add(1)`，`Serve` 方法退出时执行 `s.serveWG.Done()`：

```go
s.serveWG.Add(1)
defer func() {
	s.serveWG.Done()
	if s.quit.HasFired() {
		// Stop or GracefulStop called; block until done and return nil.
		<-s.done.Done()
	}
}()
```

并且，在 `Serve` 方法的 `for` 循环逻辑中，每次有新的请求进来，`s.serveWG.Add(1)` 和 `s.serveWG.Done()` 也会被调用一次：

```go
s.serveWG.Add(1)
go func() {
	s.handleRawConn(lis.Addr().String(), rawConn)
	s.serveWG.Done()
}()
```

所以，这里其实是在等待 `Serve` 方法执行完成并退出。

接下来的代码段是根据是否要进行优雅退出，执行不同的逻辑：

```go
s.mu.Lock()
defer s.mu.Unlock()

if graceful {
    s.drainAllServerTransportsLocked()
} else {
    s.closeServerTransportsLocked()
}
```

可以发现，接下来的全部操作都加锁处理。

优雅退出走 `s.drainAllServerTransportsLocked()` 逻辑：

```go
// s.mu must be held by the caller.
func (s *Server) drainAllServerTransportsLocked() {
	if !s.drain {
		for _, conns := range s.conns {
			for st := range conns {
				st.Drain("graceful_stop")
			}
		}
		s.drain = true
	}
}
```

它的主要作用是在服务器优雅停止的过程中，让所有的服务器传输层（ServerTransports）停止接收新的请求，但继续处理现有的请求，直到它们完成。

这里有一行注释：`s.mu must be held by the caller`。

说明了在调用 `drainAllServerTransportsLocked` 方法之前，调用者必须已经持有 `s.mu` 锁。这是为了确保在执行方法体时，服务器的状态不会被并发修改。

对于 `if !s.drain` 这个条件判断，其用于确保 `drainAllServerTransportsLocked` 方法只会执行一次。

`s.conns` 属性保存了所有连接，是 `map[string]map[transport.ServerTransport]bool` 类型。

通过嵌套的 `for` 循环遍历每个连接中的 `ServerTransport` 实例。`ServerTransport` 是 gRPC 中的一个概念，它表示一个抽象的传输层实现，负责处理客户端和服务器之间的实际数据传输。

调用 `st.Drain("graceful_stop")` 方法的作用，是告诉传输层不要再接收新的请求或连接了，但允许继续处理现有的请求，直到它们完成。

这个方法会向客户端发送信号，表明服务器正在进行优雅关闭。它会给所有的客户端发送一个控制帧 `GOAWAY`（因为 gRPC 是基于 HTTP/2 的，所以才会这样处理），告诉客户端关闭 TCP 连接。

`Drain` 方法实现如下：

```go
func (t *http2Server) Drain(debugData string) {
	t.mu.Lock()
	defer t.mu.Unlock()
	if t.drainEvent != nil {
		return
	}
	t.drainEvent = grpcsync.NewEvent()
	t.controlBuf.put(&goAway{code: http2.ErrCodeNo, debugData: []byte(debugData), headsUp: true})
}
```

在完成对所有连接的遍历和 `Drain` 操作后，将 `s.drain` 设置为 `true`，表示服务器已经进入了 "drain" 状态，这样后续不会再次执行相同的操作。

结合之前持有锁的操作，这里不会重复执行。

相对来说，没有采用优雅退出的另一个方法 `closeServerTransportsLocked` 就要暴力一些：

```go
// s.mu must be held by the caller.
func (s *Server) closeServerTransportsLocked() {
	for _, conns := range s.conns {
		for st := range conns {
			st.Close(errors.New("Server.Stop called"))
		}
	}
}
```

这里直接调用 `Close` 方法关闭连接，省略了控制帧 `GOAWAY` 的发送。

`stop` 函数接下来会等待所有现有的连接被安全关闭：

```go
for len(s.conns) != 0 {
    s.cv.Wait()
}
s.conns = nil
```

继续往下执行：

```go
if s.opts.numServerWorkers > 0 {
    // Closing the channel (only once, via grpcsync.OnceFunc) after all the
    // connections have been closed above ensures that there are no
    // goroutines executing the callback passed to st.HandleStreams (where
    // the channel is written to).
    s.serverWorkerChannelClose()
}
```

这段代码用于关闭工作线程的 `channel`，确保所有处理程序都已经终止，不会再处理新的请求。

`s.serverWorkerChannelClose` 在初始化操作时被赋值：

```go
// initServerWorkers creates worker goroutines and a channel to process incoming
// connections to reduce the time spent overall on runtime.morestack.
func (s *Server) initServerWorkers() {
	s.serverWorkerChannel = make(chan func())
	s.serverWorkerChannelClose = grpcsync.OnceFunc(func() {
		close(s.serverWorkerChannel)
	})
	for i := uint32(0); i < s.opts.numServerWorkers; i++ {
		go s.serverWorker()
	}
}
```

接下来，如果是优雅退出或者配置了 `s.opts.waitForHandlers` 选项，则代码会调用 `s.handlersWG.Wait()`，等待所有的处理程序完成：

```go
if graceful || s.opts.waitForHandlers {
	s.handlersWG.Wait()
}
```

不知你有没有注意到，`Serve` 方法内部，调用了 `s.handleRawConn(lis.Addr().String(), rawConn)` 来处理每一个请求。

而 `handleRawConn` 方法内部会调用 `s.serveStreams(context.Background(), st, rawConn)` 方法。

`serveStreams` 方法定义如下：

```go
func (s *Server) serveStreams(ctx context.Context, st transport.ServerTransport, rawConn net.Conn) {
	ctx = transport.SetConnection(ctx, rawConn)
	ctx = peer.NewContext(ctx, st.Peer())
	for _, sh := range s.opts.statsHandlers {
		ctx = sh.TagConn(ctx, &stats.ConnTagInfo{
			RemoteAddr: st.Peer().Addr,
			LocalAddr:  st.Peer().LocalAddr,
		})
		sh.HandleConn(ctx, &stats.ConnBegin{})
	}

	defer func() {
		st.Close(errors.New("finished serving streams for the server transport"))
		for _, sh := range s.opts.statsHandlers {
			sh.HandleConn(ctx, &stats.ConnEnd{})
		}
	}()

	streamQuota := newHandlerQuota(s.opts.maxConcurrentStreams)
	st.HandleStreams(ctx, func(stream *transport.Stream) {
		s.handlersWG.Add(1)
		streamQuota.acquire()
		f := func() {
			defer streamQuota.release()
			defer s.handlersWG.Done()
			s.handleStream(st, stream)
		}

		if s.opts.numServerWorkers > 0 {
			select {
			case s.serverWorkerChannel <- f:
				return
			default:
				// If all stream workers are busy, fallback to the default code path.
			}
		}
		go f()
	})
}
```

`serveStreams` 方法内部会调用 `s.handlersWG.Add(1)` 和 `s.handlersWG.Done()` 操作。

`serveStreams` 执行完成后，`stop` 方法就可以继续执行了：

```go
if s.events != nil {
	s.events.Finish()
	s.events = nil
}
```

`stop` 方法最后，对事件进行了清理，检查 `s.events` 是否不为空，如果不为空，则调用 `s.events.Finish()`，完成事件的清理工作。

至此，`GracefulStop` 的源码就分析完成了。

可以发现 `grpc` 的优雅退出跟 `net/http` 的优雅退出还是有很多相似之处的，这也很好理解，gRPC 是基于 HTTP/2 的，所以底层也是 HTTP 协议。

### 普通 Go 程序的优雅退出

在日常开发中，除了 HTTP Server 或者 gRPC Server 程序，我们也可能会开发一些其他需要优雅退出的程序。

在文章的最后一个小节，我再来介绍一种常见的周期性执行任务的 Go 程序如何实现优雅退出。

示例代码如下：

```go
package main

import (
	"log"
	"time"

	genericapiserver "k8s.io/apiserver/pkg/server"
)

type Syncer struct {
	interval time.Duration
}

func (s *Syncer) Run(quit <-chan struct{}) error {
	ticker := time.NewTicker(s.interval)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			// 业务逻辑
			log.Println("do something")
		case <-quit:
			log.Println("Stop loop")
			s.Stop()
			return nil
		}
	}
}

func (s *Syncer) Stop() {
	log.Println("Stop Syncer start")
	time.Sleep(time.Second * 5)
	log.Println("Stop Syncer done")
}

func main() {
	s := Syncer{interval: time.Second}
	quit := genericapiserver.SetupSignalHandler()

	if err := s.Run(quit); err != nil {
		log.Fatalf("Syncer run err: %s", err.Error())
	}
}
```

这里定义了一个 `Syncer` 结构体，用来实现周期性执行一段代码逻辑。

`Run` 方法中用到了 `time.Ticker`，在 `for` 循环中周期执行某些业务逻辑，比如定时同步数据状态、生成数据报表等。

`Run` 方法中还用到了类似 `http.Server.Shutdown` 方法内部的 `for` + `select` 代码结构，来实现接收退出信号 `<-quit`。当收到退出信号，就调用 `s.Stop()` 进行优雅退出逻辑。

执行示例代码，中途按 `Ctrl + C` 进行优雅退出操作：

```bash
$ go build -o main main.go && ./main
2024/08/22 09:27:34 do something
2024/08/22 09:27:35 do something
2024/08/22 09:27:36 do something
^C2024/08/22 09:27:37 Stop loop
2024/08/22 09:27:37 Stop Syncer start
2024/08/22 09:27:42 Stop Syncer done
```

也可以尝试强制退出：

```bash
$ go build -o main main.go && ./main
2024/08/22 09:28:05 do something
2024/08/22 09:28:06 do something
2024/08/22 09:28:07 do something
^C2024/08/22 09:28:07 Stop loop
2024/08/22 09:28:07 Stop Syncer start
^C
```

两种操作都没有问题。

可以将这个示例程序作为优雅退出的代码模板，集成进你的 Go 程序中。

至此，本文要讲解的内容就全部都写完了。

### 总结

所谓的优雅退出，其实就是在关闭进程的时候，不能“暴力”关闭，而是要等待进程中的逻辑（比如一次完整的 HTTP 请求）处理完成后，才关闭进程。

Go 为我们提供的 `os/signal` 包进行信号处理。默认情况下接收到退出信号 Go 程序会立即退出，使用 `signal.Notify` 注册关注的退出信号以后，我们可以实现自己的处理逻辑。

常见退出信号有 `SIGINT`、`SIGQUIT`、`SIGTERM` 和 `SIGKILL`，`SIGKILL` 不能被 Go 程序捕获。

我们分别为 HTTP Server、gRPC Server 以及周期性执行任务的 Go 程序实现了优雅退出功能，并且对 `net/http` 包的 `Shutdown` 源码以及 `grpc` 包的 `GracefulStop` 源码都进行了分析和讲解。

本文还重点讲解了 K8s 为我们提供了一种更加优雅的方式，来实现优雅退出功能。我们可以实现收到一次 `SIGINT/SIGTERM` 信号，程序优雅退出，收到两次 `SIGINT/SIGTERM` 信号，程序强制退出。

切记，忽略优雅退出可能会导致数据的不一致问题，因此实现优雅退出功能是非常有必要的。

实现优雅退出最核心的两点：

1. 接收退出信号。
2. 如何等待正在处理的任务完成，还要考虑超时机制。

这两步做完，程序就可以退出了。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/gracefulstop) 中，欢迎点击查看。

希望此文能对你有所启发。

### P.S.

这是我迄今为止，写过的最长单篇文章了，希望读完此文的你，能一次将 Go 程序中的优雅退出理解透彻。本来只是想简单分享下 K8s 中优雅退出的思路，没想到越写越多，再不收手就真的控制不住了🤣。我差点要继续去深入研究 `os/signal`、`net/http`、`grpc`... 的源码，等以后有机会再单独分享吧 :)。

**延伸阅读**

- os/signal Documentation：https://pkg.go.dev/os/signal@go1.22.0
- os/signal 源码：https://github.com/golang/go/blob/release-branch.go1.22/src/os/signal/signal.go
- net/http Documentation：https://pkg.go.dev/net/http@go1.22.0
- net/http Server Shutdown 源码：https://github.com/golang/go/blob/release-branch.go1.22/src/net/http/server.go
- Gin Web Framework 文档：https://gin-gonic.com/zh-cn/docs/examples/graceful-restart-or-stop/
- Gin graceful-shutdown examples：https://github.com/gin-gonic/examples/tree/master/graceful-shutdown
- gRPC Quick start：https://grpc.io/docs/languages/go/quickstart/
- gRPC-Go Examples：https://github.com/grpc/grpc-go/tree/master/examples
- gRPC Server GracefulStop 源码：https://github.com/grpc/grpc-go/blob/v1.65.0/server.go
- Kubernetes 优雅退出信号处理源码：https://github.com/kubernetes/apiserver/blob/release-1.31/pkg/server/signal.go
- Proper HTTP shutdown in Go：https://dev.to/mokiat/proper-http-shutdown-in-go-3fji
- Golang: graceful shutdown：https://dev.to/antonkuklin/golang-graceful-shutdown-3n6d
- How to stop http.ListenAndServe()：https://stackoverflow.com/questions/39320025/how-to-stop-http-listenandserve
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/gracefulstop

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn

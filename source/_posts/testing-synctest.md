---
title: '使用 testing/synctest 测试并发代码'
date: 2025-09-05 12:03:35
tags:
- Go
- 单元测试
categories:
- Go
---

大家好，我是江湖十年。

Go 1.25 发布有段时间了，随之带来了正式版本的并发测试包 `testing/synctest`，今天就来通过这篇文章向大家介绍一下在 Go 中如何测试并发代码，本文翻译自 Go 官方博客。

<!-- more -->

> 原文地址：[https://go.dev/blog/synctest](https://go.dev/blog/synctest)
>


_戴米安·尼尔_
_2025 年 2 月 19 日_

Go 语言的一个标志性特性是内置的并发支持。Goroutines 和 channels 是编写并发程序的简单且有效的原语。

然而，测试并发程序可能很困难且容易出错。

在 Go 1.24 中，我们引入了一个新的、实验性的 [testing/synctest](https://pkg.go.dev/testing/synctest) 包来支持测试并发代码。本文将解释这项实验背后的动机，演示如何使用 `synctest` 包，并讨论其未来的发展潜力。

在 Go 1.24 中，`testing/synctest` 包处于实验阶段，不受 Go 兼容性承诺的约束。它默认不可见。要使用它，请在环境变量中设置 `GOEXPERIMENT=synctest` 来编译你的代码。

### 测试并发程序很困难

首先，让我们考虑一个简单的例子。

`context.AfterFunc` 函数会在上下文取消后在其自己的 goroutine 中调用一个函数。以下是 `AfterFunc` 的一个可能的测试：

```go
func TestAfterFunc(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())

    calledCh := make(chan struct{}) // closed when AfterFunc is called
    context.AfterFunc(ctx, func() {
        close(calledCh)
    })

    // TODO: Assert that the AfterFunc has not been called.

    cancel()

    // TODO: Assert that the AfterFunc has been called.
}
```

我们希望在这个测试中检查两个条件：在上下文取消之前没有调用该函数，以及在上下文取消之后调用了该函数。

在并发系统中检查反向结果是有困难的。我们可以很容易的测试该函数尚未被调用，但如何检查它不会被调用呢？

一种常见的方法是等待一段时间后，再得出事件不会发生的结论。让我们尝试在测试中引入一个辅助函数来实现这个操作。

```go
// funcCalled reports whether the function was called.
funcCalled := func() bool {
    select {
    case <-calledCh:
        return true
    case <-time.After(10 * time.Millisecond):
        return false
    }
}

if funcCalled() {
    t.Fatalf("AfterFunc function called before context is canceled")
}

cancel()

if !funcCalled() {
    t.Fatalf("AfterFunc function not called after context is canceled")
}
```

这个测试很慢：10 毫秒并不是很长的时间，但经过多次测试就会累积起来。

这个测试也不稳定：10 毫秒在高速计算机中已经是很长的时间，但在共享和过载的 [CI](https://en.wikipedia.org/wiki/Continuous_integration) 系统上看到持续几秒钟的停顿并不罕见。

我们可以通过牺牲速度来减少测试的不稳定性，或者通过牺牲稳定性来减少测试的速度，但我们无法让测试既快速又稳定。

### testing/synctest 包简介

`testing/synctest` 包解决了这个问题。它允许我们将这个测试改写成简单、快速且可靠的形式，而无需对被测试的代码进行任何修改。

该包只包含两个函数：`Run` 和 `Wait`。

`Run` 在一个新的 goroutine 中调用一个函数。这个 goroutine 和它启动的所有 goroutine 都存在于一个隔离的环境中，我们称之为“气泡”（bubble）。`Wait` 会等待当前 goroutine 所在气泡中的每一个 goroutine 都阻塞在气泡内的另一个 goroutine 上。

让我们使用 `testing/synctest` 包重写上面的测试。

```go
func TestAfterFunc(t *testing.T) {
    synctest.Run(func() {
        ctx, cancel := context.WithCancel(context.Background())

        funcCalled := false
        context.AfterFunc(ctx, func() {
            funcCalled = true
        })

        synctest.Wait()
        if funcCalled {
            t.Fatalf("AfterFunc function called before context is canceled")
        }

        cancel()

        synctest.Wait()
        if !funcCalled {
            t.Fatalf("AfterFunc function not called after context is canceled")
        }
    })
}
```

这与我们最初的测试几乎相同，但是我们将测试包装在 `synctest.Run` 调用中，并且在断言该函数是否已被调用之前调用 `synctest.Wait`。

`Wait` 函数等待调用者气泡中的所有 goroutine 阻塞。当它返回时，我们知道 `context` 包要么调用了该函数，要么在我们采取进一步行动之前不会调用它。

该测试现在既快速又稳定。

测试也更简单了：我们用布尔值替换了 `calledCh` 通道（`channel`）。之前我们需要使用 `channel` 来避免测试 goroutine 和 `AfterFunc` goroutine 之间的数据竞争，但 `Wait` 函数现在提供了这种同步功能。

竞争检测器能够理解 `Wait` 调用，并且此测试在使用 `-race` 选项运行时能够通过。如果我们移除第二个 `Wait` 调用，竞争检测器就能在测试中正确报告数据竞争。

### 测试时间

并发代码通常与时间有关。

测试与时间相关的代码可能很困难。正如我们在上面看到的，在测试中使用真实时间会导致测试缓慢且不稳定。使用虚拟时间（fake time）则需要避免使用 `test` 包中的函数，并将被测代码设计为支持使用可选的虚拟时钟（fake clock）。

`testing/synctest` 包使测试使用时间的代码变得更加简单。

`Run` 启动的气泡中的 Goroutines 使用虚拟时钟。在气泡内，`time` 包中的函数会操作这个虚拟时钟。当所有 goroutines 都被阻塞时，气泡中的时间会向前推进。

为了演示，让我们为 [context.WithTimeout](https://go.dev/pkg/context#WithTimeout) 函数写一个测试，`WithTimeout` 会创建一个子 context，并在到达给定的超时时间后过期。

```go
func TestWithTimeout(t *testing.T) {
    synctest.Run(func() {
        const timeout = 5 * time.Second
        ctx, cancel := context.WithTimeout(context.Background(), timeout)
        defer cancel()

        // Wait just less than the timeout.
        time.Sleep(timeout - time.Nanosecond)
        synctest.Wait()
        if err := ctx.Err(); err != nil {
            t.Fatalf("before timeout, ctx.Err() = %v; want nil", err)
        }

        // Wait the rest of the way until the timeout.
        time.Sleep(time.Nanosecond)
        synctest.Wait()
        if err := ctx.Err(); err != context.DeadlineExceeded {
            t.Fatalf("after timeout, ctx.Err() = %v; want DeadlineExceeded", err)
        }
    })
}
```

我们编写这个测试就像使用真实时间一样。唯一的区别是我们将测试函数包装在 `synctest.Run`，并在每次 `time.Sleep` 调用后调用 `synctest.Wait` 来等待 context 包的计时器运行完毕。

### 阻塞和气泡

`testing/synctest`中的一个关键概念是气泡被_持久阻塞（durably blocked）_。当气泡中的每一个 goroutine 都被阻塞时，就会发生这种情况，并且只能由气泡中的另一个 goroutine 解除阻塞。

当气泡被持久阻塞时：

+ 如果有未完成的 `Wait` 调用，它将返回。
+ 否则，时间将向前推进到下一个可以解除 goroutine 阻塞的时间（如果有）。
+ 否则，气泡就会陷入僵局并且 `Run` 会陷入恐慌（`panics`）。

如果任何 goroutine 被阻塞，则气泡不会被持久阻塞，但可能会被气泡外的某些事件唤醒。

持久阻塞 goroutine 操作的完整列表是：

+ 在 `nil channel`上发送或接收
+ 在同一气泡内创建的 `channel` 上发送或接收被阻塞
+ 一个 `select` 语句，其中每个 `case` 都持久阻塞
+ `time.Sleep`
+ `sync.Cond.Wait`
+ `sync.WaitGroup.Wait`

#### 互斥锁

操作 `sync.Mutex` 不会持久阻塞。

函数获取全局互斥锁是很常见的。例如，`reflect` 包中的许多函数使用由互斥锁保护的全局缓存。如果 `synctest` 气泡中的 goroutine 在获取气泡外的 goroutine 持有的互斥锁时阻塞，则它不会被持久阻塞——它会阻塞，但会被其气泡外部的 goroutine 解除阻塞。

由于互斥锁通常不会长时间保持，因此我们只是将它们排除在 `testing/synctest` 的考虑之外。

#### Channels

在气泡中创建的 Channels 与在外部创建的行为不同。

仅当 Channel 是在气泡中创建的时，Channel 操作才会持久阻塞。在气泡外操作在气泡内部创建的 channel 会导致气泡 `panics`。

这些规则确保 goroutine 仅在其气泡内的 goroutine 中通信时才会被持久阻塞。

#### IO

外部 I/O 作（例如从网络连接读取）不会持久阻塞。

网络读取可能会被来自气泡外部的写入解除阻塞，甚至可能来自其他进程。即使网络连接的唯一写入者也在同一个气泡中，运行时也无法区分等待更多数据到达的连接和内核已接收数据并正在传递数据的连接。

使用 `synctest` 测试网络服务器或客户端通常需要提供虚拟网络（fake network）实现。例如，[net.Pipe](https://go.dev/pkg/net#Pipe) 函数创建一个网络对，使用内存中网络连接并可用于同步测试的连接器。

### 气泡生命周期

`Run` 函数在新气泡中启动一个 goroutine。当气泡中的每个 goroutine 都退出时，它会返回。如果气泡被持久阻塞并且无法随着时间推进而解锁，它就会产生 `panics`。

在 `Run` 返回之前，气泡中的每个 goroutine 都退出的要求意味着测试在完成之前必须小心清理任何后台 goroutine。

### 测试网络代码

让我们看下另外一个例子，这次使用 `testing/synctest` 包来测试网络程序。 对于此示例，我们将测试 `net/http` 包对状态码为 `100 Continue` 的响应处理。

发送请求的 HTTP 客户端可以包含“`Expect: 100-continue`”请求头，以告诉服务器客户端有其他数据要发送。服务器随后可能会返回 `100 Continue` 信息响应，请求客户端继续发送剩余数据，或者返回其他状态码，告知客户端不再需要该内容。例如，上传大文件的客户端可能会使用此功能在发送文件之前确认服务器是否愿意接受该文件。

我们的测试将确认，在发送“`Expect: 100-continue`”请求头时，HTTP 客户端不会在服务器请求之前发送请求的内容，并且在收到 `100 Continue` 响应后确实发送了内容。

通常，通信客户端和服务器的测试可以使用环回网络（loopback network）连接。然而，在使用 `testing/synctest` 时，我们通常希望使用虚拟的网络连接来检测网络上所有 goroutine 是否被阻塞。我们将通过创建一个 `http.Transport`（一个 HTTP 客户端）来开始此测试，该客户端使用由 [net.Pipe](https://go.dev/pkg/net#Pipe) 创建的内存网络连接。

```go
func Test(t *testing.T) {
    synctest.Run(func() {
        srvConn, cliConn := net.Pipe()
        defer srvConn.Close()
        defer cliConn.Close()
        tr := &http.Transport{
            DialContext: func(ctx context.Context, network, address string) (net.Conn, error) {
                return cliConn, nil
            },
            // Setting a non-zero timeout enables "Expect: 100-continue" handling.
            // Since the following test does not sleep,
            // we will never encounter this timeout,
            // even if the test takes a long time to run on a slow machine.
            ExpectContinueTimeout: 5 * time.Second,
        }
```

我们通过此传输发送一个设置了“`Expect: 100-continue`”头的请求。该请求会在一个新的 goroutine 中发送，因为它要到测试结束才会完成。

```go
        body := "request body"
        go func() {
            req, _ := http.NewRequest("PUT", "http://test.tld/", strings.NewReader(body))
            req.Header.Set("Expect", "100-continue")
            resp, err := tr.RoundTrip(req)
            if err != nil {
                t.Errorf("RoundTrip: unexpected error %v", err)
            } else {
                resp.Body.Close()
            }
        }()
```

我们读取客户端发送的请求头。

```go
        req, err := http.ReadRequest(bufio.NewReader(srvConn))
        if err != nil {
            t.Fatalf("ReadRequest: %v", err)
        }
```

现在我们来到了测试的核心。我们想断言客户端还不会发送请求正文。

我们启动一个新的 goroutine，将发送到服务器的 `body` 复制到 `strings.Builder` 中，等待气泡中的所有 goroutine 阻塞，并验证我们尚未从 `body` 中读取到任何内容。

如果我们忘记了 `synctest.Wait `调用，竞争检测器将正确地检测到数据竞争，但有了 `Wait`，这是安全的。

```go
        var gotBody strings.Builder
        go io.Copy(&gotBody, req.Body)
        synctest.Wait()
        if got := gotBody.String(); got != "" {
            t.Fatalf("before sending 100 Continue, unexpectedly read body: %q", got)
        }
```

我们向客户端写入“`100 Continue`”响应并验证它现在是否发送请求正文。

```go
        srvConn.Write([]byte("HTTP/1.1 100 Continue\r\n\r\n"))
        synctest.Wait()
        if got := gotBody.String(); got != body {
            t.Fatalf("after sending 100 Continue, read body %q, want %q", got, body)
        }
```

最后，我们发送“`200 OK`”响应以结束请求。

我们在本次测试中启动了多个 goroutine。`synctest.Run` 调用将等待所有 goroutine 退出后再返回。

```go
        srvConn.Write([]byte("HTTP/1.1 200 OK\r\n\r\n"))
    })
}
```

此测试可以轻松扩展以测试其他行为，例如验证如果服务器未要求则不发送请求 `body`，或者如果服务器在超时内未响应则发送请求 `body`。

### 实验状态

我们在 Go 1.24 中引入一个试验性的包 [testing/synctest](https://go.dev/pkg/testing/synctest) 。根据反馈和使用经验，我们可能会发布包含或不包含任何修改的软件包，继续进行实验，或者在 Go 的未来版本中将其移除。

默认情况下，该包不可见。要使用它，请在您的环境中设置 `GOEXPERIMENT=synctest` 并编译代码。

我们期待您的反馈！如果您尝试使用了 `testing/synctest`，请在 [go.dev/issue/67434](https://go.dev/issue/67434) 上报告您的体验，无论是正面的还是负面的。

原文完。

希望此文能对你有所启发。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/synctest) 中，欢迎点击查看。

**延伸阅读**

+ Testing concurrent code with testing/synctest：[https://go.dev/blog/synctest](https://go.dev/blog/synctest)
+ testing/synctest Documentation：[https://pkg.go.dev/testing/synctest](https://pkg.go.dev/testing/synctest)
+ Go 1.24 Release Notes：[https://go.dev/doc/go1.24](https://go.dev/doc/go1.24)
+ Go 1.25 Release Notes：[https://go.dev/doc/go1.25](https://go.dev/doc/go1.25)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/test/synctest](https://github.com/jianghushinian/blog-go-example/tree/main/test/synctest)
+ 本文永久地址：[https://jianghushinian.cn/2025/09/05/testing-synctest/](https://jianghushinian.cn/2025/09/05/testing-synctest/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

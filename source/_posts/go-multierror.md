---
title: 'go-multierror: 更方便的处理你的错误列表'
date: 2025-03-23 20:13:54
tags:
- Go
- Error
categories:
- Go
---

`go-multierror` 是一个第三方的 Go 语言库，用于处理多个错误的聚合与管理。它由 HashiCorp 提供，非常适合需要在某些操作中收集多个错误并在最后统一返回的场景。

<!-- more -->

### 使用示例

顾名思义，`go-multierror` 包的核心功能就一个，将多个错误合并为一个错误。以下是一个典型的使用示例：

```go
package main

import (
	"errors"
	"fmt"

	"github.com/hashicorp/go-multierror"
)

func main() {
	var errs *multierror.Error

	// 模拟多个操作可能失败
	if err := step1(); err != nil {
		errs = multierror.Append(errs, err)
	}

	if err := step2(); err != nil {
		errs = multierror.Append(errs, err)
	}

	// 如果有任何错误，将返回聚合的错误
	if errs != nil {
		fmt.Println("Errors occurred:")
		fmt.Println(errs.Error())
	} else {
		fmt.Println("All steps succeeded!")
	}
}

func step1() error {
	return errors.New("step1 failed")
}

func step2() error {
	return errors.New("step2 failed")
}
```

示例中，我们定义了一个 `*multierror.Error` 类型的 `errs` 变量，它提供了 `Append` 方法可以为其追加 `error`。当 `step1` 和 `step2` 都失败时，`errs` 中就记录了这两个 `error`。

执行示例代码，得到输出如下：

```bash
$ go run main.go                     
Errors occurred:
2 errors occurred:
        * step1 failed
        * step2 failed

```

可以看到，`*multierror.Error` 可以输出错误数量 2，以及其包含的每个 `error` 信息。

### 定制错误格式

我们还可以自定义错误的输出格式，修改上文示例中的 `if errs != nil` 代码块中的逻辑，将 `errs.ErrorFormat` 属性重新赋值为自定义函数：

```go
if errs != nil {
    errs.ErrorFormat = func(e []error) string {
        return e[0].Error()
    }
    fmt.Println("Errors occurred:")
    fmt.Println(errs.Error())
}
```

这个匿名函数接收一个 `error` 列表，并返回 `error` 列表中的第一个错误信息。

执行示例代码，得到输出如下：

```bash
$ go run main.go
Errors occurred:
step1 failed
```

基于此你可以定制任何自己想要的格式。

### 错误兼容性

`go-multierror` 包兼容了 `errors.Unwrap` 方法，所以它可以实现错误解包操作。

示例如下：

```go
if errs != nil {
    fmt.Println("Errors occurred:")
    fmt.Println(errs.Error())
    fmt.Println(errors.Unwrap(errs))
    fmt.Println(errors.Unwrap(errors.Unwrap(errs)))
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
Errors occurred:
2 errors occurred:
        * step1 failed
        * step2 failed


step1 failed
step2 failed
```

事实上，`go-multierror` 包完全兼容 Go 内置的 `error` 操作，无论是错误断言 `err.(*multierror.Error)` 还是 `errors.Is` 和 `errors.As` 都可以支持，你可以自行尝试。

### 并发场景

遗憾的是，`go-multierror` 包默认不支持并发操作。有如下测试代码：

```go
func TestConcurrency(t *testing.T) {
	var wg sync.WaitGroup
	errs := &multierror.Error{}

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			errs = multierror.Append(errs, errors.New("error"))
		}()
	}

	wg.Wait()
	// 预期 100 个错误，实际输出可能 < 100
	assert.Equal(t, 100, len(errs.Errors)) // 测试失败
}
```

这里并发开启 100 个 goroutine，每个 goroutine 中追加一个 `error` 到 `errs` 对象中，使用 `sync.WaitGroup` 等待并发操作完成。

> NOTE:
>
> 如果你不熟悉 `sync.WaitGroup`，可以参考我的文章「[Go 并发控制：sync.WaitGroup 详解](https://jianghushinian.cn/2024/12/23/sync-waitgroup/)」。
>

执行测试代码，得到输出如下：

```bash
$ go test -v -run=^TestConcurrency$
=== RUN   TestConcurrency
    main_test.go:26: 
                Error Trace:    /blog-go-example/error/go-multierror/main_test.go:26
                Error:          Not equal: 
                                expected: 100
                                actual  : 79
                Test:           TestConcurrency
--- FAIL: TestConcurrency (0.00s)
FAIL
exit status 1
FAIL    github.com/jianghushinian/error/go-multierror   0.196s
```

可以看到，测试结果失败了，预期 100 个错误，实际只得到 79 个错误。说明在并发操作过程中有部分 `error` 丢失了。

要解决这个问题，我们可以使用 `channel` 来并发安全的发送 `error` 对象，然后将错误聚合操作放在一个 goroutine 中处理。

示例如下：

```go
func TestConcurrencyWithChannel(t *testing.T) {
	var wg sync.WaitGroup
	errCh := make(chan error, 100)
	errs := &multierror.Error{}

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			errCh <- errors.New("error") // 通过 channel 安全发送 err
		}()
	}

	// 开启子 goroutine 等待并发程序执行完成
	go func() {
		wg.Wait()
		close(errCh)
	}()

	// main goroutine 从 channel 收到 err 并完成聚合
	for err := range errCh {
		errs = multierror.Append(errs, err) // 单 goroutine 聚合，无竞争
	}

	// 预期 100 个错误，实际输出也是 100
	assert.Equal(t, 100, len(errs.Errors)) // 测试通过
}
```

我们对上文中的测试代码进行了改造，在所有子 goroutine 中不再直接追加 `error` 到 `errs` 对象中，而是先将其发送到带有缓冲的 `channel` 中，最终由 `main` goroutine 从 `channel` 收到 `err` 并完成聚合操作。

执行修改后的测试代码，得到输出如下：

```bash
$ go test -v -run=^TestConcurrencyWithChannel$
=== RUN   TestConcurrencyWithChannel
--- PASS: TestConcurrencyWithChannel (0.00s)
PASS
ok      github.com/jianghushinian/error/go-multierror   0.403s
```

这一次测试成功了，说明我们使用 `channel` 的方式解决了 `go-multierror` 包的并发问题。

### 常用方法

`go-multierror` 包常用方法总结如下：

1. `multierror.Append(*Error, error)`：向 `multierror.Error` 对象添加一个新错误。如果传入的 `multierror.Error` 为 `nil`，会自动创建新的实例。
2. `*Error.Error()`：将所有错误格式化为字符串，按序列号展示。
3. `*Error.ErrorOrNil()`：如果没有错误，返回 `nil`。否则，返回聚合后的错误。

### 使用场景

最后我们再来探讨下 `go-multierror` 的常见使用场景，我认为有如下场景比较适合使用 `go-multierror` ：

+ **批量操作**：当程序需要对一组任务（如文件处理、并发请求等）逐个执行，但每个任务可能独立失败时，使用 `multierror` 可以方便地记录并返回所有失败信息。此时的你是否想起了 `errgroup` 呢？
+ **资源清理**：当程序释放多个资源时（比如执行多个 `defer`），若某些清理操作失败，可以收集这些错误并统一报告。
+ **复杂流程错误管理**：在长流程中，允许多个步骤分别记录错误，而不是只返回第一个错误。

### 总结

`go-multierror` 非常简单易用，它适用于需要同时管理多个错误的场景。并且完全兼容 Go `error`，所以也非常实用。总之，`go-multierror` 是一个小巧而实用的错误管理工具，特别适合在复杂场景中对多个错误进行统一处理和报告。

不过最后我想额外提一点，`go-multierror` 项目采用 [MPL-2.0 license](https://github.com/hashicorp/go-multierror?tab=MPL-2.0-1-ov-file#MPL-2.0-1-ov-file) 开源协议，这个协议是比 Apache 2.0 协议更加严格的开源协议，如果你的商业化软件使用并修改了其源码，则修改后的源码需要以同样协议进行开源。

下图是我制作的常见开源协议一览图，从左到右限制越来越宽松，如果你感兴趣也可以阅读我的另一篇文章「[开源协议简介](https://jianghushinian.cn/2023/01/15/open-source-license-introduction/)」。

![licence](licence.png)

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/error/go-multierror) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ go-multierror Documentation：[https://pkg.go.dev/github.com/hashicorp/go-multierror@v1.1.1](https://pkg.go.dev/github.com/hashicorp/go-multierror@v1.1.1)
+ go-multierror GitHub 源码：[https://github.com/hashicorp/go-multierror](https://github.com/hashicorp/go-multierror)
+ Go 错误处理指北：如何优雅的处理错误？：[https://jianghushinian.cn/2024/10/01/go-error-guidelines-error-handling/](https://jianghushinian.cn/2024/10/01/go-error-guidelines-error-handling/)
+ Go 并发控制：sync.WaitGroup 详解：[https://jianghushinian.cn/2024/12/23/sync-waitgroup/](https://jianghushinian.cn/2024/12/23/sync-waitgroup/)
+ Go 并发控制：errgroup 详解：[https://jianghushinian.cn/2024/11/04/x-sync-errgroup/](https://jianghushinian.cn/2024/11/04/x-sync-errgroup/)
+ 开源协议简介：[https://jianghushinian.cn/2023/01/15/open-source-license-introduction/](https://jianghushinian.cn/2023/01/15/open-source-license-introduction/)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/error/go-multierror](https://github.com/jianghushinian/blog-go-example/tree/main/error/go-multierror)
+ 本文永久地址：[https://jianghushinian.cn/2025/03/23/go-multierror/](https://jianghushinian.cn/2025/03/23/go-multierror/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

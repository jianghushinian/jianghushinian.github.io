---
title: '在 Go 中使用 cron 执行定时任务'
date: 2025-02-23 23:22:30
tags:
- Go
categories:
- Go
---

如果你曾经在 Go 中实现过定时任务，可能会发现，原生的 `time.Timer` 或 `time.Ticker` 虽然简单易用，但在复杂的场景下（如多任务调度、时区处理、任务失败重试等）往往显得力不从心。这时，一个功能强大且灵活的定时任务库就显得尤为重要。

`github.com/robfig/cron` 正是为此而生。它不仅支持标准的 `crontab` 表达式，还提供了秒级精度、时区设置、任务链（Job Wrappers）等高级功能，能够轻松应对各种复杂的定时任务场景。

<!-- more -->

本文就来带大家见识下 `cron` 的用法。

### 快速开始

我们可以使用如下命令在 Go 项目中安装 `cron`：

```bash
$ go get github.com/robfig/cron/v3@master
```

接着我们来一起快速入门 `cron` 的使用，示例如下：

> [https://github.com/jianghushinian/blog-go-example/tree/main/cron/main.go](https://github.com/jianghushinian/blog-go-example/tree/main/cron/main.go)
>

```go
package main

import (
	"fmt"
	"log"
	"time"

	"github.com/robfig/cron/v3"
)

// NOTE: 基础用法

func main() {
	// 创建一个新的 Cron 实例
	c := cron.New(cron.WithSeconds())

	// 添加一个每秒执行的任务
	_, err := c.AddFunc("* * * * * *", func() {
		fmt.Println("每秒执行的任务:", time.Now().Format("2006-01-02 15:04:05"))
	})
	if err != nil {
		log.Fatalf("添加任务失败: %v", err)
	}

	// 添加一个每 5 秒执行的任务
	_, err = c.AddFunc("*/5 * * * * *", func() {
		fmt.Println("每 5 秒执行的任务:", time.Now().Format("2006-01-02 15:04:05"))
	})
	if err != nil {
		log.Fatalf("添加任务失败: %v", err)
	}

	// 启动 Cron
	c.Start()
	defer c.Stop() // 确保程序退出时停止 Cron

	// 主程序等待 10 秒，以便观察任务执行
	time.Sleep(10 * time.Second)
	fmt.Println("主程序结束")
}
```

这个示例展示了 `cron` 的基础用法，注册了两个定时任务到 `cron` 中，并在程序启动 10 秒后退出。

使用 `cron.New` 方法可以创建一个 `cron` 对象，`cron.WithSeconds()` 参数可以扩展 `crontab` 表达式语法支持到秒级别，语法规则不变。

`cron` 对象的 `AddFunc` 方法可以添加一个任务到 `cron` 中，它接收两个参数，第一个字符串类型的参数用来定义 `crontab` 表达式，`* * * * * *` 表示每秒执行一次，第二个参数 `func()` 就是我们要添加的任务。这里为 `cron` 对象添加了两个任务。

`cron` 对象的 `Start` 方法内部会创建一个新的 goroutine 并启动 `cron` 调度器，调度器可以控制所有注册进来的任务的执行，在任务的执行计划到期时运行任务。

`cron` 对象的 `Stop` 方法可以停止调度器。

执行上述示例，得到输出如下：

```bash
$ go run main.go
每秒执行的任务: 2025-02-23 16:02:41
每秒执行的任务: 2025-02-23 16:02:42
每秒执行的任务: 2025-02-23 16:02:43
每秒执行的任务: 2025-02-23 16:02:44
每秒执行的任务: 2025-02-23 16:02:45
每 5 秒执行的任务: 2025-02-23 16:02:45
每秒执行的任务: 2025-02-23 16:02:46
每秒执行的任务: 2025-02-23 16:02:47
每秒执行的任务: 2025-02-23 16:02:48
每秒执行的任务: 2025-02-23 16:02:49
每秒执行的任务: 2025-02-23 16:02:50
每 5 秒执行的任务: 2025-02-23 16:02:50
主程序结束
```

可以发现，我们注册到 `cron` 中的两个任务都生效了，第一个任务每秒执行一次，第二个任务每 5 秒执行一次，10 秒后程序结束并退出。

### 进阶用法

上面示例中，我们快速入门了 `cron` 的用法，下面我再将 `cron` 的常用功能进行进一步讲解。

#### 执行计划时间格式

`cron` 包支持标准的 `crontab` 表达式，并且可以使用 `cron.WithSeconds()` 将其精度扩展到秒级别。

并且，`cron` 还支持另一种表达式：

```bash
@every <duration>
```

这是一种固定时间间隔的表达式，`@every` 是固定写法，中间一个空格，然后是 `<duration>`，`<duration>` 可以是任意 `time.ParseDuration` 可解析的字符串。比如 `@every 1h30m10s` 表示任务每隔 1 小时 30 分 10 秒执行一次，`@every 5s` 表示任务每隔 5 秒执行一次。

#### 任务对象

`cron` 包不仅支持注册函数类型的任务，其实我们可以注册任意自定义类型的任务，只需要实现 `cron.Job` 接口即可：

```go
type Job interface {
	Run()
}
```

比如我们自定义了 `Job` 类型作为一个任务对象：

```go
// Job 作业对象
type Job struct {
	name  string
	count int
}

func (j *Job) Run() {
	j.count++
	if j.count == 2 {
		panic("第 2 次执行触发 panic")
	}
	if j.count == 4 {
		time.Sleep(6 * time.Second)
		log.Println("第 4 次执行耗时 6s")
	}
	fmt.Printf("%s: 每 5 秒执行的任务, count: %d\n", j.name, j.count)
}
```

可以使用如下方式，将其注册到 cron 中：

```go
var (
    spec = "@every 5s"
    job  = &Job{name: "江湖十年"}
)

c := cron.New(cron.WithSeconds())
c.AddJob(spec, job)
```

`cron` 对象的调度器会每隔 5 秒执行一次 `job.Run` 方法。

#### 自定义日志

`cron` 支持注册自定义的日志对象，这样 `cron` 运行期间产生的日志都会输出到我们自定义的日志中。

自定义日志之需要实现 `cron.Logger` 接口即可：

```go
type Logger interface {
	// Info logs routine messages about cron's operation.
	Info(msg string, keysAndValues ...interface{})
	// Error logs an error condition.
	Error(err error, msg string, keysAndValues ...interface{})
}
```

如下是我们自定义的日志对象：

```go
// 自定义 logger
type cronLogger struct{}

func newCronLogger() *cronLogger {
	return &cronLogger{}
}

// Info implements the cron.Logger interface.
func (l *cronLogger) Info(msg string, keysAndValues ...any) {
	slog.Info(msg, keysAndValues...)
}

// Error implements the cron.Logger interface.
func (l *cronLogger) Error(err error, msg string, keysAndValues ...any) {
	slog.Error(msg, append(keysAndValues, "err", err)...)
}
```

这里为了演示，仅将日志转发给 `slog` 进行输出，你可以定制更多高级功能。

可以这样使用日志对象：

```go
// 创建自定义日志对象
logger := &cronLogger{}

// 创建一个新的 Cron 实例
c := cron.New(
    cron.WithSeconds(),      // 增加秒解析
    cron.WithLogger(logger), // 自定义日志
)
```

#### 任务装饰器

我们可以在初始化 `cron` 时，为任务定义一系列的装饰器，这样当有新的任务注册进来，就能自动为其附加装饰器的所有功能。

`cron` 为我们提供了几个自带的装饰器：

+ `cron.Recover`：恢复任务执行过程中产生的 `panic`，不要让 `cron` 调度器退出。
+ `cron.DelayIfStillRunning`：如果上一次任务还未完成，那么延迟此次任务的执行时间，只有上一次任务执行完成后，才会执行下一次任务。
+ `cron.SkipIfStillRunning`：如果上一次任务还未完成，那么跳过此次任务的执行。

我们可以像这样使用任务装饰器：

```go
// 创建自定义日志对象
logger := &cronLogger{}

// 创建一个新的 Cron 实例
c := cron.New(
    cron.WithSeconds(),      // 增加秒解析
    cron.WithLogger(logger), // 自定义日志
    cron.WithChain( // chain 是顺序敏感的
        cron.SkipIfStillRunning(logger), // 如果作业仍在运行，则跳过此次运行
        cron.Recover(logger),            // 恢复 panic
    ),
)
```

`cron.WithChain` 会将所有装饰器串联起来，使其成为一条任务链，比如 `cron.WithChain(m1, m2)`，那么最终任务执行时会这样调用：`m1(m2(job))`。

我们也可以实现自定义的装饰器，其函数定义格式为：`func(cron.Job) cron.Job`，你可以试着自己实现一个。

#### 进阶示例

学习了 `cron` 的进阶使用，我们可以编写一个示例，体验下这些功能的用法：

> [https://github.com/jianghushinian/blog-go-example/tree/main/cron/main.go](https://github.com/jianghushinian/blog-go-example/tree/main/cron/main.go)
>

```go
// 自定义 logger
type cronLogger struct{}

func newCronLogger() *cronLogger {
	return &cronLogger{}
}

// Info implements the cron.Logger interface.
func (l *cronLogger) Info(msg string, keysAndValues ...any) {
	slog.Info(msg, keysAndValues...)
}

// Error implements the cron.Logger interface.
func (l *cronLogger) Error(err error, msg string, keysAndValues ...any) {
	slog.Error(msg, append(keysAndValues, "err", err)...)
}

// Job 作业对象
type Job struct {
	name  string
	count int
}

func (j *Job) Run() {
	j.count++
	if j.count == 2 {
		panic("第 2 次执行触发 panic")
	}
	if j.count == 4 {
		time.Sleep(6 * time.Second)
		log.Println("第 4 次执行耗时 6s")
	}
	fmt.Printf("%s: 每 5 秒执行的任务, count: %d\n", j.name, j.count)
}

func main() {
	log.Println("cron start")
	// 创建自定义日志对象
	logger := &cronLogger{}

	// 创建一个新的 Cron 实例
	c := cron.New(
		cron.WithSeconds(),      // 增加秒解析
		cron.WithLogger(logger), // 自定义日志
		cron.WithChain( // chain 是顺序敏感的
			cron.SkipIfStillRunning(logger), // 如果作业仍在运行，则跳过此次运行
			cron.Recover(logger),            // 恢复 panic
		),
	)

	var (
		spec = "@every 5s"
		job  = &Job{name: "江湖十年"}
	)

	// 添加一个每 5 秒执行的任务
	id, err := c.AddJob(spec, job)
	if err != nil {
		log.Fatalf("添加任务失败: %v", err)
	}
	log.Println("任务 ID:", id)

	// 启动 Cron
	c.Start()
	defer c.Stop() // 确保程序退出时停止 Cron

	time.Sleep(34 * time.Second) // 确保 job 能执行 6 次
	c.Remove(id)                 // 从调度器中移除 job
	time.Sleep(10 * time.Second) // job 不会再次执行
	log.Println("cron done")
}
```

这个示例完整的展示了进阶用法的几项功能。

执行上述示例，得到输出如下：

```bash
$ go run main.go
2025/02/23 16:03:47 cron start
2025/02/23 16:03:47 任务 ID: 1
2025/02/23 16:03:47 INFO start
2025/02/23 16:03:47 INFO schedule now=2025-02-23T16:03:47.035+08:00 entry=1 next=2025-02-23T16:03:52.000+08:00
2025/02/23 16:03:52 INFO wake now=2025-02-23T16:03:52.000+08:00
2025/02/23 16:03:52 INFO run now=2025-02-23T16:03:52.000+08:00 entry=1 next=2025-02-23T16:03:57.000+08:00
江湖十年: 每 5 秒执行的任务, count: 1
2025/02/23 16:03:57 INFO wake now=2025-02-23T16:03:57.001+08:00
2025/02/23 16:03:57 INFO run now=2025-02-23T16:03:57.001+08:00 entry=1 next=2025-02-23T16:04:02.000+08:00
2025/02/23 16:03:57 ERROR panic stack="..." err="第 2 次执行触发 panic"
2025/02/23 16:04:02 INFO wake now=2025-02-23T16:04:02.000+08:00
2025/02/23 16:04:02 INFO run now=2025-02-23T16:04:02.000+08:00 entry=1 next=2025-02-23T16:04:07.000+08:00
江湖十年: 每 5 秒执行的任务, count: 3
2025/02/23 16:04:07 INFO wake now=2025-02-23T16:04:07.000+08:00
2025/02/23 16:04:07 INFO run now=2025-02-23T16:04:07.000+08:00 entry=1 next=2025-02-23T16:04:12.000+08:00
2025/02/23 16:04:12 INFO wake now=2025-02-23T16:04:12.000+08:00
2025/02/23 16:04:12 INFO run now=2025-02-23T16:04:12.000+08:00 entry=1 next=2025-02-23T16:04:17.000+08:00
2025/02/23 16:04:12 INFO skip
2025/02/23 16:04:13 第 4 次执行耗时 6s
江湖十年: 每 5 秒执行的任务, count: 4
2025/02/23 16:04:17 INFO wake now=2025-02-23T16:04:17.001+08:00
江湖十年: 每 5 秒执行的任务, count: 5
2025/02/23 16:04:17 INFO run now=2025-02-23T16:04:17.001+08:00 entry=1 next=2025-02-23T16:04:22.000+08:00
2025/02/23 16:04:21 INFO removed entry=1
2025/02/23 16:04:31 cron done
```

这里日志比较多，不过如果你耐心梳理一下，还是比较清晰的。

可以看到日志中输出了 `第 2 次执行触发 panic`（日志行省略了 `stack` 信息），程序并没有退出，说明我们使用的 `cron.Recover(logger) ` 装饰器生效了。

并且，在打印 `第 4 次执行耗时 6s` 之前，上一行日志输出了 `INFO skip`，表示跳过了一次任务执行，说明 `cron.SkipIfStillRunning(logger)` 装饰器生效了。

我们使用 `c.Remove(id)` 从调度器中移除了注册的任务后，任务没再执行。

好了，`cron` 的使用示例就讲解到这里。

我需要额外强调的一点是，其实 `cron` 项目存在一个 Bug，在作者的最后一次 [commit](https://github.com/robfig/cron/commit/bc59245fe10efaed9d51b56900192527ed733435) 中被修复。不过作者并没有为其打上新的 Tag，所以这也是为什么安装时我们需要指定 `@master` 来下载最新版本。

这个 Bug 修复如下：

![cron-bug](cron-bug.png)

如果你在安装 `cron` 时没有指定 `@master`，你可以尝试修改下 `cron.SkipIfStillRunning` 和 `cron.Recover` 的顺序，来看看程序执行效果。

### 原理剖析

`cron` 的常见用法我们都讲解完成了，如果你想更深入的了解 `cron`，那么我这里为你简单梳理下 `cron` 的整体设计，方便你进一步学习。

首先，我为你画了一张思维导图，展示了 `cron` 对象提供的所有 exported 方法：

![cron](cron.png)

我们最需要关注的就是创建、启动、添加任务（图中写为“作业”）和停止功能。

其中 `Cron` 是一个结构体，用于管理和调度注册进来的任务，其定义如下：

```go
// Cron 核心结构体，用于调度注册进来的作业
// 记录作业列表（entries），可以启动、停止，并且检查运行中的作业状态
type Cron struct {
	entries   []*Entry          // 作业对象列表
	chain     Chain             // 装饰器链
	stop      chan struct{}     // 停止信号
	add       chan *Entry       // Cron 运行时，增加作业的 channel
	remove    chan EntryID      // 移除指定 ID 作业的 channel
	snapshot  chan chan []Entry // 获取当前作业列表快照的 channel
	running   bool              // 标识 Cron 是否正在运行
	logger    Logger            // 日志对象，Cron 会将运行的日志内容输出到 logger
	runningMu sync.Mutex        // 当 Cron 运行时，保护并发操作的锁
	location  *time.Location    // 本地时区，Cron 根据此时区计算任务执行计划
	parser    ScheduleParser    // 任务执行计划解析器
	nextID    EntryID           // 下一个要执行的作业 ID
	jobWaiter sync.WaitGroup    // 使用 wg 等待作业完成
}
```

所有添加到 `Cron` 中的任务都会保存在 `entries` 列表中。

`Entry` 结构体表示一个任务对象，定义如下：

```go
// EntryID 标识 Cron 实例中的一个作业
type EntryID int

// Entry 作业实体对象，表示一个被注册进 Cron 执行器中的作业
type Entry struct {
	// ID 是作业的唯一 ID，可用于查找快照或将其删除
	ID EntryID
	// Schedule 作业的执行计划，应该按照此计划来执行作业
	Schedule Schedule
	// Next 下次运行作业的时间，如果 Cron 尚未启动或无法满足此作业的执行计划，则为 zero time
	Next time.Time
	// Prev 是此作业的最后一次运行时间，如果从未运行，则为 zero time
	Prev time.Time
	// WrappedJob 作业装饰器，为作业增加新的功能，会在 Schedule 被激活时运行
	WrappedJob Job
	// Job 提交到 Cron 中的作业
	Job Job
}
```

`EntryID` 就是我们调用 `c.Remove(id)` 时使用的 `id`。

`Cron` 最核心的逻辑当然是调度逻辑，其功能主要由 `for...select` 实现：

```go
for {
    select {
    case now = <-timer.C:
        // 运行下一次执行时间小于当前时间的所有作业
    case newEntry := <-c.add:
        // 有新的作业加入进来，加入调度
    case replyChan := <-c.snapshot:
        // 获取当前作业列表
    case <-c.stop:
        // 停止调度
    case id := <-c.remove:
        // 移除指定 ID 的作业
}
```

这里是调度器主要逻辑，`for...select` 能够实现高效调度，并且结合 exported 方法中的互斥锁，`cron` 能够支持并发操作。

以上，就是 `cron` 的设计原理。当然其具体内部实现还需要你自行研究，你可以参考我写好了中文注释的源码 [https://github.com/jianghushinian/blog-go-example/tree/main/cron/sourcecode/cron](https://github.com/jianghushinian/blog-go-example/tree/main/cron/sourcecode/cron)。

### 总结

`github.com/robfig/cron` 解决了复杂场景下在 Go 中执行定时任务的需求。

`cron` 包非常强大，它扩展了 `crontab` 表达式，支持秒级精度。任务链的功能可以为任务附加功能，这个设计跟 Gin 框架的中间件非常相似，你可以类比学习。此外我认为 `cron` 包设计比较好的一点是支持自定义日志包，这样我们可以按照自己的方式收集 `cron` 的执行日志。想要更深入的学习 `cron` 包，可以参考我画的思维导图进一步阅读其源码。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/cron) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ cron wiki: [https://en.wikipedia.org/wiki/Cron](https://en.wikipedia.org/wiki/Cron)
+ cron Documentation：[https://pkg.go.dev/github.com/robfig/cron](https://pkg.go.dev/github.com/robfig/cron)
+ cron GitHub 源码：[https://github.com/robfig/cron](https://github.com/robfig/cron)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/cron](https://github.com/jianghushinian/blog-go-example/tree/main/cron)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

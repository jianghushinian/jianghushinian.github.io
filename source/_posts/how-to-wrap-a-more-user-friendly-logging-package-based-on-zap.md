---
title: 如何基于 zap 封装一个更好用的日志库
date: 2023-04-16 09:51:08
tags:
- Go
- Log
categories:
- Go
---

在今天的软件开发中，日志对于定位和解决问题至关重要。Go 社区有许多优秀的日志库供我们选择，其中有一款来自 Uber 公司的开源 Go 语言日志库 —— zap，非常流行，且以快著称。但与此同时，相较于诸如 Go log 标准库、Logrus 第三方日志库等，zap 在使用上就没有那么直观和舒适了。因此，在本文中，我们将深入探讨如何基于 zap 日志库封装一个更易用、更实用的日志工具，从而帮助开发者更轻松地管理日志，提高工作效率。

<!-- more -->

> 笔记：本文是对[《Go 第三方 log 库之 zap 使用》](https://jianghushinian.cn/2023/03/19/use-of-zap-in-go-third-party-log-library/)一文的填坑，如果你还没有看过这篇文章，强烈建议看完后再来阅读此篇文章。

### zap 使用示例

现在我们想打印一条日志到控制台。

使用 zap 实现方式如下：

```go
package main

import "go.uber.org/zap"

func main() {
	logger, _ := zap.NewProduction()
	defer logger.Sync()
	logger.Info("log info")
}
```

执行以上代码会输出一条 `Info` 级别的日志到标准错误输出 `stderr`。

如果使用 Go log 标准库实现，则可以这么写：

```go
package main

import "log"

func main() {
	log.Print("log info")
}
```

执行以上代码同样会输出一条日志到标准错误输出 `stderr`。

虽然 Go log 标准库没有日志级别的概念，但 zap 需要三行代码才能实现的功能，Go log 标准库只需要一行代码就可以，使用体验更好。

再比如，我们想设置日志级别。

使用 zap 实现方式如下：

```go
package main

import "go.uber.org/zap"

func main() {
	config := zap.NewProductionConfig()
	config.Level = zap.NewAtomicLevelAt(zap.ErrorLevel)
	logger, _ := config.Build()
	defer logger.Sync()
	logger.Error("log error")
}   
```

在 zap 中想设置日志级别，首先需要先构造一个 `zap.Config` 对象 `config`，然后更改 `config` 的日志级别属性 `Level` 的值，再通过 `config.Build()` 构建 `zap.Logger` 对象，之后才能使用。

而在 Logrus 日志库中，则只需要一行代码即可实现，使用 `logrus.SetLevel` 方法即可完成。

```go
package main

import "github.com/sirupsen/logrus"

func main() {
	logrus.SetLevel(logrus.ErrorLevel)
	logrus.Error("log error")
}
```

以上两个简单的示例，足以体现 zap 使用门槛相对来说的确更高一些。

更多关于 zap 的使用方式，可以参考[《Go 第三方 log 库之 zap 使用》](https://jianghushinian.cn/2023/03/19/use-of-zap-in-go-third-party-log-library/)一文。

### 封装 zap

上面演示了 Go log 标准库开箱即用的使用体验，以及 Logrus 日志库提供的方便快捷 API。接下来我们要对 zap 日志库进行封装改造，使其更加好用。

#### 定义默认日志对象

Go log 标准库是通过定义了一个默认日志对象 `std`，来实现开箱即用的效果。我们这里就模仿 Go log 标准库来对 zap 进行封装。

> https://github.com/jianghushinian/gokit/blob/main/log/zap/log.go

```go
package zap

import (
	"io"
	"os"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

type Level = zapcore.Level

const (
	DebugLevel = zapcore.DebugLevel
	InfoLevel  = zapcore.InfoLevel
	WarnLevel  = zapcore.WarnLevel
	ErrorLevel = zapcore.ErrorLevel
	PanicLevel = zapcore.PanicLevel
	FatalLevel = zapcore.FatalLevel
)

type Logger struct {
	l *zap.Logger
	al *zap.AtomicLevel
}

func New(out io.Writer, level Level) *Logger {
	if out == nil {
		out = os.Stderr
	}

	al := zap.NewAtomicLevelAt(level)
	cfg := zap.NewProductionEncoderConfig()
	cfg.EncodeTime = zapcore.RFC3339TimeEncoder

	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(cfg),
		zapcore.AddSync(out),
		al,
	)
	return &Logger{l: zap.New(core), al: &al}
}

func (l *Logger) SetLevel(level Level) {
	if l.al != nil {
		l.al.SetLevel(level)
	}
}

type Field = zap.Field

func (l *Logger) Debug(msg string, fields ...Field) {
	l.l.Debug(msg, fields...)
}

func (l *Logger) Info(msg string, fields ...Field) {
	l.l.Info(msg, fields...)
}

func (l *Logger) Warn(msg string, fields ...Field) {
	l.l.Warn(msg, fields...)
}

func (l *Logger) Error(msg string, fields ...Field) {
	l.l.Error(msg, fields...)
}

func (l *Logger) Panic(msg string, fields ...Field) {
	l.l.Panic(msg, fields...)
}

func (l *Logger) Fatal(msg string, fields ...Field) {
	l.l.Fatal(msg, fields...)
}

func (l *Logger) Sync() error {
	return l.l.Sync()
}

var std = New(os.Stderr, InfoLevel)

func Default() *Logger         { return std }
func ReplaceDefault(l *Logger) { std = l }

func SetLevel(level Level) { std.SetLevel(level) }

func Debug(msg string, fields ...Field) { std.Debug(msg, fields...) }
func Info(msg string, fields ...Field)  { std.Info(msg, fields...) }
func Warn(msg string, fields ...Field)  { std.Warn(msg, fields...) }
func Error(msg string, fields ...Field) { std.Error(msg, fields...) }
func Panic(msg string, fields ...Field) { std.Panic(msg, fields...) }
func Fatal(msg string, fields ...Field) { std.Fatal(msg, fields...) }

func Sync() error { return std.Sync() }
```

如果你看过我写的[《深入探究 Go log 标准库》](https://jianghushinian.cn/2023/03/11/dive-into-the-go-log-standard-library/)一文，那么对这份代码一定会非常熟悉，想必不用我讲也能过理解其含义，这份代码完全参考了 Go log 标准库的设计思路。

首先为了使用方便，我为 `zapcore.Level` 类型定义了别名 `Level`，这样用户在使用我们封装的 zap 包设置日志级别时，就只需要引入封装好的日志包，而无需引入原始的 zap 包了。

然后我定义了 `Logger` 结构体，用来表示日志对象。它只包含两个字段，分别是 `*zap.Logger` 对象和日志级别 `*zap.AtomicLevel`（zap 通过 `zap.AtomicLevel` 操作 `zapcore.Level` 来保证操作的原子性）。

通过 `New` 函数可以构造一个 `Logger` 对象，`New` 函数接收两个参数分别用来设置日志输出位置和日志级别。

同样的为了使用方便，我还为 `zap.Field` 类型定义了别名 `Field`，并将所有 zap 中定义的类型都拷贝到 `field.go` 中。

> https://github.com/jianghushinian/gokit/blob/main/log/zap/field.go

```go
package zap

import "go.uber.org/zap"

var (
	Skip        = zap.Skip
	Binary      = zap.Binary
	Bool        = zap.Bool
	...
)
```

接下来为 `Logger` 结构体定义了 `Debug`、`Info` 等日志输出方法，这些方法也仅是对 `zap.Logger` 对象对应方法的一层包装。

然后就到了定义默认日志对象的环节，通过 `var std = New(os.Stderr, InfoLevel)` 我们定义了 `std` 日志对象，尽管它是不可导出的变量，但我们实现了 `Debug`、`Info` 等公开函数，其内部正是调用了 `std` 对应的方法，完成日志输出。

我们可以按照如下方式，使用这个封装后的 zap 包。

```go
package main

import (
	log "github.com/jianghushinian/gokit/log/zap"
)

func main() {
	defer log.Sync()
	log.Info("Info msg")

	log.SetLevel(log.ErrorLevel)
	log.Info("Info msg")
	log.Error("Error msg")
}
```

执行示例代码后，得到如下输出：

```log
{"level":"info","ts":"2023-04-16T16:08:01+08:00","msg":"Info msg"}
{"level":"error","ts":"2023-04-16T16:08:01+08:00","msg":"Error msg"}
```

可以发现，我们实现了像 Go log 标准库一样的开箱即用效果。在使用前，不再需要实例化一个 `zap.Logger` 对象，而是可以直接调用包级别的 `Info` 函数输出日志。

并且我们可以只使用一行代码 `log.SetLevel(log.ErrorLevel)`，将日志级别设置为 `Error`。

用户也可以通过 `New` 函数来构造自己的 `Logger` 对象。

```go
package main

import (
	"os"

	log "github.com/jianghushinian/gokit/log/zap"
)

func main() {
	logger := log.New(os.Stderr, log.ErrorLevel)
	defer logger.Sync()
	logger.Info("Info msg")
	logger.Error("Error msg")
}
```

此外，代码中还提供了 `ReplaceDefault` 函数，供用户替换默认的 `std` 对象，这样用户在构造自己的 `Logger` 对象后，仍然可以使用包级别的日志函数。

```go
package main

import (
	"os"

	log "github.com/jianghushinian/gokit/log/zap"
)

func main() {
	logger := log.New(os.Stderr, log.ErrorLevel)
	log.ReplaceDefault(logger)
	defer log.Sync()
	log.Info("Info msg")
	log.Error("Error msg")
}
```

#### 指定 Encoder

上面介绍的 `New` 函数定义如下：

```go
func New(out io.Writer, level Level) *Logger {
	if out == nil {
		out = os.Stderr
	}

	al := zap.NewAtomicLevelAt(level)
	cfg := zap.NewProductionEncoderConfig()
	cfg.EncodeTime = zapcore.RFC3339TimeEncoder

	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(cfg),
		zapcore.AddSync(out),
		al,
	)
	return &Logger{l: zap.New(core), al: &al}
}
```

其内部通过调用 `zapcore.NewCore` 获得一个 `zapcore.Core` 对象，这是 zap 日志库的核心对象，将它传递给 `zap.New` 就可以拿到 `zap.Logger` 对象。

`zapcore.NewCore` 接收三个参数，`Encoder`、`WriteSyncer`、`LevelEnabler`，其功能如下：

- `Encoder`: 编码器，用来定义日志的输出格式。

- `WriteSyncer`: 指定日志输出位置。

- `LevelEnabler`: 指定日志级别。

这三个参数，正是用来控制一个日志库的核心功能。

其中，日志输出位置和日志级别都是通过函数参数传递进来的，而编码器则是固定的。我们首先通过 `zap.NewProductionEncoderConfig()` 拿到一个编码器配置，然后使用 `cfg.EncodeTime = zapcore.RFC3339TimeEncoder` 指定时间格式化为 `RFC3339` 格式，最后通过 `zapcore.NewJSONEncoder(cfg)` 的形式构造了一个 JSON 格式的 `Encoder` 并传递给 `zapcore.NewCore`。

最终，我们得到的日志格式长这样：

```log
{"level":"info","ts":"2023-04-16T16:08:01+08:00","msg":"Info msg"}
```

这里 `Encoder` 之所没有当作参数传递进来，是因为我想定义一个规范，使得引入此日志库的项目所打印出来的日志格式是一致的。这在微服务项目开发中尤其有效，保证了各个模块间日志格式统一，方便收集、解析、和排查问题。

#### 支持日志选项

zap 在使用 `zap.NewProduction()` 创建 `logger` 时，其实是支持选项参数的：

```go
package main

import "go.uber.org/zap"

func main() {
	logger, _ := zap.NewProduction(zap.WithCaller(false))
	defer logger.Sync()
	logger.Info("log info")
}
```

以上示例代码中，我们就通过 `zap.NewProduction(zap.WithCaller(false))` 的方式关闭了输出日志时携带函数调用信息的功能。

zap 支持的所有选项你可以在[这里](https://pkg.go.dev/go.uber.org/zap#Option)查看。

所以我们封装的日志包也要支持选项功能。

定义 `options.go` 如下：

> https://github.com/jianghushinian/gokit/blob/main/log/zap/options.go

```go
package zap

import "go.uber.org/zap"

type Option = zap.Option

var (
	WrapCore      = zap.WrapCore
	Hooks         = zap.Hooks
	Fields        = zap.Fields
	ErrorOutput   = zap.ErrorOutput
	Development   = zap.Development
	AddCaller     = zap.AddCaller
	WithCaller    = zap.WithCaller
	AddCallerSkip = zap.AddCallerSkip
	AddStacktrace = zap.AddStacktrace
	IncreaseLevel = zap.IncreaseLevel
	WithFatalHook = zap.WithFatalHook
	WithClock     = zap.WithClock
)
```

跟日志级别的做法相同，我为 `zap.Option` 定义了类型别名`Option`。

修改 `New` 函数定义如下：

```go
func New(out io.Writer, level Level, opts ...Option) *Logger {
	if out == nil {
		out = os.Stderr
	}

	al := zap.NewAtomicLevelAt(level)
	cfg := zap.NewProductionEncoderConfig()
	cfg.EncodeTime = zapcore.RFC3339TimeEncoder

	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(cfg),
		zapcore.AddSync(out),
		al,
	)
	return &Logger{l: zap.New(core, opts...), al: &al}
}
```

改动很小，只需要加上可选参数 `opts` 并将其原样传给 `zap.New` 就实现了选项功能的支持。

现在，可以按照如下方式开启日志包记录函数调用信息功能：

```go
package main

import (
	"os"

	log "github.com/jianghushinian/gokit/log/zap"
)

func main() {
	logger := log.New(os.Stderr, log.InfoLevel, log.AddCaller(), log.AddCallerSkip(1))
	defer logger.Sync()
	logger.Info("Info msg")
}
```

其中 `log.AddCaller()` 选项用来开启记录，`log.AddCallerSkip(1)` 用来设置通过调用栈获取文件名和行号时跳过的调用深度。

执行以上示例代码，将得到如下日志输出：

```log
{"level":"info","ts":"2023-04-16T17:27:11+08:00","caller":"main.go:12","msg":"Info msg"}
```

#### 支持将不同级别日志输出到不同位置

有时候，为了方便对不同级别日志进行分开管理，我们可能想要将不同级别的日志输出到不同位置。

在 zap 中可以通过 `zapcore.NewTee()` 实现，它返回一个切片 `[]zapcore.Core`，这样每一个 `zapcore.Core` 对应一种日志级别，就能够实现将不同级别日志输出到不同位置了。

定义 `tee.go` 如下：

> https://github.com/jianghushinian/gokit/blob/main/log/zap/tee.go

```go
package zap

import (
	"io"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

type LevelEnablerFunc func(Level) bool

type TeeOption struct {
	Out io.Writer
	LevelEnablerFunc
}

func NewTee(tees []TeeOption, opts ...Option) *Logger {
	var cores []zapcore.Core
	for _, tee := range tees {
		cfg := zap.NewProductionEncoderConfig()
		cfg.EncodeTime = zapcore.RFC3339TimeEncoder
		core := zapcore.NewCore(
			zapcore.NewJSONEncoder(cfg),
			zapcore.AddSync(tee.Out),
			zap.LevelEnablerFunc(tee.LevelEnablerFunc),
		)
		cores = append(cores, core)
	}
	return &Logger{l: zap.New(zapcore.NewTee(cores...), opts...)}
}
```

我们为这种情况，专门定义了一个 `NewTee` 函数来构造 `Logger` 对象。

它接收一个 `tees []TeeOption` 参数，其中 `TeeOption` 包含两个属性，分别是日志输出位置和日志级别，当满足定义的日志级别时将日志输出到指定位置。

这里的日志级别是一个函数而不是常量，这样可以增加灵活性，只要函数返回值为 `true` 就会记录日志。

这样，通过定义如下函数，可以实现只有 `Info` 级别才会记录日志：

```go
func (level log.Level) bool {
	return level == log.InfoLevel
}
```

而如下函数的定义，则可以实现 `Info` 及以上级别日志都会记录：

```go
func (level log.Level) bool {
	return level >= log.InfoLevel
}
```

使用示例如下：

```go
package main

import (
	"os"

	log "github.com/jianghushinian/gokit/log/zap"
)

func main() {
	file, _ := os.OpenFile("test-warn.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
	tees := []log.TeeOption{
		{
			Out: os.Stdout,
			LevelEnablerFunc: func(level log.Level) bool {
				return level == log.InfoLevel
			},
		},
		{
			Out: file,
			LevelEnablerFunc: func(level log.Level) bool {
				return level == log.WarnLevel
			},
		},
	}
	logger := log.NewTee(tees)
	defer logger.Sync()

	logger.Info("Info tee msg")
	logger.Warn("Warn tee msg")
	logger.Error("Error tee msg") // 不会输出
}
```

执行以上示例代码，控制台输出 `Info` 级别日志：

```go
{"level":"info","ts":"2023-04-16T17:46:35+08:00","msg":"Info tee msg"}
```

`test-warn.log` 日志文件则输出 `Warn` 级别日志：

```go
{"level":"warn","ts":"2023-04-16T17:46:35+08:00","msg":"Warn tee msg"}
```

`Error` 级别日志由于不满足条件，不会被输出。

#### 日志轮转

日志轮转功能是一个日志库必不可少的功能，然而 zap 库本身其实并不支持日志轮转，我们可以借助 `file-rotatelogs` 和 `lumberjack` 第三方库来实现。

定义 `rotate.go` 如下：

> https://github.com/jianghushinian/gokit/blob/main/log/zap/rotate.go

```go
package zap

import (
	"io"
	"strings"
	"time"

	rotatelogs "github.com/lestrrat-go/file-rotatelogs"
	"gopkg.in/natefinch/lumberjack.v2"
)

type RotateConfig struct {
	// 共用配置
	Filename string // 完整文件名
	MaxAge   int    // 保留旧日志文件的最大天数

	// 按时间轮转配置
	RotationTime time.Duration // 日志文件轮转时间

	// 按大小轮转配置
	MaxSize    int  // 日志文件最大大小（MB）
	MaxBackups int  // 保留日志文件的最大数量
	Compress   bool // 是否对日志文件进行压缩归档
	LocalTime  bool // 是否使用本地时间，默认 UTC 时间
}

// NewProductionRotateByTime 创建按时间轮转的 io.Writer
func NewProductionRotateByTime(filename string) io.Writer {
	return NewRotateByTime(NewProductionRotateConfig(filename))
}

// NewProductionRotateBySize 创建按大小轮转的 io.Writer
func NewProductionRotateBySize(filename string) io.Writer {
	return NewRotateBySize(NewProductionRotateConfig(filename))
}

func NewProductionRotateConfig(filename string) *RotateConfig {
	return &RotateConfig{
		Filename: filename,
		MaxAge:   30, // 日志保留 30 天

		RotationTime: time.Hour * 24, // 24 小时轮转一次

		MaxSize:    100, // 100M
		MaxBackups: 100,
		Compress:   true,
		LocalTime:  false,
	}
}

func NewRotateByTime(cfg *RotateConfig) io.Writer {
	opts := []rotatelogs.Option{
		rotatelogs.WithMaxAge(time.Duration(cfg.MaxAge) * time.Hour * 24),
		rotatelogs.WithRotationTime(cfg.RotationTime),
		rotatelogs.WithLinkName(cfg.Filename),
	}
	if !cfg.LocalTime {
		rotatelogs.WithClock(rotatelogs.UTC)
	}
	filename := strings.SplitN(cfg.Filename, ".", 2)
	l, _ := rotatelogs.New(
		filename[0]+".%Y-%m-%d-%H-%M-%S."+filename[1],
		opts...,
	)
	return l
}

func NewRotateBySize(cfg *RotateConfig) io.Writer {
	return &lumberjack.Logger{
		Filename:   cfg.Filename,
		MaxSize:    cfg.MaxSize,
		MaxAge:     cfg.MaxAge,
		MaxBackups: cfg.MaxBackups,
		LocalTime:  cfg.LocalTime,
		Compress:   cfg.Compress,
	}
}
```

我们使用 [file-rotatelogs](github.com/lestrrat-go/file-rotatelogs) 包来支持按照时间轮转日志，使用 [lumberjack](https://github.com/natefinch/lumberjack) 包来支持按照日志文件大小轮转日志。

定义 `RotateConfig` 结构体用来配置日志轮转条件，`NewProductionRotateByTime` 函数返回一个可以按时间轮转的 `io.Writer`，`NewProductionRotateBySize` 函数则返回一个可以按日志文件大小轮转的 `io.Writer`。拿到 `io.Writer` 对象，就可以当作日志输出传递给 `New` 函数了。

我们可以结合 `NewTee` 来使用日志轮转功能，示例如下：

```go
package main

import (
	log "github.com/jianghushinian/gokit/log/zap"
)

func main() {
	tees := []log.TeeOption{
		{
			Out: log.NewProductionRotateBySize("rotate-by-size.log"),
			LevelEnablerFunc: log.LevelEnablerFunc(func(level log.Level) bool {
				return level < log.WarnLevel
			}),
		},
		{
			Out: log.NewProductionRotateByTime("rotate-by-time.log"),
			LevelEnablerFunc: log.LevelEnablerFunc(func(level log.Level) bool {
				return level >= log.WarnLevel
			}),
		},
	}
	lts := log.NewTee(tees)
	defer lts.Sync()

	lts.Debug("Debug msg")
	lts.Info("Info msg")
	lts.Warn("Warn msg")
	lts.Error("Error msg")
}
```

此示例将 `Warn` 以下级别日志按大小轮转，`Warn` 及以上级别日志按时间轮转。你可以自己执行以上示例代码，观察日志输出结果。

### 总结

本文算是一个填坑，我在[《Go 第三方 log 库之 zap 使用》](https://jianghushinian.cn/2023/03/19/use-of-zap-in-go-third-party-log-library/)一文中讲解了如何使用我们基于 zap 封装的日志库，本文讲解了这个日志库的设计思路。

主要思路借鉴了 Go log 标准库以及 Logrus 日志库，我们首先对比了 zap 日志库在使用时的劣势，然后根据另外两个日志库的优点，对 zap 进行了二次封装。

我们封装的日志包实现了开箱即用的效果，并且固定了日志输出格式，同时日志包还支持选项模式、将不同级别日志输出到不同位置，最后我还结合 `file-rotatelogs` 和 `lumberjack` 第三方库实现了日志轮转功能。

本文源码实现在[这里](https://github.com/jianghushinian/gokit/tree/main/log)，你可以点击链接进去查看。

**参考**

- 基于 zap 开发的日志库: https://github.com/jianghushinian/gokit/tree/main/log
- Go log 标准库: https://github.com/golang/go/tree/go1.20/src/log

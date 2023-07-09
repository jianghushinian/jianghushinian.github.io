---
title: Go 第三方 log 库之 zap 使用
date: 2023-03-19 09:43:30
tags:
- Go
- Log
categories:
- Go
---

zap 是由 Uber 公司开源的一款 Go 日志库，就像它的命名一样，zap 以快著称。官方 GitHub 仓库中只用一句话来概括 zap：「在 Go 中进行快速、结构化、分级的日志记录」。这句话简单明了的概括了 zap 的核心特性，今天我们就来介绍下 zap 日志库的基本使用和高级特性，以及如何在实际应用程序中使用，来提高应用程序的可靠性。

<!-- more -->

### 特点

zap 具有如下特点：

- 快，非常快，这也是 zap 最显著的特点。速度快的原因是 zap 避免使用 `interface{}` 和反射，并且使用 `sync.Pool` 减少堆内存分配。在 zap 面前 Logrus 的执行速度只有被吊打的份，你可以在官方 [GitHub 仓库](https://github.com/uber-go/zap#performance)中看到 zap 与不同日志库的速度对比。

- 支持结构化日志记录。这是一个优秀的日志库必备功能。

- 支持七种日志级别：`Debug`、`Info`、`Warn`、`Error`、`DPanic`、`Panic`、`Fatal`，其中 `DPanic` 是指在开发环境下（`development`）记录日志后会进行 `panic`。

- 支持输出调用堆栈。

- 支持 Hooks 机制。

### 使用

#### 基本使用

zap 基本使用方式如下：

```go
package main

import (
	"time"

	"go.uber.org/zap"
)

func main() {
	// 生产环境
	{
		logger, _ := zap.NewProduction()
		defer logger.Sync() // 刷新 buffer，保证日志最终会被输出

		url := "https://jianghushinian.cn/"
		logger.Info("production failed to fetch URL",
			zap.String("url", url), // 因为没有使用 interface{} 和反射机制，所以需要指定具体类型
			zap.Int("attempt", 3),
			zap.Duration("backoff", time.Second),
		)
	}

	// 开发环境
	{
		logger, _ := zap.NewDevelopment()
		defer logger.Sync()

		url := "https://jianghushinian.cn/"
		logger.Debug("development failed to fetch URL",
			zap.String("url", url),
			zap.Int("attempt", 3),
			zap.Duration("backoff", time.Second),
		)
	}
}
```

zap 针对生产环境和开发环境提供了不同的函数来创建 `Logger` 对象。

如果想在日志后面追加 key-value，则需要根据 value 的数据类型使用 `zap.String`、`zap.Int` 等方法实现。这一点在使用上显然不如 Logrus 等其他日志库来的方便，但这也是 zap 速度快的原因之一，zap 内部尽量避免使用 `interface{}` 和反射来提高代码执行效率。

记录日志的 `logger.Xxx` 方法签名如下：

```go
func (log *Logger) Info(msg string, fields ...Field)
```

其中 `fields` 是 `zapcore.Field` 类型，用来存储 key-value，并记录 value 类型，不管是 `zap.String` 还是 `zap.Int` 底层都是 `zapcore.Field` 类型来记录的。zap 为每一种 Go 的内置类型都定义了对应的 `zap.Xxx` 方法，甚至还实现 `zap.Any()` 来支持 `interface{}`。

执行以上代码，控制台得到如下输出：

```log
{"level":"info","ts":1679212318.10218,"caller":"zap/main.go:16","msg":"production failed to fetch URL","url":"https://jianghushinian.cn/","attempt":3,"backoff":1}
2023-03-19T15:51:58.102+0800    DEBUG   zap/main.go:29        development failed to fetch URL {"url": "https://jianghushinian.cn/", "attempt": 3, "backoff": "1s"}
```

可以发现，通过 `zap.NewProduction()` 创建的日志对象输出格式为 JSON，而通过 `zap.NewDevelopment()` 创建的日志对象输出格式为 Text，日志后面追加的 key-value 会被转换成 JSON。并且，两者输出的字段内容也略有差异，如生产环境日志输出的时间格式为 `Unix epoch` 利于程序解析，而开发环境日志输出的时间格式为 `ISO8601` 更利于人类阅读。

导致以上这些差异的原因是配置不同，我们来看下 `zap.NewProduction` 和 `zap.NewDevelopment` 的代码实现：

```go
func NewProduction(options ...Option) (*Logger, error) {
	return NewProductionConfig().Build(options...)
}

func NewProductionConfig() Config {
	return Config{
		Level:       NewAtomicLevelAt(InfoLevel),
		Development: false,
		Sampling: &SamplingConfig{
			Initial:    100,
			Thereafter: 100,
		},
		Encoding:         "json",
		EncoderConfig:    NewProductionEncoderConfig(),
		OutputPaths:      []string{"stderr"},
		ErrorOutputPaths: []string{"stderr"},
	}
}

func NewDevelopment(options ...Option) (*Logger, error) {
	return NewDevelopmentConfig().Build(options...)
}

func NewDevelopmentConfig() Config {
	return Config{
		Level:            NewAtomicLevelAt(DebugLevel),
		Development:      true,
		Encoding:         "console",
		EncoderConfig:    NewDevelopmentEncoderConfig(),
		OutputPaths:      []string{"stderr"},
		ErrorOutputPaths: []string{"stderr"},
	}
}
```

可以看到，两者在实现思路上是一样的，都是先创建一个配置对象 `zap.Config`，然后再调用配置对象的 `Build` 方法来构建 `Logger`。

`zap.Config` 定义如下：

```go
type Config struct {
	Level AtomicLevel `json:"level" yaml:"level"`
	Development bool `json:"development" yaml:"development"`
	DisableCaller bool `json:"disableCaller" yaml:"disableCaller"`
	DisableStacktrace bool `json:"disableStacktrace" yaml:"disableStacktrace"`
	Sampling *SamplingConfig `json:"sampling" yaml:"sampling"`
	Encoding string `json:"encoding" yaml:"encoding"`
	EncoderConfig zapcore.EncoderConfig `json:"encoderConfig" yaml:"encoderConfig"`
	OutputPaths []string `json:"outputPaths" yaml:"outputPaths"`
	ErrorOutputPaths []string `json:"errorOutputPaths" yaml:"errorOutputPaths"`
	InitialFields map[string]interface{} `json:"initialFields" yaml:"initialFields"`
}
```

每个配置项说明如下：

- `Level`: 日志级别。

- `Development`: 是否为开发模式。

- `DisableCaller`: 禁用调用信息，值为 `true` 时，日志中将不再显示记录日志时所在的函数调用文件名和行号。

- `DisableStacktrace`: 禁用堆栈跟踪捕获。

- `Sampling`: 采样策略配置，单位为每秒，作用是限制日志在每秒内的输出数量，以此来防止全局的 CPU 和 I/O 负载过高。

- `Encoding`: 指定日志编码器，目前支持 `json` 和 `console`。

- `EncoderConfig`: 编码配置，决定了日志字段格式。

- `OutputPaths`: 配置日志输出位置，URLs 或文件路径，可配置多个。

- `ErrorOutputPaths`: zap 包内部出现错误的日志输出位置，URLs 或文件路径，可配置多个，默认 `os.Stderr`。

- `InitialFields`: 初始化字段配置，该配置的字段会以结构化的形式打印在每条日志输出中。

我们再来对比下 `NewProductionEncoderConfig()` 和 `NewDevelopmentEncoderConfig()` 这两个配置的不同：

```go
func NewProductionEncoderConfig() zapcore.EncoderConfig {
	return zapcore.EncoderConfig{
		TimeKey:        "ts",
		LevelKey:       "level",
		NameKey:        "logger",
		CallerKey:      "caller",
		FunctionKey:    zapcore.OmitKey,
		MessageKey:     "msg",
		StacktraceKey:  "stacktrace",
		LineEnding:     zapcore.DefaultLineEnding,
		EncodeLevel:    zapcore.LowercaseLevelEncoder,
		EncodeTime:     zapcore.EpochTimeEncoder,
		EncodeDuration: zapcore.SecondsDurationEncoder,
		EncodeCaller:   zapcore.ShortCallerEncoder,
	}
}

func NewDevelopmentEncoderConfig() zapcore.EncoderConfig {
	return zapcore.EncoderConfig{
		// Keys can be anything except the empty string.
		TimeKey:        "T",
		LevelKey:       "L",
		NameKey:        "N",
		CallerKey:      "C",
		FunctionKey:    zapcore.OmitKey,
		MessageKey:     "M",
		StacktraceKey:  "S",
		LineEnding:     zapcore.DefaultLineEnding,
		EncodeLevel:    zapcore.CapitalLevelEncoder,
		EncodeTime:     zapcore.ISO8601TimeEncoder,
		EncodeDuration: zapcore.StringDurationEncoder,
		EncodeCaller:   zapcore.ShortCallerEncoder,
	}
}
```

对比来看，两者有很多不同的配置，比如生产环境下 `EncodeTime` 值为 `zapcore.EpochTimeEncoder`，开发环境下 `EncodeTime` 值为 `zapcore.ISO8601TimeEncoder`。这就是生产环境日志输出的时间格式为 `Unix epoch` 而开发环境日志输出的时间格式为 `ISO8601` 的原因。

`zapcore.EncoderConfig` 其他几个常用的配置项说明如下：

- `MessageKey`: 日志信息的键名，默认 `msg`。

- `LevelKey`: 日志级别的键名，默认 `level`。

- `TimeKey`: 日志时间的键名。

- `EncodeLevel`: 日志级别的格式，默认为小写，如 `info`。

除了提供 `zap.NewProduction()` 和 `zap.NewDevelopment()` 两个构造函数外，zap 还提供了 `zap.NewExample()` 来创建一个 `Logger` 对象，这个方法主要用于测试，这里就不多介绍了。

#### 给语法加点糖

zap 虽然速度足够快，但是多数情况下，我们并不需要极致的性能，而是想让代码写起来更爽一些。zap 为我们提供了解决方案 —— `SugaredLogger`。

```go
package main

import (
	"time"

	"go.uber.org/zap"
)

func main() {
	logger, _ := zap.NewProduction()
	defer logger.Sync()

	url := "https://jianghushinian.cn/"
	sugar := logger.Sugar()
	sugar.Infow("production failed to fetch URL",
		"url", url,
		"attempt", 3,
		"backoff", time.Second,
	)
	sugar.Info("Info")
	sugar.Infof("Infof: %s", url)
	sugar.Infoln("Infoln")
}
```

通过 `logger.Sugar()` 方法可以将一个 `Logger` 对象转换成一个 `SugaredLogger` 对象。

`SugaredLogger` 提供了更人性化的接口，日志中追加 key-value 时不在需要 `zap.String("url", url)` 这种显式指明类型的写法，只需要保证 key 为 `string` 类型，value 则可以为任意类型，能够减少我们编写的代码量。

此外，为了满足不同需求，`SugaredLogger` 提供了四种方式输出日志：`sugar.Xxx`、`sugar.Xxxw`、`sugar.Xxxf`、`sugar.Xxxln`。

执行以上代码，控制台得到如下输出：

```log
{"level":"info","ts":1679217743.5967638,"caller":"zap/sugar.go:15","msg":"production failed to fetch URL","url":"https://jianghushinian.cn/","attempt":3,"backoff":1}
{"level":"info","ts":1679217743.5969589,"caller":"zap/sugar.go:20","msg":"Info"}
{"level":"info","ts":1679217743.5969741,"caller":"zap/sugar.go:21","msg":"Infof: https://jianghushinian.cn/"}
{"level":"info","ts":1679217743.5969841,"caller":"zap/sugar.go:22","msg":"Infoln"}
```

我们知道，这种方便的写法是有一定代价的，所以开发中是否需要使用 `SugaredLogger` 来记录日志，需要根据程序的特点来决定。`SugaredLogger` 与 `Logger` 的性能对比同样可以在官方 [GitHub 仓库](https://github.com/uber-go/zap#performance)中看到。

#### 定制 Logger

通过查看 `zap.NewProduction()` 和 `zap.NewDevelopment()` 两个构造函数源码，我们知道可以使用 `zap.Config` 对象的 `Build` 方法创建 `Logger` 对象。那么我们很容易能够想到，如果要定制 `Logger`，只需要创建一个定制的 `zap.Config` 即可。

```go
package main

import (
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

func newCustomLogger() (*zap.Logger, error) {
	cfg := zap.Config{
		Level:       zap.NewAtomicLevelAt(zap.DebugLevel),
		Development: false,
		Encoding:    "json",
		EncoderConfig: zapcore.EncoderConfig{
			TimeKey:        "time",
			LevelKey:       "level",
			NameKey:        "logger",
			CallerKey:      "", // 不记录日志调用位置
			FunctionKey:    zapcore.OmitKey,
			MessageKey:     "message",
			LineEnding:     zapcore.DefaultLineEnding,
			EncodeLevel:    zapcore.LowercaseLevelEncoder,
			EncodeTime:     zapcore.RFC3339TimeEncoder,
			EncodeDuration: zapcore.SecondsDurationEncoder,
			EncodeCaller:   zapcore.ShortCallerEncoder,
		},
		OutputPaths:      []string{"stdout", "test.log"},
		ErrorOutputPaths: []string{"error.log"},
	}
	return cfg.Build()
}

func main() {
	logger, _ := newCustomLogger()
	defer logger.Sync()

	// 增加一个 skip 选项，触发 zap 内部 error，将错误输出到 error.log
	logger = logger.WithOptions(zap.AddCallerSkip(100))

	logger.Info("Info msg")
	logger.Error("Error msg")
}
```

以上代码通过 `newCustomLogger` 函数创建了一个自定义的 `Logger`，同样通过先定义一个 `zap.Config` 然后再调用其 `Build` 方法来实现。

配置日志分别输出到标准输出和 `test.log` 文件，执行以上代码，控制台和 `test.log` 都会得到如下输出：

```log
{"level":"info","time":"2023-03-19T19:19:18+08:00","message":"Info msg"}
{"level":"error","time":"2023-03-19T19:19:18+08:00","message":"Error msg"}
```

另外，我们还通过 `logger.WithOptions()` 为 `Logger` 对象增加了一个选项 `zap.AddCallerSkip(100)`，这个选项的作用是指定在通过调用栈获得行号时跳过的调用深度，因为我们的函数调用栈并不是 100 层，所以会触发 zap 内部错误，zap 会将错误日志输出到 `ErrorOutputPaths` 配置指定的位置中，即 `error.log`。

`error.log` 得到的错误日志如下：

```log
2023-03-19 11:19:18.438824 +0000 UTC Logger.check error: failed to get caller
2023-03-19 11:19:18.44921 +0000 UTC Logger.check error: failed to get caller
```

`logger.WithOptions()` 支持的选项如下：

- `WrapCore(f func(zapcore.Core) zapcore.Core)`: 使用一个新的 `zapcore.Core` 替换掉 `Logger` 内部原有的的 `zapcore.Core` 属性。

- `Hooks(hooks ...func(zapcore.Entry) error)`: 注册钩子函数，用来在日志打印时同时调用注册的钩子函数。

- `Fields(fs ...Field)`: 添加公共字段。

- `ErrorOutput(w zapcore.WriteSyncer)`: 指定日志组件内部出现异常时的输出位置。

- `Development()`: 将日志记录器设为开发模式，这将使 `DPanic` 级别日志记录错误后执行 `panic()`。

- `AddCaller()`: 与 `WithCaller(true)` 等价。

- `WithCaller(enabled bool)`: 指定是否在日志输出内容中增加调用信息，即文件名和行号。

- `AddCallerSkip(skip int)`: 指定在通过调用栈获取文件名和行号时跳过的调用深度。

- `AddStacktrace(lvl zapcore.LevelEnabler)`: 用来指定某个日志级别及以上级别输出调用堆栈。

- `IncreaseLevel(lvl zapcore.LevelEnabler)`: 提高日志级别，如果传入的 `lvl` 比现有级别低，则不会改变日志级别。

- `WithFatalHook(hook zapcore.CheckWriteHook)`: 当出现 `Fatal` 级别日志时调用的钩子函数。

- `WithClock(clock zapcore.Clock)`: 指定日志记录器用来确定当前时间的 `zapcore.Clock` 对象，默认为 `time.Now` 的系统时钟。

创建自定义的配置对象，除了在代码中指定配置参数，也可以将这些配置项写入到 JSON 文件中，然后通过 `json.Unmarshal` 的方式将配置绑定到 `zap.Config`，可以参考[官方示例](https://pkg.go.dev/go.uber.org/zap#example-package-BasicConfiguration)。

### 二次开发

#### 封装自己的 zap 包

通过前文对 zap 的介绍，相信你对 zap 的特点和用法都已了然于心。如果你用惯了 Go log 标准库，或者是 Logrus 第三方库，那么对于 zap 所提供的 API 一定不会感到满意。因此，我基于 zap 包定制开发了自己的日志包，项目地址在[这里](https://github.com/jianghushinian/gokit/tree/main/log)，你可以点进去查看源码。

基于 zap 定制的日志包并没有太多的逻辑，只在 zap/zapcore （zapcore 故名思义，是 zap 的核心，zap 是对 zapcore 的封装）基础上进行了很薄的一层封装，所以性能上无需担心。

> 提示：关于定制开发的日志包设计思路这里就不讲解了，之后我会单独写一篇文章来进行解读，敬请期待（先挖个坑）。

日志包对外提供了类似 Go log 标准库风格的 API，几种常见使用方式如下：

#### 像 Go log 标准库一样开箱即用

```go
package main

import (
	"os"
	"time"

	log "github.com/jianghushinian/gokit/log/zap"
)

func main() {
	defer log.Sync()
	log.Info("failed to fetch URL", log.String("url", "https://jianghushinian.cn/"))
	log.Warn("Warn msg", log.Int("attempt", 3))
	log.Error("Error msg", log.Duration("backoff", time.Second))

	log.SetLevel(log.ErrorLevel)
	log.Info("Info msg")
	log.Warn("Warn msg")
	log.Error("Error msg")

	// 替换默认 Logger
	file, _ := os.OpenFile("custom.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
	logger := log.New(file, log.InfoLevel)
	log.ReplaceDefault(logger)
	log.Info("Info msg in replace default logger after")
}
```

自定义日志包提供了默认的 `std` `Logger` 对象和 `Info`、`Warn` 等包级别的开放函数，以此实现开箱即用的效果。日志包还提供了 `SetLevel` 函数来支持运行时修改日志级别。

另外，如果你对默认的 `Logger` 不满意，可以使用 `log.New` 来创建新的 `Logger`，接下来只需要通过 `log.ReplaceDefault(logger)` 一行代码，就可以将默认的 `Logger` 替换成新创建的 `Logger` 对象。之后通过 `log.Info` 来输出日志底层使用的已经是替换后的 `Logger` 对象了。

执行以上代码，控制台输出:

```log
{"level":"info","ts":"2023-03-19T21:57:59+08:00","msg":"failed to fetch URL","url":"https://jianghushinian.cn/"}
{"level":"warn","ts":"2023-03-19T21:57:59+08:00","msg":"Warn msg","attempt":3}
{"level":"error","ts":"2023-03-19T21:57:59+08:00","msg":"Error msg","backoff":1}
{"level":"error","ts":"2023-03-19T21:57:59+08:00","msg":"Error msg"}
```

`custom.log` 输出:

```log
{"level":"info","ts":"2023-03-19T21:57:59+08:00","msg":"Info msg in replace default logger after"}
```

#### 将日志输出到多个位置

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

自定义日志包通过 `log.NewTee` 构造函数提供了将日志输出到多个位置的能力，`log.NewTee` 接收一个 `log.TeeOption` 列表，用来定义不同级别的日志输出方式，其本质上是定义了多个 `zap.Core` 对象。

其中 `Out` 字段为日志输出目标地址，`LevelEnablerFunc` 用来指定日志级别。这有别于使用 `log.InfoLevel` 这种常量日志级别的设定，使用 `LevelEnablerFunc` 只需要定义一个接收 `log.Level` 作为参数并返回 `bool` 类型值的函数即可。

这样如果函数内部写为 `return level == log.InfoLevel` 表示 `Info` 级别使用此配置日志对象输出，日过函数内部实现为 `return level >= log.WarnLevel` 则表示 `Warn` 及以上级别即 `Warn`、`Error`、`DPanic`、`Panic`、`Fatal` 级别都使用此配置对应的日志对象作为输出，更为灵活。

执行以上代码，控制台输出:

```log
{"level":"info","ts":"2023-03-19T22:06:25+08:00","msg":"Info tee msg"}
```

`test-warn.log` 输出:

```log
{"level":"warn","ts":"2023-03-19T22:06:25+08:00","msg":"Warn tee msg"}
```

选项模式、Hooks、日志轮转等更多使用场景可以参考项目的[使用示例](https://github.com/jianghushinian/gokit/tree/main/log/zap#%E4%BD%BF%E7%94%A8%E7%A4%BA%E4%BE%8B)。在 Gin 框架中的使用示例，可以参考[这里](https://github.com/jianghushinian/gokit/tree/main/log/zap/examples/gin)。

### 总结

本文讲解了 Go 第三方日志库 zap 的特点及使用。

zap 作为以快著称的日志库，在使用上不如 Logrus 来的方便，于是我基于 zap 定制开发了自己的日志包，并简单介绍了如何使用。

关于 zap 的更多使用方式可以参考[官方文档](https://pkg.go.dev/go.uber.org/zap)，如果你对我封装日志包感兴趣，可以查看[源码](https://github.com/jianghushinian/gokit/tree/main/log)。

**参考**

> zap 源码: https://github.com/uber-go/zap
> zap 文档: https://pkg.go.dev/go.uber.org/zap
> 基于 zap 开发的日志包: https://github.com/jianghushinian/gokit/tree/main/log

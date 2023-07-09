---
title: 如何用 Go 实现一个配置包
date: 2023-03-31 12:52:34
tags:
- Go
- Web
categories:
- Go
---

在现代软件开发中，配置文件是不可或缺的一部分。在编写 Go 项目时，不管是一个简单的单文件脚本还是一个庞大的微服务项目，程序的灵活性和可扩展性都需要依赖于配置文件的加载。配置文件可以包含程序的各种设置，如端口号、数据库连接信息、认证密钥等，这些设置可能需要根据不同的环境和部署情况进行调整。因此，一个可靠的配置文件加载机制对于项目的成功实现至关重要。本文就来探究下在 Go 项目中如何更加方便的加载和管理配置。

<!-- more -->

在 Go 中，有很多方法可以加载和管理配置，例如使用命令行标志、环境变量或者读取特定格式的文件。选择哪种方法取决于项目的具体需求和开发人员的偏好。

### 需求

根据我的开发经验，一个配置包通常需要支持如下功能：

- 支持从环境变量、命令行标志加载配置文件，然后将配置文件内容映射到 Go 结构体中，这个过程叫作反序列化。

- 支持将 Go 结构体中的配置信息写入配置文件，这个过程叫作序列化。

- 起码要支持 YAML、JSON 这两种最常见的配置文件格式。

### config 包实现

#### 反序列化

反序列化操作是配置包最常用的功能，通过反序列化我们可以将配置文件内容映射到 Go 结构体中。所以我们先来实现如何进行配置文件的反序列化操作。

```go
type FileType int

const (
	FileTypeYAML = iota
	FileTypeJSON
)

func LoadConfig(filename string, cfg interface{}, typ FileType) error {
	data, err := os.ReadFile(filename)
	if err != nil {
		return fmt.Errorf("ReadFile: %v", err)
	}
	switch typ {
	case FileTypeYAML:
		return yaml.Unmarshal(data, cfg)
	case FileTypeJSON:
		return json.Unmarshal(data, cfg)
	}
	return errors.New("unsupported file type")
}
```

首先定义一个 `FileType` 类型，用来区分不同的配置文件格式，我们要实现的日志包能够支持 YAML 和 JSON 两种格式。

接下来实现了 `LoadConfig` 函数，用来加载配置，它接收三个参数：配置文件名称、配置结构体、文件类型。

在函数内部，首先使用 `os.ReadFile(filename)` 读取配置文件内容，然后根据文件类型，使用不同的方法来反序列化，最终将配置信息映射到 `cfg` 结构体中。

#### 序列化

配置包还要支持序列化操作，也就是将配置信息从内存中写入文件。有时候我们想保存一个 `example-config.yaml` 供项目使用者参考，这个功能就很实用。

```go
func DumpConfig(filename string, cfg interface{}, typ FileType) error {
	var (
		data []byte
		err  error
	)
	switch typ {
	case FileTypeYAML:
		data, err = yaml.Marshal(cfg)
	case FileTypeJSON:
		data, err = json.Marshal(cfg)
	}
	if err != nil {
		return err
	}
	f, err := os.Create(filename)
	if err != nil {
		return err
	}
	_, err = f.Write(data)
	_ = f.Close()
	return err
}
```

我们实现了 `DumpConfig` 函数用来进行序列化操作，该函数同样接收三个参数：配置文件名、配置结构体、文件类型。

在函数中，首先根据文件类型对配置对象 `cfg` 进行序列化操作得到 `data`，然后通过 `os.Create(filename)` 创建配置文件文件，最后使用 `f.Write(data)` 将序列化后的内容写入文件。

现在配置包的核心功能已经实现了，并且两个函数都被允许导出，如果其他项目想使用，其实是可以拿过去直接使用的。不过目前的配置包在易用性上还有可优化的空间。

#### 通过环境变量/命令行参数指定配置文件

以上序列化/反序列化函数的实现，配置文件名都是需要通过参数传递进来的，我们还可以通过环境变量或者命令行参数的形式来指定配置文件名。

```go
var (
	cfgPath = flag.String("c", "config.yaml", "path to config file")
	dump    = flag.Bool("d", false, "dump config to file")
)

func init() {
	_ = flag.Set("c", os.Getenv("CONFIG_PATH"))
	_ = flag.Set("d", os.Getenv("DUMP_CONFIG"))
}
```

命令行参数解析可以使用 Go 内置的 flag 包来实现，我们在这里声明了两个标志，`-c` 用来传递配置文件名，`-d` 用来决定要进行序列化还是反序列化操作。

此外，在 `init` 函数中，我们实现了从环境变量中加载配置文件名和需要执行的操作。当配置包被导入时，会优先执行 `init` 函数，所以这些信息会优先从环境变量读取。

#### 封装

现在我们可以通过环境变量或命令行参数的形式拿到配置文件名和需要执行的操作两个参数，并且分别赋值给 `cfgPath`、`dump` 两个变量，接下来我们将对之前实现的序列化/反序列化函数进行封装，让其使用起来更加便捷。

##### 反序列化

首先，我们可以将反序列化操作函数 `LoadConfig` 进一步封装，将其包装成两个函数分别加载 YAML 配置和 JSON 配置，这样调用方在使用时能够少传递一个参数。实现如下：

```go
func LoadYAMLConfig(filename string, cfg interface{}) error {
	return LoadConfig(filename, cfg, FileTypeYAML)
}

func LoadJSONConfig(filename string, cfg interface{}) error {
	return LoadConfig(filename, cfg, FileTypeJSON)
}
```

其次，我们拿到的 `cfgPath` 参数也能派上用场了，可以对 `LoadYAMLConfig`、`LoadJSONConfig` 进一步封装：

```go
func LoadYAMLConfigFromFlag(cfg interface{}) error {
	if !flag.Parsed() {
		flag.Parse()
	}
	return LoadYAMLConfig(*cfgPath, cfg)
}

func LoadJSONConfigFromFlag(cfg interface{}) error {
	if !flag.Parsed() {
		flag.Parse()
	}
	return LoadJSONConfig(*cfgPath, cfg)
}
```

因为 `cfgPath` 是通过环境变量或命令行参数形式传入，所以调用反序列化函数的时候也就没必要手动传入了。

不过在使用 `cfgPath` 变量之前，我们调用了 `flag.Parsed()` 并对其返回值进行了判断，如果 `flag.Parsed()` 返回 `false` 则调用 `flag.Parse()`。

这一步操作是因为，配置包的调用方可能也使用了 flag 包，那么如果调用方已经调用过 `flag.Parse()` 解析了命令行参数，调用 `flag.Parsed()` 就会返回 `true`，此时配置包内部没必要再解析一遍命令行参数，可以直接使用。

> 提示：如果你不熟悉 flag 包的用法，可以参考我之前写的文章[《深入探究 Go flag 标准库》](https://jianghushinian.cn/2023/03/26/dive-into-the-go-flag-standard-library/)。

##### 序列化

封装好了反序列化操作，对应的，也要封装序列化操作。代码实现上与反序列化操作如出一辙，我将代码贴在这里，就不再进行讲解了。

```go
func DumpYAMLConfig(filename string, cfg interface{}) error {
	return DumpConfig(filename, cfg, FileTypeYAML)
}

func DumpJSONConfig(filename string, cfg interface{}) error {
	return DumpConfig(filename, cfg, FileTypeJSON)
}

func DumpYAMLConfigFromFlag(cfg interface{}) error {
	if !flag.Parsed() {
		flag.Parse()
	}
	return DumpYAMLConfig(*cfgPath, cfg)
}

func DumpJSONConfigFromFlag(cfg interface{}) error {
	if !flag.Parsed() {
		flag.Parse()
	}
	return DumpJSONConfig(*cfgPath, cfg)
}
```

#### 统一出口函数

我们通过环境变量或命令行参数拿到的另外一个用户指定的变量 `dump` 还没有使用，`dump` 是一个 `bool` 值，代表用户是否想要序列化配置，如果为 `true` 则进行序列化操作，否则进行反序列化操作。

由此，我们可以将序列化/反序列化操作封装成一个函数，内部通过 `dump` 的值来决定执行哪种操作。实现如下：

```go
func LoadOrDumpYAMLConfigFromFlag(cfg interface{}) error {
	if !flag.Parsed() {
		flag.Parse()
	}
	if *dump {
		if err := DumpYAMLConfig(*cfgPath, cfg); err != nil {
			return err
		}
		os.Exit(0)
	}
	return LoadYAMLConfig(*cfgPath, cfg)
}

func LoadOrDumpJSONConfigFromFlag(cfg interface{}) error {
	if *dump {
		if err := DumpJSONConfigFromFlag(cfg); err != nil {
			return err
		}
		os.Exit(0)
	}
	return LoadJSONConfigFromFlag(cfg)
}
```

我们对 YAML 和 JSON 格式的配置分别实现了 `LoadOrDumpYAMLConfigFromFlag` 和 `LoadOrDumpJSONConfigFromFlag` 函数。顾名思义，这个函数既能加载（`Load`）配置，也能将配置写入（`Dump`）文件。

两个函数内部思路相同，如果 `dump` 为 `true`，就序列化配置信息到文件中，并且使用 `os.Exit(0)` 退出程序。此时，控制台不会有任何输出，这遵循了 Linux/Unix 系统的设计哲学，没有消息就是最好的消息；如果 `dump` 为 `false`，则进行反序列化操作。

以上我们就实现了自定义配置包的全部功能，完整代码可以[点击这里](https://github.com/jianghushinian/gokit/tree/main/config/config)查看。

### config 包使用

配置包的完整使用示例我写在了项目 GitHub 仓库的 [README.md](https://github.com/jianghushinian/gokit/tree/main/config/config) 文件中，你可以点进去查看。

### 总结

本文带大家一起使用 Go 语言实现了一个简洁的日志包，虽然功能不多，但足够中小型项目使用。如果对加载配置文件有更要的要求，如更多格式的日志文件支持、热加载等，则可以考虑使用更强大的 Go 第三方库 Viper，或者使用 Nacos、Disconf 等配置中心。

在这个云原生的时代，其实多数情况下，配置包不需要实现多么复杂的功能，尽量保持简洁。配置文件可以使用 K8s `ConfigMap` 来实现，配置包的高级特性也可以考虑通过 `Sidecar` 的形式来实现。

**参考**

> config 源码: https://github.com/jianghushinian/gokit/tree/main/config/config

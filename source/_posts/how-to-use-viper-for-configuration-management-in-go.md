---
title: 在 Go 中如何使用 Viper 来管理配置
date: 2023-04-25 20:53:03
tags:
- Go
- Config
categories:
- Go
---

Viper 是一个功能齐全的 Go 应用程序配置库，支持很多场景。它可以处理各种类型的配置需求和格式，包括设置默认值、从多种配置文件和环境变量中读取配置信息、实时监视配置文件等。无论是小型应用还是大型分布式系统，Viper 都可以提供灵活而可靠的配置管理解决方案。在本文中，我们将深入探讨 Viper 的各种用法和使用场景，以帮助读者更好地了解和使用 Viper 来管理应用程序配置。

<!-- more -->

### 为什么选择 Viper

当我们在做技术选型时，肯定要知道为什么选择某一项技术，而之所以选择使用 Viper 来管理应用程序的配置，Viper 官方给出了如下答案：

当构建应用程序时，你不想担心配置文件格式，只想专注于构建出色的软件。Viper 就是为了帮助我们解决这个问题而存在的。

Viper 可以完成以下工作：

1. 查找、加载和反序列化 JSON、TOML、YAML、HCL、INI、envfile 或 Java Properties 格式的配置文件。

2. 为不同的配置选项设置默认值。

3. 为通过命令行标志指定的选项设置覆盖值。

4. 提供别名系统，以便轻松重命名配置项而不破坏现有代码。

5. 可以轻松区分用户提供的命令行参数或配置文件中的值是否与默认值相同。

> 注：关于上面第 5 点，我个人理解的使用场景是：
> 1. 先从命令行参数或配置文件中读取配置。
> 2. 可以使用 `viper.IsSet(key)` 方法判断用户是否设置了 `key` 所对应的 `value`，如果设置了，可以通过 `viper.Get(key)` 获取值。
> 3. 调用 `viper.SetDefault(key, default_value)` 来设置默认值（默认值不会覆盖上一步所获取到的值）。
> 在第 2 步中可以拿到用户设置的值 `value`，在第 3 步中可以知道默认值 `default_value`，这样其实就可以判断两者是否相同了。

Viper 采用以下优先级顺序来加载配置，按照优先级由高到低排序如下：

- 显式调用 `viper.Set` 设置的配置值

- 命令行参数

- 环境变量

- 配置文件

- key/value 存储

- 默认值

> 注意 ⚠️：Viper 配置中的键不区分大小写，如 `user/User/USER` 被视为是相等的 `key`，关于是否将其设为可选，目前还在讨论中。

Viper 包中最核心的两个功能是：如何把配置值读入 Viper 和从 Viper 中读取配置值，接下来我将分别介绍这两个功能。

### 把配置值读入 Viper

Viper 支持多种方式读入配置：

- 设置默认配置值

- 从配置文件读取配置

- 监控并重新读取配置文件

- 从 `io.Reader` 读取配置

- 从环境变量读取配置

- 从命令行参数读取配置

- 从远程 key/value 存储读取配置

我们一个一个来看。

#### 设置默认配置值

一个好的配置系统应该支持默认值。Viper 支持使用 `viper.SetDefault(key, value)` 为 `key` 设置默认值 `value`，在没有通过配置文件、环境变量、远程配置或命令行标志设置 `key` 所对应值的情况下，这很有用。

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
)

func main() {
	// 设置默认配置
	viper.SetDefault("username", "jianghushinian")
	viper.SetDefault("server", map[string]string{"ip": "127.0.0.1", "port": "8080"})

	// 读取配置值
	fmt.Printf("username: %s\n", viper.Get("Username")) // key 不区分大小写
	fmt.Printf("server: %+v\n", viper.Get("server"))
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go
username: jianghushinian
server: map[ip:127.0.0.1 port:8080]
```

#### 从配置文件读取配置

Viper 支持从 JSON、TOML、YAML、HCL、INI、envfile 或 Java Properties 格式的配置文件中读取配置。Viper 可以搜索多个路径，但目前单个 Viper 实例只支持单个配置文件。**Viper 不会默认配置任何搜索路径，将默认决定留给应用程序**。

主要有两种方式来加载配置文件：

- 通过 `viper.SetConfigFile()` 指定配置文件，如果配置文件名中没有扩展名，则需要使用 `viper.SetConfigType()` 显式指定配置文件的格式。

- 通过 `viper.AddConfigPath()` 指定配置文件的搜索路径中，可以通过多次调用，来设置多个配置文件搜索路径。然后通过 `viper.SetConfigName()` 指定不带扩展名的配置文件，Viper 会根据所添加的路径顺序查找配置文件，如果找到就停止查找。

```go
package main

import (
	"errors"
	"flag"
	"fmt"

	"github.com/spf13/viper"
)

var (
	cfg = flag.String("c", "", "config file.")
)

func main() {
	flag.Parse()

	if *cfg != "" {
		viper.SetConfigFile(*cfg)   // 指定配置文件（路径 + 配置文件名）
		viper.SetConfigType("yaml") // 如果配置文件名中没有扩展名，则需要显式指定配置文件的格式
	} else {
		viper.AddConfigPath(".")             // 把当前目录加入到配置文件的搜索路径中
		viper.AddConfigPath("$HOME/.config") // 可以多次调用 AddConfigPath 来设置多个配置文件搜索路径
		viper.SetConfigName("cfg")           // 指定配置文件名（没有扩展名）
	}

	// 读取配置文件
	if err := viper.ReadInConfig(); err != nil {
		if _, ok := err.(viper.ConfigFileNotFoundError); ok {
			fmt.Println(errors.New("config file not found"))
		} else {
			fmt.Println(errors.New("config file was found but another error was produced"))
		}
		return
	}

	fmt.Printf("using config file: %s\n", viper.ConfigFileUsed())

	// 读取配置值
	fmt.Printf("username: %s\n", viper.Get("username"))
}
```

假如有如下配置文件 `config.yaml` 与示例程序在同一目录中：

```yaml
username: jianghushinian
password: 123456
server:
  ip: 127.0.0.1
  port: 8080
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go -c ./config.yaml 
using config file: ./config.yaml
username: jianghushinian
```

#### 监控并重新读取配置文件

Viper 支持在应用程序运行过程中实时读取配置文件，即热加载配置。

只需要调用 `viper.WatchConfig()` 即可开启此功能。

```go
package main

import (
	"fmt"
	"time"

	"github.com/fsnotify/fsnotify"
	"github.com/spf13/viper"
)

func main() {
	viper.SetConfigFile("./config.yaml")
	viper.ReadInConfig()

	// 注册每次配置文件发生变更后都会调用的回调函数
	viper.OnConfigChange(func(e fsnotify.Event) {
		fmt.Printf("config file changed: %s\n", e.Name)
	})

	// 监控并重新读取配置文件，需要确保在调用前添加了所有的配置路径
	viper.WatchConfig()

	// 阻塞程序，这个过程中可以手动去修改配置文件内容，观察程序输出变化
	time.Sleep(time.Second * 10)

	// 读取配置值
	fmt.Printf("username: %s\n", viper.Get("username"))
}
```

值得注意的是，在调用 `viper.WatchConfig()` 监控并重新读取配置文件之前，需要确保添加了所有的配置路径。

并且，我们还可以通过 `viper.OnConfigChange()` 函数注册一个每次配置文件发生变更后都会调用的回调函数。

我们依然使用上面的 `config.yaml` 配置文件：

```yaml
username: jianghushinian
password: 123456
server:
  ip: 127.0.0.1
  port: 8080
```

执行以上示例代码，并在程序阻塞的时候，手动修改配置文件中 `username` 所对应的值为 `江湖十年`，可以得到如下输出：

```bash
$ go run main.go
config file changed: config.yaml
username: 江湖十年
```

#### 从 `io.Reader` 读取配置

Viper 支持从任何实现了 `io.Reader` 接口的配置源中读取配置。

```go
package main

import (
	"bytes"
	"fmt"

	"github.com/spf13/viper"
)

func main() {
	viper.SetConfigType("yaml") // 或者使用 viper.SetConfigType("YAML")

	var yamlExample = []byte(`
username: jianghushinian
password: 123456
server:
  ip: 127.0.0.1
  port: 8080
`)

	viper.ReadConfig(bytes.NewBuffer(yamlExample))

	// 读取配置值
	fmt.Printf("username: %s\n", viper.Get("username"))
}
```

这里我们通过 `bytes.NewBuffer()` 构造了一个 `bytes.Buffer` 对象，它实现了 `io.Reader` 接口，所以可以直接传递给 `viper.ReadConfig()` 来从中读取配置。

执行以上示例代码得到如下输出：

```bash
$ go run main.go
username: jianghushinian
```

#### 从环境变量读取配置

Viper 还支持从环境变量读取配置，有 5 个方法可以帮助我们使用环境变量:

- `AutomaticEnv()`：可以绑定全部环境变量（用法上类似 flag 包的 `flag.Parse()`）。调用后，Viper 会自动检测和加载所有环境变量。

- `BindEnv(string...) : error`：绑定一个环境变量。需要一个或两个参数，第一个参数是配置项的键名，第二个参数是环境变量的名称。如果未提供第二个参数，则 Viper 将假定环境变量名为：`环境变量前缀_键名`，且为全大写形式。例如环境变量前缀为 `ENV`，键名为 `username`，则环境变量名为 `ENV_USERNAME`。当显式提供第二个参数时，它不会自动添加前缀，也不会自动将其转换为大写。例如，使用 `viper.BindEnv("username", "username")` 绑定键名为 `username` 的环境变量，应该使用 `viper.Get("username")` 读取环境变量的值。

  在使用环境变量时，需要注意，每次访问它的值时都会去环境变量中读取。当调用 `BindEnv` 时，Viper 不会固定它的值。

- `SetEnvPrefix(string)`：可以告诉 Viper 在读取环境变量时使用的前缀。`BindEnv` 和 `AutomaticEnv` 都将使用此前缀。例如，使用 `viper.SetEnvPrefix("ENV")` 设置了前缀为 `ENV`，并且使用 `viper.BindEnv("username")` 绑定了环境变量，在使用 `viper.Get("username")` 读取环境变量时，实际读取的 `key` 是 `ENV_USERNAME`。

- `SetEnvKeyReplacer(string...) *strings.Replacer`：允许使用 `strings.Replacer` 对象在一定程度上重写环境变量的键名。例如，存在 `SERVER_IP="127.0.0.1"` 环境变量，使用 `viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_", "-", "_"))` 将键名中的 `.` 或 `-` 替换成 `_`，则通过 `viper.Get("server_ip")`、`viper.Get("server.ip")`、`viper.Get("server-ip")` 三种方式都可以读取环境变量对应的值。

- `AllowEmptyEnv(bool)`：当环境变量为空时（有键名而没有值的情况），默认会被认为是未设置的，并且程序将回退到下一个配置来源。要将空环境变量视为已设置，可以使用此方法。

> 注意 ⚠️：Viper 在读取环境变量时，是区分大小写的。

使用示例：

```go
package main

import (
	"fmt"
	"strings"

	"github.com/spf13/viper"
)

func main() {
	viper.SetEnvPrefix("env") // 设置读取环境变量前缀，会自动转为大写 ENV
	viper.AllowEmptyEnv(true) // 将空环境变量视为已设置

	viper.AutomaticEnv()      // 可以绑定全部环境变量
	viper.BindEnv("username") // 也可以单独绑定某一个环境变量
	viper.BindEnv("password")

	// 将键名中的 . 或 - 替换成 _
	viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_", "-", "_"))

	// 读取配置
	fmt.Printf("username: %v\n", viper.Get("username"))
	fmt.Printf("password: %v\n", viper.Get("password"))
	fmt.Printf("server.ip: %v\n", viper.Get("server.ip"))

	// 读取全部配置，只能获取到通过 BindEnv 绑定的环境变量，无法获取到通过 AutomaticEnv 绑定的环境变量
	fmt.Println(viper.AllSettings())
}
```

执行以上示例代码得到如下输出：

```bash
$ ENV_USERNAME=jianghushinian ENV_SERVER_IP=127.0.0.1 ENV_PASSWORD= go run main.go
username: jianghushinian
password: 
server.ip: 127.0.0.1
map[password: username:jianghushinian]
```

#### 从命令行参数读取配置

Viper 支持 [pflag](https://github.com/spf13/pflag) 包（它们其实都在 [spf13](https://github.com/spf13) 仓库下），能够绑定命令行标志，从而读取命令行参数。

同 `BindEnv` 类似，在调用绑定方法时，不会设置值，而是在每次访问时设置。这意味着我们可以随时绑定它，例如可以在 `init()` 函数中。

- `BindPFlag`：对于单个标志，可以调用此方法进行绑定。

- `BindPFlags`：可以绑定一组现有的标志集 `pflag.FlagSet`。

示例程序如下：

```go
package main

import (
	"fmt"

	"github.com/spf13/pflag"
	"github.com/spf13/viper"
)

var (
	username = pflag.StringP("username", "u", "", "help message for username")
	password = pflag.StringP("password", "p", "", "help message for password")
)

func main() {
	pflag.Parse()

	viper.BindPFlag("username", pflag.Lookup("username")) // 绑定单个标志
	viper.BindPFlags(pflag.CommandLine)                   // 绑定标志集

	// 读取配置值
	fmt.Printf("username: %s\n", viper.Get("username"))
	fmt.Printf("password: %s\n", viper.Get("password"))
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go -u jianghushinian -p 123456
username: jianghushinian
password: 123456
```

因为 pflag 能够兼容标准库的 flag 包，所以我们也可以变相的让 Viper 支持 flag。

```go
package main

import (
	"flag"
	"fmt"

	"github.com/spf13/pflag"
	"github.com/spf13/viper"
)

func main() {
	flag.String("username", "", "help message for username")

	pflag.CommandLine.AddGoFlagSet(flag.CommandLine) // 将 flag 命令行参数注册到 pflag
	pflag.Parse()

	viper.BindPFlags(pflag.CommandLine)

	// 读取配置值
	fmt.Printf("username: %s\n", viper.Get("username"))
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go --username jianghushinian
username: jianghushinian
```

如果你不使用 flag 或 pflag，则 Viper 还提供了 Go 接口的形式来支持其他 Flags，具体用法可以参考[官方文档](https://github.com/spf13/viper#flag-interfaces)。

#### 从远程 key/value 存储读取配置

要在 Viper 中启用远程支持，需要匿名导入 `viper/remote` 包：

```go
import _ "github.com/spf13/viper/remote"
```

Viper 支持 etcd、Consul 等远程 key/value 存储，这里以 Consul 为例进行讲解。

首先需要准备 Consul 环境，最方便快捷的方式就是启动一个 Docker 容器：

```bash
$ docker run \
    -d \
    -p 8500:8500 \
    -p 8600:8600/udp \
    --name=badger \
    consul agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0
```

Docker 容器启动好后，浏览器访问 `http://localhost:8500/`，即可进入 Consul 控制台，在 `user/config` 路径下编写 YAML 格式的配置。

![Consul Config](consul.png)

使用 Viper 从 Consul 读取配置示例代码如下：

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
	_ "github.com/spf13/viper/remote" // 必须导入，才能加载远程 key/value 配置
)

func main() {
	viper.AddRemoteProvider("consul", "localhost:8500", "user/config") // 连接远程 consul 服务
	viper.SetConfigType("YAML")                                        // 显式设置文件格式文 YAML
	viper.ReadRemoteConfig()

	// 读取配置值
	fmt.Printf("username: %s\n", viper.Get("username"))
	fmt.Printf("server.ip: %s\n", viper.Get("server.ip"))
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go
username: jianghushinian
server.ip: 127.0.0.1
```

> 笔记：如果你想停止通过 Docker 安装的 Consul 容器，则可以执行 `docker stop badger` 命令。如果需要删除，则可以执行 `docker rm badger` 命令。

### 从 Viper 中读取配置值

前文中我们介绍了各种将配置读入 Viper 的技巧，现在该学习如何使用这些配置了。

在 Viper 中，有如下几种方法可以获取配置值：

- `Get(key string) interface{}`：获取配置项 `key` 所对应的值，`key` 不区分大小写，返回接口类型。

- `Get<Type>(key string) <Type>`：获取指定类型的配置值，<Type> 可以是 Viper 支持的类型：`GetBool`、`GetFloat64`、`GetInt`、`GetIntSlice`、`GetString`、`GetStringMap`、`GetStringMapString`、`GetStringSlice`、`GetTime`、`GetDuration`。

- `AllSettings() map[string]interface{}`：返回所有配置。根据我的经验，如果使用环境变量指定配置，则只能获取到通过 `BindEnv` 绑定的环境变量，无法获取到通过 `AutomaticEnv` 绑定的环境变量。

- `IsSet(key string) bool`：值得注意的是，在使用 `Get` 或 `Get<Type>` 获取配置值，如果找不到，则每个 `Get` 函数都会返回一个零值。为了检查给定的键是否存在，可以使用 `IsSet` 方法，存在返回 `true`，不存在返回 `false`。

#### 访问嵌套的键

有如下配置文件 `config.yaml`：

```yaml
username: jianghushinian
password: 123456
server:
  ip: 127.0.0.1
  port: 8080
```

可以通过 `.` 分隔符来访问嵌套字段。

```go
viper.Get("server.ip")
```

示例如下：

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
)

func main() {
	viper.SetConfigFile("./config.yaml")
	viper.ReadInConfig()

	// 读取配置值
	fmt.Printf("username: %v\n", viper.Get("username"))
	fmt.Printf("server: %v\n", viper.Get("server"))
	fmt.Printf("server.ip: %v\n", viper.Get("server.ip"))
	fmt.Printf("server.port: %v\n", viper.Get("server.port"))
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go
username: jianghushinian
server: map[ip:127.0.0.1 port:8080]
server.ip: 10.0.0.1
server.port: 8080
```

有一种情况是，配置中本就存在着叫 `server.ip` 的键，那么它会遮蔽 `server` 对象下的 `ip` 配置项。

现在 `config.yaml` 配置如下：

```yaml
username: jianghushinian
password: 123456
server:
  ip: 127.0.0.1
  port: 8080
server.ip: 10.0.0.1
```

示例程序如下：

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
)

func main() {
	viper.SetConfigFile("./config.yaml")
	viper.ReadInConfig()

	// 读取配置值
	fmt.Printf("username: %v\n", viper.Get("username"))
	fmt.Printf("server: %v\n", viper.Get("server"))
	fmt.Printf("server.ip: %v\n", viper.Get("server.ip"))
	fmt.Printf("server.port: %v\n", viper.Get("server.port"))
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go 
username: jianghushinian
server: map[ip:127.0.0.1 port:8080]
server.ip: 10.0.0.1
server.port: 8080
```

`server.ip` 打印结果为 `10.0.0.1`，而不再是 `server` map 中所对应的值 `127.0.0.1`。

#### 提取子树

当使用 Viper 读取 `config.yaml` 配置文件后，`viper` 对象就包含了所有配置，并能通过 `viper.Get("server.ip")` 获取子配置。

我们可以将这份配置理解为一颗树形结构，`viper` 对象就包含了这个完整的树，可以使用如下方法获取 `server` 子树。

```go
srvCfg := viper.Sub("server")
```

使用示例如下：

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
)

func main() {
	viper.SetConfigFile("./config.yaml")
	viper.ReadInConfig()

	// 获取 server 子树
	srvCfg := viper.Sub("server")

	// 读取配置值
	fmt.Printf("ip: %v\n", srvCfg.Get("ip"))
	fmt.Printf("port: %v\n", srvCfg.Get("port"))
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go
ip: 127.0.0.1
port: 8080
```

#### 反序列化

Viper 提供了 2 个方法进行反序列化操作，以此来实现将所有或特定的值解析到结构体、map 等。

- `Unmarshal(rawVal interface{}) : error`：反序列化所有配置项。

- `UnmarshalKey(key string, rawVal interface{}) : error`：反序列化指定配置项。

使用示例如下：

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
)

type Config struct {
	Username string
	Password string
	// Viper 支持嵌套结构体
	Server struct {
		IP   string
		Port int
	}
}

func main() {
	viper.SetConfigFile("./config.yaml")
	viper.ReadInConfig()

	var cfg *Config
	if err := viper.Unmarshal(&cfg); err != nil {
		panic(err)
	}

	var password *string
	if err := viper.UnmarshalKey("password", &password); err != nil {
		panic(err)
	}

	fmt.Printf("cfg: %+v\n", cfg)
	fmt.Printf("password: %s\n", *password)
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go 
cfg: &{Username:jianghushinian Password:123456 Server:{IP:127.0.0.1 Port:8080}}
password: 123456
```

如果配置项的 `key` 本身就包含 `.`，则需要修改分隔符。

示例如下：

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
)

type Config struct {
	Chart struct {
		Values map[string]interface{}
	}
}

func main() {
	// 默认的键分隔符为 `.`，这里将其修改为 `::`
	v := viper.NewWithOptions(viper.KeyDelimiter("::"))

	v.SetDefault("chart::values", map[string]interface{}{
		"ingress": map[string]interface{}{
			"annotations": map[string]interface{}{
				"traefik.frontend.rule.type":                 "PathPrefix",
				"traefik.ingress.kubernetes.io/ssl-redirect": "true",
			},
		},
	})

	var cfg *Config
	if err := v.Unmarshal(&cfg); err != nil {
		panic(err)
	}

	fmt.Printf("cfg: %+v\n", cfg)
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go 
cfg: &{Chart:{Values:map[ingress:map[annotations:map[traefik.frontend.rule.type:PathPrefix traefik.ingress.kubernetes.io/ssl-redirect:true]]]}}
```

> 注意⚠️：Viper 在后台使用 [mapstructure](github.com/mitchellh/mapstructure) 来解析值，其默认情况下使用 `mapstructure` tags。当我们需要将 Viper 读取的配置反序列到结构体中时，如果出现结构体字段跟配置项不匹配，则可以设置 `mapstructure` tags 来解决。

#### 序列化

一个好用的配置包不仅能够支持反序列化操作，还要支持序列化操作。Viper 支持将配置序列化成字符串，或直接序列化到文件中。

##### 序列化成字符串

我们可以将全部配置序列化配置为 YAML 格式字符串。

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
	yaml "gopkg.in/yaml.v2"
)

// 序列化配置为 YAML 格式字符串
func yamlStringSettings() string {
	c := viper.AllSettings() // 获取全部配置
	bs, _ := yaml.Marshal(c) // 根据需求序列化成不同格式
	return string(bs)
}

func main() {
	viper.SetConfigFile("./config.yaml")
	viper.ReadInConfig()

	fmt.Printf(yamlStringSettings())
}
```

执行以上示例代码得到如下输出：

```bash
$ go run main.go
password: 123456
server:
  ip: 127.0.0.1
  port: 8080
username: jianghushinian
```

##### 写入配置文件

Viper 还支持直接将配置序列化到文件中，提供了如下几个方法：

- `WriteConfig`：将当前的 `viper` 配置写入预定义路径。如果没有预定义路径，则会报错。如果预定义路径已经存在配置文件，将会被覆盖。

- `SafeWriteConfig`：将当前的 `viper` 配置写入预定义路径。如果没有预定义路径，则会报错。如果预定义路径已经存在配置文件，不会覆盖，会报错。

- `WriteConfigAs`： 将当前的 `viper` 配置写入给定的文件路径。如果给定的文件路径已经存在配置文件，将会被覆盖。

- `SafeWriteConfigAs`：将当前的 `viper` 配置写入给定的文件路径。如果给定的文件路径已经存在配置文件，不会覆盖，会报错。

使用示例：

```go
viper.WriteConfig() // 将当前配置写入由 `viper.AddConfigPath()` 和 `viper.SetConfigName` 设置的预定义路径。
viper.SafeWriteConfig()
viper.WriteConfigAs("/path/to/my/.config")
viper.SafeWriteConfigAs("/path/to/my/.config") // 将会报错，因为它已经被写入了。
viper.SafeWriteConfigAs("/path/to/my/.other_config")
```

### 多实例对象

由于大多数应用程序都希望使用单个配置实例对象来管理配置，因此 viper 包默认提供了这一功能，它类似于一个单例。当我们使用 Viper 时不需要配置或初始化，Viper 实现了开箱即用的效果。

在上面的所有示例中，演示了如何以单例方式使用 Viper。我们还可以创建多个不同的 Viper 实例以供应用程序中使用，每个实例都有自己单独的一组配置和值，并且它们可以从不同的配置文件、key/value 存储等位置读取配置信息。

Viper 包支持的所有功能都被镜像为 `viper` 对象上的方法，这种设计思路在 Go 语言中非常常见，如标准库中的 log 包。

多实例使用示例：

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
)

func main() {
	x := viper.New()
	y := viper.New()

	x.SetConfigFile("./config.yaml")
	x.ReadInConfig()
	fmt.Printf("x.username: %v\n", x.Get("username"))

	y.SetDefault("username", "江湖十年")
	fmt.Printf("y.username: %v\n", y.Get("username"))
}
```

在这里，我创建了两个 Viper 实例 `x` 和 `y`，它们分别从配置文件读取配置和通过默认值的方式设置配置，使用时互不影响，使用者可以自行管理它们的生命周期。

执行以上示例代码得到如下输出：

```bash
$ go run main.go
x.username: jianghushinian
y.username: 江湖十年
```

### 使用建议

Viper 提供了众多方法可以管理配置，在实际项目开发中我们可以根据需要进行使用。如果是小型项目，推荐直接使用 `viper` 实例管理配置。

```go
package main

import (
	"fmt"

	"github.com/spf13/viper"
)

func main() {
	viper.SetConfigFile("./config.yaml")
	if err := viper.ReadInConfig(); err != nil {
		panic(fmt.Errorf("read config file error: %s \n", err.Error()))
	}

	// 监控配置文件变化
	viper.WatchConfig()

	// use config...
	fmt.Println(viper.Get("username"))
}
```

如果是中大型项目，一般都会有一个用来记录配置的结构体，可以使用 Viper 将配置反序列化到结构体中。

```go
package main

import (
	"fmt"

	"github.com/fsnotify/fsnotify"
	"github.com/spf13/viper"
)

type Config struct {
	Username string
	Password string
	// Viper 支持嵌套结构体
	Server struct {
		IP   string
		Port int
	}
}

func main() {
	viper.SetConfigFile("./config.yaml")
	if err := viper.ReadInConfig(); err != nil {
		panic(fmt.Errorf("read config file error: %s \n", err.Error()))
	}

	// 将配置信息反序列化到结构体中
	var cfg *Config
	if err := viper.Unmarshal(&cfg); err != nil {
		panic(fmt.Errorf("unmarshal config error: %s \n", err.Error()))
	}

	// 注册每次配置文件发生变更后都会调用的回调函数
	viper.OnConfigChange(func(e fsnotify.Event) {
		// 每次配置文件发生变化，需要重新将其反序列化到结构体中
		if err := viper.Unmarshal(&cfg); err != nil {
			panic(fmt.Errorf("unmarshal config error: %s \n", err.Error()))
		}
	})

	// 监控配置文件变化
	viper.WatchConfig()

	// use config...
	fmt.Println(cfg.Username)
}
```

需要注意的是，直接使用 `viper` 实例管理配置的情况下，当我们通过 `viper.WatchConfig()` 监听了配置文件变化，如果配置变化，则变化会立刻体现在 `viper` 实例对象上，下次通过 `viper.Get()` 获取的配置即为最新配置。但是在使用结构体管理配置时，`viper` 实例对象变化了，记录配置的结构体 `Config` 是不会自动更新的，所以需要使用 `viper.OnConfigChange` 在回调函数中重新将变更后的配置反序列化到 `Config` 中。

### 总结

本文探讨 Viper 的各种用法和使用场景，首先说明了为什么使用 Viper，它的优势是什么。

接着讲解了 Viper 包中最核心的两个功能：如何把配置值读入 Viper 和从 Viper 中读取配置值。Viper 对着两个功能都提供了非常多的方法来支持。

然后又介绍了如何用 Viper 来管理多份配置，即使用多实例。

对于 Viper 的使用我也给出了自己的建议，针对小型项目，推荐直接使用 `viper` 实例管理配置，如果是中大型项目，则推荐使用结构体来管理配置。

最后，Viper 正在向着 v2 版本迈进，欢迎读者在[这里](https://docs.google.com/forms/d/e/1FAIpQLSeesGIS8Rya-iOXmXjUst8sT3NMSmdclBPa-aJT3HkPjDWnng/viewform)分享想法，也期待下次来写一篇 v2 版本的文章与读者一起学习进步。

**联系我**

- 微信：jianghushinian

- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)

- 博客地址：https://jianghushinian.cn/

**参考**

- Viper 源码仓库：https://github.com/spf13/viper

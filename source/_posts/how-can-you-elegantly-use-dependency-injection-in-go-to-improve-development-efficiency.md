---
title: '在 Go 中如何优雅的使用 wire 依赖注入工具提高开发效率？'
date: 2024-06-09 15:05:19
tags:
- Go
categories:
- Go
---

如果你做过 Java 开发，那么想必一定听说或使用过**依赖注入**。依赖注入是一种软件设计模式，它允许将组件的依赖项外部化，从而使组件本身更加**模块化和可测试**。在 Java 中，依赖注入广泛应用于各种框架中，帮助开发者解耦代码和提高应用的灵活性。本文就来介绍下什么是依赖注入，以及在 Go 语言中如何实践依赖注入，提高 Go 项目的开发效率和可维护性。

<!-- more -->

### 什么是依赖注入？

正如前文所述，[依赖注入](https://zh.wikipedia.org/wiki/依赖注入)（dependency injection，缩写为 DI）是一种软件设计模式。

官方定义比较晦涩，我直接举个例子你就理解了。

在 Web 开发中，我们可以在 `store` 层（有些地方可会将其命名为 `repository`、`repo` 等）来操作数据库进行 CRUD。Go 语言中可以使用 GORM 操作数据库，所以 `store` 依赖 `*gorm.DB`，示例代码如下：

```go
type userStore struct {
	db *gorm.DB
}

func NewStore() *userStore {
	db := NewDB()
	return &userStore{db: db}
}

func (u *userStore) Create(ctx context.Context, user *model.UserM) error {
	return u.db.Create(&user).Error
}
```

> NOTE: 如果你对 GORM 不太了解，可以阅读我的另一篇文章[《Go 语言流行 ORM 框架 GORM 使用介绍》](https://jianghushinian.cn/2023/05/27/go-popular-orm-framework-gorm-introduction/)。


针对这一小段示例代码，我们可以按照如下方式创建一个用户：

```go
store := NewStore()
store.Create(ctx, user)
```

我们还可以将示例代码修改成这样：

```go
type userStore struct {
	db *gorm.DB
}

func NewStore(db *gorm.DB) *userStore {
	return &userStore{db: db}
}

func (u *userStore) Create(ctx context.Context, user *model.UserM) error {
	return u.db.Create(&user).Error
}
```

修改后示例代码中，我将 `*gorm.DB` 对象 `db` 的实例化过程，移动到了 `NewStore` 函数外面，在调用 `NewStore` 创建 `*userStore` 对象 `store` 时，将其通过参数形式传递进来。

现在，如果要创建一个用户，用法如下：

```go
db := NewDB()
store := NewStore(db)
store.Create(ctx, user)
```

没错，我们已经在使用**依赖注入**了。

我们还是使用 `store.Create(ctx, user)` 创建用户。但构造 `store` 时，`*userStore` **依赖** `*gorm.DB`，我们使用构造函数 `NewStore` 创建 `*userStore` 对象，并且将它的**依赖对象** `*gorm.DB` 通过函数参数的形式**注入**进来，这种编程思想，就叫「依赖注入」。

回想一下，我们平时在编写 Go 代码的过程中，为了方便测试，是不是经常将某个方法的依赖项通过参数传递进来，而非在方法内部实例化，这就是在使用依赖注入编写代码。

我在文章[《在 Go 中如何编写出可测试的代码》](https://jianghushinian.cn/2023/07/22/how-to-write-testable-code-in-go/#%E4%BD%BF%E7%94%A8%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5%E6%9D%A5%E8%A7%A3%E5%86%B3%E5%A4%96%E9%83%A8%E4%BE%9D%E8%B5%96)中就有提到**如何使用依赖注入来解决外部依赖问题**，你可以点击文章进行阅读。

在 Go 中使用依赖注入的核心目的，就是为了解耦代码。这样做的主要好处是：

1. 方便测试。依赖由外部注入，方便使用 `fake object` 来替换依赖项。
2. 每个对象仅需要初始化一次，其他方法都可以复用。比如使用 `db := NewDB()` 初始化得到一个 `*gorm.DB` 对象，在 `NewUserStore(db)` 时可以使用，在 `NewPostStore(db)` 时还可以使用。

> NOTE: 我不太喜欢使用比较官方的话术来讲解技术，因为本来技术就需要理解成本，而官方的定义往往晦涩难懂。为了降低读者的心智负担，我更喜欢用白话讲解。
> 但说到「依赖注入」，定会有人提及「控制反转」。为了不一些让读者产生困惑，这里简单说明下控制反转和依赖注入的关系：[控制反转](https://zh.wikipedia.org/wiki/控制反转)（英语：Inversion of Control，缩写为 IoC），是面向对象编程中的一种设计原则，可以用来减低计算机代码之间的耦合度。其中最常见的方式叫做[依赖注入](https://zh.wikipedia.org/wiki/依赖注入)（dependency injection，缩写为 DI）。
> 我们可以简单的将控制反转理解为一种思想，而依赖注入是这一思想的具体实现方式。

### 依赖注入工具 Wire 简介

wire 是一个由 Google 开发的自动依赖注入框架，专门用于 Go 语言。wire 通过**代码生成而非运行时反射**来实现依赖注入，这与许多其他语言中的依赖注入框架不同。这种方法使得注入的代码在编译时就已经确定，从而提高了性能并保证了代码的可维护性。

#### 安装 Wire

wire 分成两部分，一个是在项目中使用的 Go 包，用于在代码中引用 wire 代码；另一个是命令行工具，用于生成依赖注入代码。

- 在项目中导入需要先通过 `go get` 获取 wire 依赖包。

```bash
$ go get -u github.com/google/wire
```

在 Go 代码中像其他 Go 包一样使用：

```go
import "github.com/google/wire"
```

- 使用 `go install` 可以安装 wire 命令工具。

```bash
$ go install github.com/google/wire/cmd/wire
```

安装后通过 `--help` 标志执行 `wire` 命令查看其支持的所有子命令：

```bash
$ wire --help   
Usage: wire <flags> <subcommand> <subcommand args>

Subcommands:
        check            print any Wire errors found
        commands         list all command names
        diff             output a diff between existing wire_gen.go files and what gen would generate
        flags            describe all known top-level flags
        gen              generate the wire_gen.go file for each package
        help             describe subcommands and their syntax
        show             describe all top-level provider sets
```

由于绝大多数 wire 子命令不常用，所以这部分会放在本文最后再来讲解。

### Wire 快速开始

示例程序 `main.go` 代码如下：

```go
package main

import "fmt"

type Message string

func NewMessage() Message {
	return Message("Hi there!")
}

type Greeter struct {
	Message Message
}

func NewGreeter(m Message) Greeter {
	return Greeter{Message: m}
}

func (g Greeter) Greet() Message {
	return g.Message
}

type Event struct {
	Greeter Greeter
}

func NewEvent(g Greeter) Event {
	return Event{Greeter: g}
}

func (e Event) Start() {
	msg := e.Greeter.Greet()
	fmt.Println(msg)
}
```

示例代码很好理解，定义了 `Message` 类型是 `string` 的类型别名。定义了 `Greeter` 类型及其构造函数 `NewGreeter`，并且接收 `Message` 作为参数，`Greeter.Greet` 方法会返回 `Message` 信息。最后还定义了一个 `Event` 类型，它存储了 `Greeter`，`Greeter` 通过构造函数 `NewEvent` 参数传递进来，`Event.Start` 方法会代理到 `Greeter.Greet` 方法。

定义如下 `main` 函数来执行这个示例程序：

```go
func main() {
	message := NewMessage()
	greeter := NewGreeter(message)
	event := NewEvent(greeter)

	event.Start()
}
```

执行示例代码，得到如下输出：

```bash
$ go run main.go
Hi there!
```

可以发现，`main` 函数内部的代码有着明显的依赖关系，`NewEvent` 依赖 `NewGreeter`，`NewGreeter` 又依赖 `NewMessage`。

```
NewEvent -> NewGreeter -> NewMessage
```

我们可以将这部分代码进行抽离，封装到 `InitializeEvent` 函数中，保持入口函数 `main` 足够整洁，修改后代码如下：

```go
func InitializeEvent() Event {
	message := NewMessage()
	greeter := NewGreeter(message)
	event := NewEvent(greeter)
	return event
}

func main() {
	event := InitializeEvent()
	event.Start()
}
```

现在是时候让 wire 登场了，在 `main.go` 同级目录创建 `wire.go` 文件（这是一个约定俗称的文件命名，不是强制约束）： 

```go
//go:build wireinject

package main

import (
	"github.com/google/wire"
)

func InitializeEvent() Event {
	wire.Build(NewEvent, NewGreeter, NewMessage)
	return Event{}
}
```

我们将 `main.go` 文件中的 `InitializeEvent` 函数迁移过来，并且修改了内部逻辑，不再手动调用每个构造函数，而是将它们依次传递给 `wire.Build` 函数，然后使用 `return` 返回一个空的 `Event{}` 对象。

现在在当前目录下执行 `wire` 命令：

```bash
$ wire gen .    
wire: github.com/jianghushinian/blog-go-example/wire/getting-started: wrote /Users/jianghushinian/projects/blog-go-example/wire/getting-started/wire_gen.go
```

其中：

- `gen` 是 `wire` 的子命令，他会扫描指定包中使用了 `wire.Build` 的代码，然后为其生成一个 `wire_gen.go` 的文件。
- `.` 表示当前目录，用于指定包，不指定的话默认就是当前目录。如果项目下有很多包，可以使用 `./...` 表示全部包，这个参数其实跟我们执行 `go test` 测试时是一个道理。

根据输出结果可以发现，`wire` 命令为我们在当前目录下生成了 `wire_gen.go` 文件，内容如下：

```go
// Code generated by Wire. DO NOT EDIT.

//go:generate go run -mod=mod github.com/google/wire/cmd/wire
//go:build !wireinject
// +build !wireinject

package main

// Injectors from wire.go:

func InitializeEvent() Event {
	message := NewMessage()
	greeter := NewGreeter(message)
	event := NewEvent(greeter)
	return event
}
```

神奇的事情发生了，`wire` 为我们生成了 `InitializeEvent` 函数的代码，并且跟我们自己实现的代码一模一样。

这就是 wire 的威力，它可以为我们自动生成依赖注入代码，只需要我们将所有依赖项（这里是几个构造函数）传给 `wire.Build` 即可。

由于现在当前目录下存在 3 个 `.go` 文件：

```bash
$ tree    
.
├── go.mod
├── go.sum
├── main.go
├── wire.go
└── wire_gen.go
```

所以不能再使用 `go run main.go` 来执行示例代码了，可以使用 `go run .` 来执行：

```bash
$ go run .
Hi there!
```

细心的你可能会觉得疑惑🤔，代码中有两处 `InitializeEvent` 函数的定义，程序编译执行的时候不会报错吗？

我们在 `wire.go` 中定义了 `InitializeEvent` 函数：

```go
func InitializeEvent() Event {
	wire.Build(NewEvent, NewGreeter, NewMessage)
	return Event{}
}
```

然后 `wire` 命令帮我们在 `wire_gen.go` 中生成了新的 `InitializeEvent` 函数：

```go
func InitializeEvent() Event {
	message := NewMessage()
	greeter := NewGreeter(message)
	event := NewEvent(greeter)
	return event
}
```

而且这二者都是在同一个包下。

程序没有编译报错，主要取决于 `wire.go` 和 `wire_gen.go` 文件中的 `//go:build` 注释。

在 `wire.go` 文件中，注释为：

```go
//go:build wireinject
```

首先 `//go:build` 叫[构建约束](https://pkg.go.dev/go/build#hdr-Build_Constraints)（build constraint）或构建标记（build tag），是一个必须放在 `.go` 文件最开始的注释代码。有了它之后，我们就可以告诉 `go build` 如何来构建代码。

其次，`wireinject` 是传递给构建约束的选项。选项就相当于一个 `if` 判断条件，可以根据选项来定制构建时如何处理 Go 文件。

这个构建约束有两个作用：

- 将此文件标记文件为 `wire` 处理的目标：`//go:build wireinject` 告诉 `wire` 工具及开发者，该文件包含使用 `wire` 进行依赖注入的设置。即这通常意味着文件中包含了 `wire.Build` 函数调用。有了它，文件才会被 `wire` 识别。
- 条件编译：确保在正常的构建过程中，带有这个构建约束的文件不会被编译进最终的可执行文件中。它只有在使用 `wire` 工具生成依赖注入代码时才被处理。这也是为什么代码不会编译报错，其实 `wire.go` 文件只是给 `wire` 命令用的，`go run .` 执行的是 `main.go` 和 `wire_gen.go` 两个文件，会忽略 `wire.go`。

注意⚠️：`//go:build wireinject` 和 `package main` 之间需要保留一个空行，否则程序会报错。你记住就行，不必过于纠结于此，这个问题在 wire 仓库的 [issues 117](https://github.com/google/wire/issues/117) 中也有提及。

我们再来看 `wire_gen.go` 文件，注释为：

```go
// Code generated by Wire. DO NOT EDIT.

//go:generate go run -mod=mod github.com/google/wire/cmd/wire
//go:build !wireinject
// +build !wireinject
```

第一行注释仅作为提示用，无特殊用途。

`//go:generate` 这行注释是一个 [go generate](https://go.dev/blog/generate) 指令。`go generate` 是一个由 Go 工具链提供的命令，用于在编译前自动执行生成代码的命令。这个特定的生成指令告诉 Go 在执行 `go generate` 命令时，运行 `wire` 工具来自动生成或更新 `wire_gen.go` 文件。

`go run -mod=mod github.com/google/wire/cmd/wire` 这部分指令运行 `wire` 命令，其中 `-mod=mod` 确保使用的是项目的 `go.mod` 文件中指定的依赖版本。

我们也可以验证下：

```bash
$ go generate
wire: github.com/jianghushinian/blog-go-example/wire/getting-started: wrote /Users/jianghushinian/projects/blog-go-example/wire/getting-started/wire_gen.go
```

执行 `go generate` 确实会自动执行 `wire` 命令。

注释 `//go:build !wireinject` 同样是一个构建约束。与 `wire.go` 中的约束不同，这里的 `!wireinject` 多了一个 `!`，`!` 在编程中通常是取反的意思，所以它用来告诉 `wire` 忽略此文件。因为这个文件是最终执行的代码，`wire` 并不需要知道此文件的存在。

最后一个注释 `// +build !wireinject` 其实还是一个构建约束，只不过这是旧版本的条件编译标记（在 Go 1.17 版本之前使用）。它的作用与 `//go:build !wireinject` 相同，确保向后兼容性。这意味着在较老的 Go 版本中，编译条件也能被正确处理。

### 为什么要使用依赖注入工具？

前文讲解了依赖注入思想，以及通过[快速开始](#Wire-快速开始)的示例程序，我们极速入门了 wire 依赖注入工具的使用。

不过直到到现在我们都还没有讨论过为什么要使用依赖注入工具？

其实通过前文的示例，我们应该已经体会到，wire 最大的作用就是解放双手，提高生产力。

示例程序中，依赖链只有 3 个对象，一个中大型项目，依赖对象可能有几十个，wire 的作用会愈加明显。

使用依赖注入思想可以有效的解耦代码，那么使用依赖注入工具则进一步提高了生产力。我们无需手动实例化所有的依赖对象，仅需要编写函数声明，将依赖项扔给 `wire.Build`，`wire` 命令就能自动生成代码，可见 wire 是我们偷懒的利器，毕竟懒才是程序员的第一驱动力 :)。

如果你对依赖注入工具的作用还存在质疑，请接着往下看！

### Wire 核心概念

我们已经通过[快速开始](#Wire-快速开始)示例演示了 wire 的核心能力，现在是时候正式介绍下 wire 中的概念了。

在 wire 中，有两个核心概念：`providers`（提供者）和 `injectors`（注入器）。

这两个概念也很好理解，前文中的 `NewEvent`、`NewGreeter`、`NewMessage` 都是一个 `provider`。简单一句话：`provider` 就是一个可以**产生值的函数**，这些函数都是**普通的 Go 函数**。

值得注意的是，`provider` 必须是可导出的函数，即函数名称首字母大写。

而 `InitializeEvent` 实际上就是一个 `injector`。`injector` 是一个按依赖顺序调用 `provider` 的函数，该函数声明的主体是对 `wire.Build` 的调用。使用 wire 时，我们仅需编写 `injector` 的签名，然后由 `wire` 命令生成函数体。

### Wire 高级用法

wire 还有很多高级用法，值得介绍一下。

#### injector 函数参数和返回错误

首先是 `injector` 函数支持传参和返回 `error`。

修改示例程序代码如下：

```bash
type Message string

// 接收参数作为消息内容
func NewMessage(phrase string) Message {
	return Message(phrase)
}

type Greeter struct {
	Message Message
}

func NewGreeter(m Message) Greeter {
	return Greeter{Message: m}
}

func (g Greeter) Greet() Message {
	return g.Message
}

type Event struct {
	Greeter Greeter
}

// 增加返回错误信息
func NewEvent(g Greeter) (Event, error) {
	// 模拟创建 Event 报错
	if time.Now().Unix()%2 == 0 {
		return Event{}, errors.New("new event error")
	}
	return Event{Greeter: g}, nil
}

func (e Event) Start() {
	msg := e.Greeter.Greet()
	fmt.Println(msg)
}
```

这里主要修改了两处代码，`NewMessage` 接收一个字符串类型的参数作为消息内容，在 `NewEvent` 内部模拟了创建 `Event` 出错的场景，并将错误返回。

现在 `InitializeEvent` 函数定义如下：

```go
func InitializeEvent(phrase string) (Event, error) {
	wire.Build(NewEvent, NewMessage, NewGreeter)
	return Event{}, nil
}
```

这次传给 `wire.Build` 的 3 个构造函数顺序不同，可见顺序并不重要。但为了代码可维护性，我建议还是要按照依赖顺序依次传入 `provider`。

这里返回值增加了 `error`，所以 `return` 的值增加了一个 `nil`，其实我们返回什么并不重要，只要返回的类型正确即可（确保编译通过），因为最终生成的代码返回值是由 `wire` 生成的。

> NOTE: 为了逻辑清晰，我只贴出核心代码，并且 `wire.go` 文件也不再贴出 `//go:build wireinject` 相关代码，后文也是如此，你在实践时不要忘记。完整代码详见文末给出的 [GitHub 地址](https://github.com/jianghushinian/blog-go-example/tree/main/wire)。

使用 `wire` 生成代码如下：

```go
func InitializeEvent(phrase string) (Event, error) {
	message := NewMessage(phrase)
	greeter := NewGreeter(message)
	event, err := NewEvent(greeter)
	if err != nil {
		return Event{}, err
	}
	return event, nil
}
```

现在我们执行示例代码，可能出现两种情况：

```bash
$ go run .
Hello World!
```

或者：

```bash
$ go run .
new event error
```

#### 使用 ProviderSet 进行分组

wire 为我们提供了 `provider sets`，顾名思义，它可以包含一组 `providers`。使用 `wire.NewSet` 函数可以将多个 `provider` 添加到一个集合中。

我们把 `NewMessage`、`NewGreeter` 两个构造函数合并成一个 `provider sets`：

```go
var providerSet wire.ProviderSet = wire.NewSet(NewMessage, NewGreeter)
```

`wire.NewSet` 接收不定长参数，并将它们组装成一个 `wire.ProviderSet` 类型返回。

`wire.Build` 可以直接接收 `wire.ProviderSet` 类型，现在我们只需要给它传递两个参数即可：

```go
func InitializeEvent(phrase string) (Event, error) {
	wire.Build(NewEvent, providerSet)
	return Event{}, nil
}
```

使用 `wire` 生成代码如下：

```go
func InitializeEvent(phrase string) (Event, error) {
	message := NewMessage(phrase)
	greeter := NewGreeter(message)
	event, err := NewEvent(greeter)
	if err != nil {
		return Event{}, err
	}
	return event, nil
}
```

与之前生成的代码一模一样。

分组后，代码会更加清晰，每个 `provider sets` 仅包含一组关联的 `providers`，下文中实践部分你还能够看到 `provider sets` 更具有意义的用法。

#### 使用 Struct 定制 Provider

有时候一个 `struct` 比较简单，我们通常不会为其定义一个构造函数，此时我们可以使用 `wire.Struct` 作为 `provider`。

为了演示此功能，我们修改 `Message` 定义如下：

```go
type Message struct {
	Content string
	Code    int
}
```

修改 `InitializeEvent` 代码如下：

```go
func InitializeEvent(phrase string, code int) (Event, error) {
	wire.Build(NewEvent, NewGreeter, wire.Struct(new(Message), "Content"))
	return Event{}, nil
}
```

这里使用 `wire.Struct(new(Message), "Content")` 替代了原来的 `NewMessage` 作为一个 `provider`。

`wire.Struct` 函数签名如下：

```go
func Struct(structType interface{}, fieldNames ...string) StructProvider
```

`structType` 就是我们要使用的 `struct`，`fieldNames` 用来控制哪些字段会被赋值。

正常来说 `InitializeEvent` 的 `phrase` 参数会传给 `Message` 的 `Content` 字段，`code` 参数会传给 `Code`。wire 根据**参数类型**来判断应该将参数传给谁。

由于我们在这里仅显式指定了 `Content` 字段，所以最终只有 `Content` 字段会被赋值。

使用 `wire` 生成代码如下：

```go
func InitializeEvent(phrase string, code int) (Event, error) {
	message := Message{
		Content: phrase,
	}
	greeter := NewGreeter(message)
	event, err := NewEvent(greeter)
	if err != nil {
		return Event{}, err
	}
	return event, nil
}
```

从生成的代码可以发现，确实没给 `Code` 赋值。

如果我们想给 `Code` 赋值，可以这样写：

```go
wire.Build(NewEvent, NewGreeter, wire.Struct(new(Message), "Content", "Code"))
```

也可以这样写：

```go
wire.Build(NewEvent, NewGreeter, wire.Struct(new(Message), "*"))
```

`*` 表示通配符，即使用 `Message` 所有字段。

使用 `wire.Struct` 的好处是少定义一个构造函数，并且可以定制使用字段。

#### 使用 Struct 字段作为 Provider

我们还可以指定 `struct` 的具体某个字段作为一个 `provider`。

这需要用到 `wire.FieldsOf`，函数签名如下：

```go
func FieldsOf(structType interface{}, fieldNames ...string) StructFields
```

可以发现它跟 `wire.Struct` 函数参数一样，区别是返回值不同。

现在示例代码如下：

```go
type Content string

type Message struct {
	Content Content
	Code    int
}

// NewMessage 注意，这里返回的是指针类型
func NewMessage(content string, code int) *Message {
	return &Message{
		Content: Content(content),
		Code:    code,
	}
}
```

`Message` 的 `Content` 字段修改为 `string` 的别名类型 `Content`，**这是有用意的，稍后讲解**。

为了演示更多种情况，这里采用日常开发中更加常用的场景，即构造函数返回 `struct` 的指针类型。

修改 `InitializeEvent` 代码如下：

```go
func InitializeMessage(phrase string, code int) Content {
	wire.Build(NewMessage, wire.FieldsOf(new(*Message), "Content"))
	return Content("")
}
```

因为示例代码中 `NewMessage` 返回 `*Message` 类型，而非 `Message` 类型，所以传递给 `wire.FieldsOf` 必须是 `new(*Message)` 而不是 `new(Message)`。

`InitializeMessage` 函数返回 `Content` 类型，而非 `Message` 类型。

使用 `wire` 生成代码如下：

```go
func InitializeMessage(phrase string, code int) Content {
	message := NewMessage(phrase, code)
	content := message.Content
	return content
}
```

根据生成的代码可以发现，在通过 `NewMessage` 函数创建 `*Message` 对象以后，会将 `message.Content` 字段提取出来并返回。

前文在讲解 [使用 Struct 定制 Provider](#使用-Struct-字段作为-Provider)时我提到过「wire 根据**参数类型**来判断应该将参数传给谁」。

其实不仅仅是参数，wire 规定 `injector` 函数的参数和返回值类型都必须唯一。不然 wire 无法对应上哪个值该给谁，这也是为什么我专门定义了 `Content` 类型作为 `Message` 的字段，因为 `InitializeMessage` 的参数 `phrase` 已经是 `string` 类型了，所以其返回值就不能是 `string` 类型了。

现在就来演示一下 `injector` 函数的参数和返回值类型出现重复的情况，我们可以尝试把 `Message` 改回去：

```go
type Message struct {
	Content string
	Code    int
}

// NewMessage 注意，这里返回的是指针类型
func NewMessage(content string, code int) *Message {
	return &Message{
		Content: content,
		Code:    code,
	}
}
```

`InitializeEvent` 函数返回值也改为 `string`：

```go
func InitializeMessage(phrase string, code int) string {
	wire.Build(NewMessage, wire.FieldsOf(new(*Message), "Content"))
	return ""
}
```

现在使用 `wire` 生成代码，会得到类似如下错误：

```bash
$ wire gen .
wire: wire.go:10:2: multiple bindings for string
        current:
        <- wire.FieldsOf (structfields.go:8:2)
        previous:
        <- argument phrase to injector function InitializeMessage (wire.go:7:1)
wire: github.com/jianghushinian/blog-go-example/wire/getting-started/advanced/structfields: generate failed
wire: at least one generate failure
```

> NOTE: 注意，这里为了展示清晰，我将输出的文件绝对路径进行了修改，去掉了路径部分，只保留了文件名，不影响输出语义。后文中可能也会如此。

根据错误信息 `multiple bindings for string` 可知，wire 不支持函数的参数和返回值类型出现重复。

所以，当遇到 `injector` 函数出现参数或返回值类型重复的情况，可以通过给类型定义别名来解决。

#### 绑定「值」作为 Provider

可以直接将一个**值**作为参数传给 `wire.Value` 来构造一个 `provider`。

定义 `Message`：

```go
type Message struct {
	Message string
	Code    int
}
```

我们可以直接实例化这个 `struct`，然后将其传给 `wire.Value`：

```go
func InitializeMessage() Message {
	// 假设没有提供 NewMessage，可以直接绑定值并返回
	wire.Build(wire.Value(Message{
		Message: "Binding Values",
		Code:    1,
	}))
	return Message{}
}
```

使用 `wire` 生成代码如下：

```go
func InitializeMessage() Message {
	message := _wireMessageValue
	return message
}

var (
	_wireMessageValue = Message{
		Message: "Binding Values",
		Code:    1,
	}
)
```

可以发现，实际上 wire 为我们定义了一个变量，并将这个变量作为 `InitializeMessage` 函数返回值。

这种拿来即用的方式，提供了非常大的便利。

`wire.Value` 接收任何**值类型**，所以不止 `struct`，一个普通的 `int`、`string` 等类型都可以，就交给你自己去尝试了。

#### 绑定「接口」作为 Provider

`provider` 依赖项或返回值并不总是**值**，很多时候是一个**接口**。

我们可以接将一个**接口**作为参数传给 `wire.InterfaceValue` 来构造一个 `provider`。

创建一个 `Write` 函数，它依赖一个 `io.Writer` 接口：

```go
func Write(w io.Writer, value any) {
	n, err := fmt.Fprintln(w, value)
	fmt.Printf("n: %d, err: %v\n", n, err)
}
```

同 `wire.Value` 用法类似，我们可以使用 `wire.InterfaceValue` 绑定接口：

```go
func InitializeWriter() io.Writer {
	wire.Build(wire.InterfaceValue(new(io.Writer), os.Stdout))
	return nil
}
```

使用 `wire` 生成代码如下：

```go
func InitializeWriter() io.Writer {
	writer := _wireFileValue
	return writer
}

var (
	_wireFileValue = os.Stdout
)
```

生成代码套路跟 `wire.Value` 没什么区别。

#### 绑定结构体到接口

有时候我们可能会写出如下代码：

```go
type Message struct {
	Content string
	Code    int
}

type Store interface {
	Save(msg *Message) error
}

type store struct{}

// 确保 store 实现了 Store 接口
var _ Store = (*store)(nil)

func New() *store {
	return &store{}
}

func (s *store) Save(msg *Message) error {
	return nil
}

func SaveMessage(s Store, msg *Message) error {
	fmt.Printf("save message: %+v\n", msg)
	return s.Save(msg)
}

func RunStore(msg *Message) error {
	s := New()
	return SaveMessage(s, msg)
}
```

`Store` 接口定义了一个 `Save` 方法用来保存 `Message`，定义了 `store` 结构体，结构体的指针 `*store` 实现了 `Store` 接口，所以 `store` 的构造函数 `New` 返回 `*store`。

我们还定义了 `SaveMessage` 方法，它接收两个参数，分别是 `Store` 接口以及 `*Message`。

最终定义的 `RunStore` 方法接收 `*Message`，并在内部创建 `*store`，然后将这两个变量传给 `SaveMessage` 保存消息。

假如我们想使用 `wire` 命令来生成 `RunStore` 函数，定义如下：

```go
func WireRunStore(msg *Message) error {
	wire.Build(SaveMessage, New)
	return nil
}
```

使用 `wire` 生成代码将得到报错：

```bash
$ wire gen .
wire: wire.go:7:1: inject WireRunStore: no provider found for github.com/jianghushinian/blog-go-example/wire/getting-started/advanced/bindingstruct.Store
        needed by error in provider "SaveMessage" (bindingstruct.go:29:6)
wire: github.com/jianghushinian/blog-go-example/wire/getting-started/advanced/bindingstruct: generate failed
wire: at least one generate failure
```

这是因为 **wire 的构建依靠参数类型，但不支持接口类型**。而 `SaveMessage` 的 `s Store` 参数就是接口。

此时，我们可以使用 `wire.Bind` 告诉 `wire` 工具，将一个结构体绑定到接口：

```go
func WireRunStore(msg *Message) error {
	// new(Store) 接口无需使用指针
	wire.Build(SaveMessage, New, wire.Bind(new(Store), new(*store)))
	return nil
}
```

这样，`wire` 就知道 `New` 创建得到的 `*store` 类型需要传递给 `SaveMessage` 的 `s Store` 参数了。

使用 `wire` 生成代码如下：

```go
func WireRunStore(msg *Message) error {
	bindingstructStore := New()
	error2 := SaveMessage(bindingstructStore, msg)
	return error2
}
```

没有问题。

#### 清理函数

有时候我们的函数返回值可能包含一个清理函数，用来释放资源，示例代码如下：

```go
func OpenFile(path string) (*os.File, func(), error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, nil, err
	}

	cleanup := func() {
		fmt.Println("cleanup...")
		if err := f.Close(); err != nil {
			fmt.Println(err)
		}
	}

	return f, cleanup, nil
}

func ReadFile(f *os.File) (string, error) {
	b := make([]byte, 1024)
	_, err := f.Read(b)
	if err != nil {
		return "", err
	}
	return string(b), nil
}
```

`OpenFile` 函数接收一个文件路径作为参数，其内部会打开这个文件，并返回文件对象 `*os.File`。除此以外，还会返回一个清理函数和 `error`，清理函数内部会调用 `f.Close()` 关闭文件对象。

`ReadFile` 函数依赖 `*os.File` 文件对象，可以读取并返回其内容。

我们可以定义如下 `injector` 函数：

```go
func InitializeFile(path string) (*os.File, func(), error) {
	wire.Build(OpenFile)
	return nil, nil, nil
}
```

使用 `wire` 生成代码如下：

```go
func InitializeFile(path string) (*os.File, func(), error) {
	file, cleanup, err := OpenFile(path)
	if err != nil {
		return nil, nil, err
	}
	return file, func() {
		cleanup()
	}, nil
}
```

可以发现，wire 能够正确处理这种情况。

不过，wire 规定清理函数签名只能为 `func()`。而 `InitializeFile` 函数的返回值，也是我们工作中使用 wire 的典型场景：`injector` 函数返回 3 个值，分别是对象、清理函数以及 `error`。

示例代码使用方式如下：

```go
f, cleanup, err := InitializeFile("testdata/multi.txt")
if err != nil {
	fmt.Println(err)
}
content, err := ReadFile(f)
if err != nil {
	fmt.Println(err)
}
fmt.Println(content)
cleanup()
```

还有一种情况，假如我们传递给的 `wire.Build` 多个 `provider` 都存在清理函数，这时候 `wire` 命名生成的代码会是什么样呢？

这个就当做作业留给你自己去尝试了。

> NOTE: 如果你懒得尝试🤣，其实我也写好了例子，你可以点击 [GitHub 地址](https://github.com/jianghushinian/blog-go-example/tree/main/wire/getting-started/advanced/cleanupfunctions/multi)进行查看。篇幅所限，我就不贴代码了，感兴趣可以点进去查看。

#### 备用注入器语法，给语法加点糖

前文讲过，`injector` 函数返回值并不重要，只要我们写在 `return` 后面的返回值类型，跟函数签名一致即可。因为 wire 会忽略它们，所以上面很多示例返回值我都使用 `nil` 来替代。

那么，既然返回值没什么用，我们是否可以偷个懒，不写 `return` 呢？

答案是可以的，我们可以直接 `panic`，这样程序依然可以通过编译。

示例代码如下：

```go
type Message string

func NewMessage(phrase string) Message {
	return Message(phrase)
}
```

这里直接在 `injector` 函数中使用 `panic` 来简化代码：

```go
func InitializeMessage(phrase string) Message {
	panic(wire.Build(NewMessage))
}
```

使用 `wire` 生成代码如下：

```go
func InitializeMessage(phrase string) Message {
	message := NewMessage(phrase)
	return message
}
```

没有任何问题。

这种方式少写了一行 `return`，算是 wire 给我们提供的一个“语法糖”。Kratos 框架[文档](https://go-kratos.dev/docs/guide/wire/)中也是这么写的，你可以点击查看。

至于到底选用 `return` 还是 `panic`，社区并没有一致的规范，看个人喜好就好。我目前更喜欢使用 `return`，毕竟谁都不希望自己程序出现 `panic`，占位也不行 :)。

至此，终于将 wire 的常用功能全部讲解完毕，接下来就进入 wire 生产实践了。

### Wire 生产实践

这里以一个 `user` 服务作为示例，演示下一个生产项目中是如何使用 wire 依赖注入工具的。

`user` 项目目录结构如下：

```bash
$ tree user
user
├── assets
│   ├── curl.sh
│   └── schema.sql
├── cmd
│   └── main.go
├── go.mod
├── go.sum
├── internal
│   ├── biz
│   │   └── user.go
│   ├── config
│   │   └── config.go
│   ├── controller
│   │   └── user.go
│   ├── model
│   │   └── user.go
│   ├── router.go
│   ├── store
│   │   └── user.go
│   ├── user.go
│   ├── wire.go
│   └── wire_gen.go
└── pkg
    ├── api
    │   └── user.go
    └── db
        └── db.go

12 directories, 16 files
```

> NOTE: `user` 项目[源码在此](https://github.com/jianghushinian/blog-go-example/tree/main/wire/user)，你可以点击查看，建议下载下来执行启动下程序，加深理解。

这是一个典型的 Web 应用，用来对用户进行 CRUD。不过为了保持代码简洁清晰，方便理解，`user` 项目仅实现了创建用户的功能。

我先简单介绍下各个目录的功能。

`assets` 努目录用于存放项目资源。`schema.sql` 中是建表语句，`curl.sh` 保存了一个 `curl` 请求命令，用于测试创建用户功能。

`cmd` 中当然是程序入口文件。

`internal` 下保存了项目业务逻辑。

`pkg` 目录存放可导出的公共库。`api` 用于存放请求对象；`db` 用于构造数据库对象。

项目设计了 4 层架构，`controller` 即对应 MVC 经典模式中的 Controller，`biz` 是业务层，`store` 层用于跟数据库交互，还有一个 `model` 层定义模型，用于映射数据库表。

`router.go` 用于注册路由。

`user.go` 用于定义创建和启动 `user` 服务的应用对象。

而 `wire.go` 和 `wire_gen.go` 两个文件就无需我过多讲解了。

> NOTE: 本项目目录结构遵循最佳实践，可以参考我的另一篇文章[《如何设计一个优秀的 Go Web 项目目录结构》](https://jianghushinian.cn/2023/02/25/how-to-design-a-good-go-web-project-directory-structure/)。

简单介绍完了目录结构，再来梳理下我们所设计的 4 层架构依赖关系：首先 `controller` 层依赖 `biz` 层，然后 `biz` 层又依赖 `store` 层，接着 `store` 层又依赖了数据库（即依赖 `pkg/db/`），而 `controller`、`biz`、`store` 这三者又都依赖 `model` 层。

现在看了我的讲解，你可能有些发懵，没关系，下面我将主要代码逻辑都贴出来，加深你的理解。

`assets/schema.sql` 中的建表语句如下：

```sql
CREATE TABLE `user`
(
    `id`        BIGINT       NOT NULL AUTO_INCREMENT,
    `email`     VARCHAR(255),
    `nickname`  VARCHAR(255),
    `username`  VARCHAR(255) NOT NULL,
    `password`  VARCHAR(255) NOT NULL,
    `createdAt` DATETIME,
    `updatedAt` DATETIME,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

`user` 项目仅有一张表。

`cmd/main.go` 代码如下：

```go
package main

import (
	user "github.com/jianghushinian/blog-go-example/wire/user/internal"
	"github.com/jianghushinian/blog-go-example/wire/user/internal/config"
	"github.com/jianghushinian/blog-go-example/wire/user/pkg/db"
)

func main() {
	cfg := &config.Config{
		MySQL: db.MySQLOptions{
			Address:  "127.0.0.1:3306",
			Database: "user",
			Username: "root",
			Password: "123456",
		},
	}

	app, cleanup, err := user.NewApp(cfg)
	if err != nil {
		panic(err)
	}

	defer cleanup()
	app.Run()
}
```

入口函数 `main` 中先创建了配置对象 `cfg`，接着实例化 `app` 对象，最后调用 `app.Run()` 启动 `user` 服务。

这也是一个典型的 Web 应用启动步骤。

`Config` 定义如下：

```go
type Config struct {
	MySQL db.MySQLOptions `json:"mysql" yaml:"mysql"`
}

type MySQLOptions struct {
	Address  string
	Database string
	Username string
	Password string
}
```

`user.go` 中的 `App` 定义如下：

```go
// App 代表一个 Web 应用
type App struct {
	*config.Config

	g  *gin.Engine
	uc *controller.UserController
}

// NewApp Web 应用构造函数
func NewApp(cfg *config.Config) (*App, func(), error) {
	gormDB, cleanup, err := db.NewMySQL(&cfg.MySQL)
	if err != nil {
		return nil, nil, err
	}
	
	userStore := store.New(gormDB)
	userBiz := biz.New(userStore)
	userController := controller.New(userBiz)
	
	engine := gin.Default()
	app := &App{
		Config: cfg,
		g:      engine,
		uc:     userController,
	}

	return app, cleanup, err
}

// Run 启动 Web 应用
func (a *App) Run() {
	// 注册路由
	InitRouter(a)

	if err := a.g.Run(":8000"); err != nil {
		panic(err)
	}
}
```

`App` 代表一个 Web 应用，它嵌入了配置、`gin` 框架的 `*Engine` 对象，以及 `controller`。

`NewApp` 是 `App` 的构造函数，通过 `Config` 来创建一个 `*App` 对象。

根据其内部代码逻辑，也能看出项目的 4 层架构依赖关系：创建 `App` 对象依赖 `Config`，`Config` 是通过参数传递进来的；`*Engine` 对象可以通过 `gin.Default()` 得到；而 `userController` 则通过 `controller.New` 创建，`controller` 依赖 `biz`，`biz` 依赖 `store`，`store` 依赖 `*gorm.DB`。

可以发现，依赖关系非常清晰，并且我们使用了依赖注入思想编写代码，那么此时，正是 wire 的用武之地。

不过，我们先不急着讲解如何在这里使用 wire。我先将项目剩余主要代码贴出来，便于你理解这个 Web 应用。

我们可以通过 `pkg/db/db.go` 中的 `NewMySQL` 创建出 `*gorm.DB` 对象：

```go
// NewMySQL 根据选项构造 *gorm.DB
func NewMySQL(opts *MySQLOptions) (*gorm.DB, func(), error) {
	// 可以用来释放资源，这里仅作为示例使用，没有释放任何资源，因为 gorm 内部已经帮我们做了
	cleanFunc := func() {}

	db, err := gorm.Open(mysql.Open(opts.DSN()), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Silent),
	})
	return db, cleanFunc, err
}
```

有了 `*gorm.DB` 就可以创建 `store` 对象了，`internal/store/user.go` 主要代码如下：

```go
package store

...

// ProviderSet 一个 Wire provider sets，用来初始化 store 实例对象，并将 UserStore 接口绑定到 *userStore 类型实现上
var ProviderSet = wire.NewSet(New, wire.Bind(new(UserStore), new(*userStore)))

// UserStore 定义 user 暴露的 CRUD 方法
type UserStore interface {
	Create(ctx context.Context, user *model.UserM) error
}

// UserStore 接口实现
type userStore struct {
	db *gorm.DB
}

// 确保 userStore 实现了 UserStore 接口
var _ UserStore = (*userStore)(nil)

// New userStore 构造函数
func New(db *gorm.DB) *userStore {
	return &userStore{db}
}

// Create 插入一条 user 记录
func (u *userStore) Create(ctx context.Context, user *model.UserM) error {
	return u.db.Create(&user).Error
}
```

有了 `store` 就可以创建 `biz` 对象了，`internal/biz/user.go` 主要代码如下：

```go
package biz

...

// ProviderSet 一个 Wire provider sets，用来初始化 biz 实例对象，并将 UserBiz 接口绑定到 *userBiz 类型实现上
var ProviderSet = wire.NewSet(New, wire.Bind(new(UserBiz), new(*userBiz)))

// UserBiz 定义 user 业务逻辑操作方法
type UserBiz interface {
	Create(ctx context.Context, r *api.CreateUserRequest) error
}

// UserBiz 接口的实现
type userBiz struct {
	s store.UserStore
}

// 确保 userBiz 实现了 UserBiz 接口
var _ UserBiz = (*userBiz)(nil)

// New userBiz 构造函数
func New(s store.UserStore) *userBiz {
	return &userBiz{s: s}
}

// Create 创建用户
func (b *userBiz) Create(ctx context.Context, r *api.CreateUserRequest) error {
	var userM model.UserM
	_ = copier.Copy(&userM, r)

	return b.s.Create(ctx, &userM)
}
```

接着，有了 `biz` 就可以创建 `controller` 对象了，`internal/controller/user.go` 主要代码如下：

```go
package controller

...

// UserController 用来处理用户请求
type UserController struct {
	b biz.UserBiz
}

// New controller 构造函数
func New(b biz.UserBiz) *UserController {
	return &UserController{b: b}
}

// Create 创建用户
func (ctrl *UserController) Create(c *gin.Context) {
	var r api.CreateUserRequest
	if err := c.ShouldBindJSON(&r); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{
			"err": err.Error(),
		})
		return
	}

	if err := ctrl.b.Create(c, &r); err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{
			"err": err.Error(),
		})
		return
	}

	c.JSON(http.StatusOK, gin.H{})
}
```

这些对象都有了，就可以调用 `NewApp` 构造出 `App` 了。

`App` 在启动前，还会调用 `InitRouter` 进行路由注册：

```go
// InitRouter 初始化路由
func InitRouter(a *App) {
	// 创建 users 路由分组
	u := a.g.Group("/users")
	{
		u.POST("", a.uc.Create)
	}
}
```

现在 `user` 项目逻辑已经清晰了，是时候启动应用程序了：

```bash
$ cd user   
$ go run cmd/main.go
```

程序启动后，会监听 `8000` 端口，可以使用 `assets/curl.sh` 中的 `curl` 命令进行访问：

```bash
$ curl --location --request POST 'http://127.0.0.1:8000/users' \
--header 'Content-Type: application/json' \
--data-raw '{
    "email": "jianghushinian007@outlook.com",
    "nickname": "江湖十年",
    "username": "jianghushinian",
    "password": "pass"
}'
```

不出意外，你将在数据库中看到新创建的用户。

执行以下 SQL：

```sql
USE user;
SELECT * FROM user;
```

将输出新创建出来的用户。

```bash
+----+-------------------------------+----------+----------------+----------+---------------------+---------------------+
| id | email                         | nickname | username       | password | createdAt           | updatedAt           |
+----+-------------------------------+----------+----------------+----------+---------------------+---------------------+
|  1 | jianghushinian007@outlook.com | 江湖十年  | jianghushinian | pass     | 2024-06-11 00:01:35 | 2024-06-11 00:01:35 |
+----+-------------------------------+----------+----------------+----------+---------------------+---------------------+
```

现在，是时候讨论如何在 `user` 项目中使用 wire 来提高开发效率了。

回顾下 `NewApp` 的定义：

```go
// NewApp Web 应用构造函数
func NewApp(cfg *config.Config) (*App, func(), error) {
	gormDB, cleanup, err := db.NewMySQL(&cfg.MySQL)
	if err != nil {
		return nil, nil, err
	}
	
	userStore := store.New(gormDB)
	userBiz := biz.New(userStore)
	userController := controller.New(userBiz)
	
	engine := gin.Default()
	app := &App{
		Config: cfg,
		g:      engine,
		uc:     userController,
	}

	return app, cleanup, err
}
```

其实这里面一层层的依赖注入，都是套路代码，基本上一个 Web 应用都可以按照这个套路来写。

这就涉及到套路代码写多了其实是比较烦的，这还只是一个微型项目，如果是中大项目，可以预见这个 `NewApp` 代码量会很多，所以是时候让 wire 出场了：

```go
func NewApp(cfg *config.Config) (*App, func(), error) {
	engine := gin.Default()
	app, cleanup, err := wireApp(engine, cfg, &cfg.MySQL)

	return app, cleanup, err
}
```

我们可以将 `NewApp` 中的主逻辑全部拿走，放在 `wireApp` 中（在 `wire.go` 文件中）。

`wireApp` 定义如下：

```go
func wireApp(engine *gin.Engine, cfg *config.Config, mysqlOptions *db.MySQLOptions) (*App, func(), error) {
	wire.Build(
		db.NewMySQL,
		store.ProviderSet,
		biz.ProviderSet,
		controller.New,
		wire.Struct(new(App), "*"),
	)
	return nil, nil, nil
}
```

有了前文的讲解，其实这里无需我多言，你都能够看懂，因为并没有新的知识。

不过我们还是简单分析下这里都用到了 wire 的哪些特性。

首先 `wireApp` 返回值是典型的三件套：`(*App, func(), error)`，对象、清理函数和 `error`。

这里使用了两个 `wire.ProviderSet` 进行分组，定义如下：

```go
var ProviderSet = wire.NewSet(New, wire.Bind(new(UserStore), new(*userStore)))
var ProviderSet = wire.NewSet(New, wire.Bind(new(UserBiz), new(*userBiz)))
```

并且在构造 `wire.ProviderSet` 时，还使用了 `wire.Bind(new(UserStore), new(*userStore))` 将一个结构体绑定到接口。

最后，我们使用了 `struct` 作为 `provider`：`wire.Struct(new(App), "*")` ，通配符 `*` 用来表示所有字段。

在真实项目中，wire 就这么使用。

如果你觉得 `user` 项目太小，使用 wire 的价值还不够大。你可以看看 `onex` 项目，比如 [usercenter](https://github.com/superproj/onex/blob/master/internal/usercenter/wire.go) 中的代码，这个开源项目完全是生产级别。

### 为什么选择 Wire

通常来说，这部分内容是应该放在文章开头的。我将其放在这里，目的是为了让你熟悉 wire 后，再回过头来对比，wire 有哪些优势，加深你对为什么选择 wire 的理解。

其实 Go 生态中依赖注入工具不止有 Google 的 [wire](https://github.com/google/wire) 一家独大，还有 Uber 开源的 [dig](https://github.com/uber-go/dig)，以及 Facebook 开源的 [inject](github.com/facebookarchive/inject) 比较流行。

但我为什么要选择 wire？

一句话概括：wire 使用**代码生成**，而非**反射**。

 我们可以分别举例看下 dig 以及 inject 是如何使用的。

dig 的使用示例如下：

```go
package main

import (
	"fmt"
	"log"

	"go.uber.org/dig"
)


type User struct {
	name string
}

// NewUser - Creates a new instance of User
func NewUser(name string) User {
	return User{name: name}
}

// Get - A method with user as dependency
func (u *User) Get(message string) string {
	return fmt.Sprintf("Hello %s - %s", u.name, message)
}

// Run - Depends on user and calls the Get method on User
func Run(user User) {
	result := user.Get("It's nice to meet you!")
	fmt.Println(result)
}

func main() {
	// Initialize a new dig container
	container := dig.New()
	// Provide a name parameter to the container
	container.Provide(func() string { return "jianghushinian" })
	// Provide a new User instance to the container using the name injected above
	if err := container.Provide(NewUser); err != nil {
		log.Fatal(err)
	}
	// Invoke the Run function; Dig automatically injects the User instance provided above
	if err := container.Invoke(Run); err != nil {
		log.Fatal(err)
	}
}
```

简单解释下示例代码：

`dig.New()` 实例化一个 dig 容器。

`container.Provide(func() string { return "jianghushinian" })` 将一个匿名函数提供给容器。

然后调用 `container.Provide(NewUser)`，dig 首先将字符串值 `jianghushinian` 作为 `name` 参数提供给 `NewUser` 函数。之后，`NewUser` 函数会根据此值创建出来一个 `User` 结构体的新实例，随后 dig 将其提供给容器。

最后，`container.Invoke(Run)` 会将容器中保存的 `User` 结构体传递给 `Run` 函数并运行。

我们可以类比 wire 来学习 dig：可以把 `Provide` 看作 `providers`，`Invoke` 看作 `injectors`，这样就好理解了。

以上示例代码可以直接执行，无需像使用 wire 一样需要提前生成代码：

```bsah
$ go run main.go
Hello jianghushinian - It's nice to meet you!
```

这就是 dig 的使用。

再来看一个 inject 的使用示例：

```go
package main

import (
	"fmt"
	"log"

	"github.com/facebookgo/inject"
)

type User struct {
	Name string `inject:"name"`
}

// Get - A method with user as dependency
func (u *User) Get(message string) string {
	return fmt.Sprintf("Hello %s - %s", u.Name, message)
}

// Run - Depends on user and calls the Get method on User
func Run(user *User) {
	result := user.Get("It's nice to meet you!")
	fmt.Println(result)
}

func main() {
	// new an inject Graph
	var g inject.Graph

	// inject name
	name := "jianghushinian"

	// provide string value
	err := g.Provide(&inject.Object{Value: name, Name: "name"})
	if err != nil {
		log.Fatal(err)
	}

	// create a User instance and supply it to the dependency graph
	user := &User{}
	err = g.Provide(&inject.Object{Value: user})
	if err != nil {
		log.Fatal(err)
	}

	// resolve all dependencies
	err = g.Populate()
	if err != nil {
		log.Fatal(err)
	}

	Run(user)
}
```

这个示例代码我就不详细讲解了，学会了 wire 和 dig，这段代码很容易理解。

可以发现的是，无论是 dig 还是 inject，它们使用的都是运行时反射机制，来实现依赖注入功能。

这会带来最直观的两个问题：

1. 使用反射可能影响性能。
2. 我们需要根据工具的要求编写代码，而这份代码正确与否，只有在运行期间才能确定。也就是说，代码是“黑盒”的，通过 review 代码，很难一眼看出代码是否存在问题。

而 wire 采用代码生成，它会根据我们编写的 `injector` 函数签名，生成最终代码。所以在执行代码之前，我们就已经有了 `injector` 函数的源码。

这既不会影响性能，也不会让代码变成“黑盒”，在执行程序之前我们就知道代码长什么样。而这样做还能带来一个好处，能够大大简化我们排错的过程。

Python 之禅中有一句话叫「显式优于隐式」，wire 做到了。

### Wire 命令行工具

文章最后，我再来简单介绍下 `wire` 命令行工具。

之所以放在最后讲解，是因为 `wire` 的子命令确实不太常用，如果你去网上搜索，几乎没人介绍。不过为了保证文章的完整性，我还是简单讲解下，作为扩展内容，你好有个印象。

使用 `--help` 查看使用帮助信息。

```bash
$ wire --help
Usage: wire <flags> <subcommand> <subcommand args>

Subcommands:
        check            print any Wire errors found
        commands         list all command names
        diff             output a diff between existing wire_gen.go files and what gen would generate
        flags            describe all known top-level flags
        gen              generate the wire_gen.go file for each package
        help             describe subcommands and their syntax
        show             describe all top-level provider sets
```

可以发现 wire 连最基本的 `--version` 命令都不存在，即不支持查看版本信息。起初这点我是疑惑的，不过看了官方描述，也就不足为奇了。因为 wire 已经不再加入新功能，所以你可以理解为它就这一个版本。

官方描述说当前[项目状态](https://github.com/google/wire?tab=readme-ov-file#project-status)不接受新功能，只接受错误报告和 Bug fix。看来官方也想保持 wire 的简洁。

有人说项目不维护了。但我认为这又何尝不是一件好事情，其实项目还在维护，只是不增加新功能了。这在日新月异的技术行业里，是好事，极大的好事。我们不用投入太多精力学习这个工具，学一次受用很久。这也是我写这篇想着尽量把 wire 功能介绍完全，方便大家学习。

回归正题，首先要讲解的是 `gen` 子命令。已经是我们的老朋友了，可以根据我们编写的 `injector` 函数签名，自动生成目标代码。

其实如果直接使用 `wire` 命令，后面什么也不接，`wire` 默认会调用 `gen` 子命令：

```bash
$ wire       
wire: github.com/jianghushinian/blog-go-example/wire/getting-started: wrote /Users/jianghushinian/projects/blog-go-example/wire/getting-started/wire_gen.go
```

`check` 子命令可以帮我们检查代码错误，比如我们将 [Wire 快速开始](#Wire-快速开始) 部分的示例中的 `injector` 函数 `InitializeEvent` 故意写错。

`InitializeEvent` 代码如下：

```go
func InitializeEvent() Event {
	wire.Build(NewEvent, NewGreeter, NewMessage)
	return Event{}
}
```

现在修改成错误的，漏写了 `NewMessage` 方法：

```go
func InitializeEvent() Event {
	wire.Build(NewEvent, NewGreeter)
	return Event{}
}
```

使用 `wire check` 检查代码错误：

```bash
$ wire check
wire: wire.go:7:1: inject InitializeEvent: no provider found for github.com/jianghushinian/blog-go-example/wire/getting-started.Message
        needed by github.com/jianghushinian/blog-go-example/wire/getting-started.Greeter in provider "NewGreeter" (main.go:15:6)
        needed by github.com/jianghushinian/blog-go-example/wire/getting-started.Event in provider "NewEvent" (main.go:27:6)
wire: error loading packages
```

但其实我们直接执行 `wire` 命令生成代码时，也会得到相同的错误。

`commands` 子命令可以打印 `wire` 支持的所有子命令，嗯，仅此而已。

```bash
$ wire commands
commands
flags
help
check
diff
gen
show
```

`flags` 子命令可以打印每个子命令接收的标志：

```bash
$ wire flags gen
  -header_file string
        path to file to insert as a header in wire_gen.go
  -output_file_prefix string
        string to prepend to output file names.
  -tags string
        append build tags to the default wirebuild
```

可以发现 `gen` 子命令支持 3 个标志，至于效果你可以自行尝试。

`diff` 子命令用于打印 `wire` 生成的 `wire_gen.go` 文件和之前有何不同：

```bash
$ wire diff .    
github.com/jianghushinian/blog-go-example/wire/getting-started: diff from wire_gen.go:
@@ -11,2 +11,2 @@
-func InitializeEvent() Event {
-       message := NewMessage()
+func InitializeEvent(string2 string) Event {
+       message := NewMessage(string2)
```

`show` 子命令用于分析和展示指定包中的依赖注入配置：

```bash
$ wire show .    

Injectors:
        "github.com/jianghushinian/blog-go-example/wire/getting-started".InitializeEvent
```

`wire` 命令行工具的讲解就介绍到这里。

### 总结

终于到了总结环节，又是一篇万字长文。

本文主旨是为了讲解在 Go 中，如何优雅的使用 wire 依赖注入工具提高开发效率。

首先介绍了什么是依赖注入，以及在 Go 中如何使用依赖注入思想编写代码。

接着又对依赖注入工具 wire 进行了简单介绍，并安装了 `wire` 命令行工具。

然后通过一个 wire 快速开始的示例程序，极速入门 wire 的使用。

有了使用经验，我又讲解了为什么要使用 wire？因为它们帮我们自动生成依赖注入代码，提高开发效率。

接下来我对 wire 的核心概念进行了讲解。我们知道了什么是 `providers` 和 `injectors`，知道了这两个核心概念，wire 就入门了。

我还介绍了 wire 和很多高级特性。`injector` 函数支持参数，也支持返回清理函数和错误。我们可以使用 `ProviderSet` 对 `providers` 进行分组。可以使用 `wire.Struct` 将一个结构体作为 `provider`。也可以指定结构体的具体某个字段作为 `provider`。`wire.Value` 可以将一个值构造成 `provider`。`wire.InterfaceValue` 可以将一个接口构造成 `provider`。通过 `wire.Bind(new(Fooer), new(MyFoo)))` 可以将 `MyFoo` 结构体绑定到 `Fooer` 接口。wire 还为我们提供了备用注入器语法，可以使用 `panic` 取代在 `injector` 函数中编写返回值。

wire 的用法都讲解完成以后，我又以一个 `user` Web 应用作为案例，为你讲解了在生产实践中 wire 的使用。

既然我们学会了 wire，那就应该知道我们为什么要选择使用 wire。我对比了 Uber 开源的 dig，以及 Facebook 开源的 inject，为你讲解了选择 wire 的原因。可以用一句话概括：wire 使用代码生成，而非反射。

最后，我又简单介绍了 `wire` 命令行工具的使用。

记住，依赖注入并不神秘，wire 的作用也显而易见，就是为了解放双手。如果你更喜欢手动编写代码，那么也完全没有任何问题。不要过于神化依赖注入工具，起码在 Go 语言中是这样。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/wire) 中，欢迎点击查看。

由于篇幅所限，有些示例文章中并没有给出执行结果，你一定要把我的示例代码 `clone` 下来，依次执行一遍，这样才能更加深刻的理解。

至此本文完结，如果你想要更深入的了解 wire，那就去看它的[源码](https://github.com/google/wire)吧，祝你好运 :)。

希望此文能对你有所启发。

**延伸阅读**

- Compile-time Dependency Injection With Go Cloud's Wire: https://go.dev/blog/wire
- Wire README: https://github.com/google/wire/blob/main/README.md
- Wire Documentation: https://pkg.go.dev/github.com/google/wire/internal/wire
- Wire 源码: https://github.com/google/wire
- onex usercenter: https://github.com/superproj/onex/tree/master/internal/usercenter
- Go Dependency Injection with Wire: https://blog.drewolson.org/go-dependency-injection-with-wire/
- Golang Dependency Injection Using Wire: https://clavinjune.dev/en/blogs/golang-dependency-injection-using-wire/
- Dependency Injection in Go using Wire: https://www.mohitkhare.com/blog/go-dependency-injection/
- Wire 依赖注入: https://go-kratos.dev/docs/guide/wire/
- Dependency Injection with Dig: https://www.jetbrains.com/guide/go/tutorials/dependency_injection_part_one/di_with_dig/
- inject Documentation: https://pkg.go.dev/github.com/facebookgo/inject
- Build Constraints: https://pkg.go.dev/go/build#hdr-Build_Constraints
- 控制反转: https://zh.wikipedia.org/wiki/控制反转
- 依赖注入: https://zh.wikipedia.org/wiki/依赖注入
- SOLID (面向对象设计): https://zh.wikipedia.org/wiki/SOLID_(面向对象设计)
- 设计模式之美 —— 19 | 理论五：控制反转、依赖反转、依赖注入，这三者有何区别和联系？: https://time.geekbang.org/column/article/177444
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/wire

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn

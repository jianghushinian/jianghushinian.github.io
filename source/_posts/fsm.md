---
title: '在 Go 中如何使用有限状态机优雅解决程序中状态转换问题'
date: 2025-05-25 21:30:44
tags:
- Go
- FSM
- 有限状态机
categories:
- Go
---

在编程中，有限状态机（FSM）是管理复杂状态流转的优雅工具，其核心在于通过明确定义**状态**、**事件**和**转换规则**，将业务逻辑模块化。本文将探讨在 Go 中如何使用有限状态机。

<!-- more -->

### 有限状态机

在介绍有限状态机之前，我们可以先来看一个示例程序：

> [https://github.com/jianghushinian/blog-go-example/blob/main/fsm/main.go](https://github.com/jianghushinian/blog-go-example/blob/main/fsm/main.go)
>

```go
package main

import (
	"fmt"
)

type State string

const (
	ClosedState State = "closed"
	OpenState   State = "open"
)

type Event string

const (
	OpenEvent  Event = "open"
	CloseEvent Event = "close"
)

type Door struct {
	to    string
	state State
}

func NewDoor(to string) *Door {
	return &Door{
		to:    to,
		state: ClosedState,
	}
}

func (d *Door) CurrentState() State {
	return d.state
}

func (d *Door) HandleEvent(e Event) {
	switch e {
	case OpenEvent:
		d.state = OpenState
	case CloseEvent:
		d.state = ClosedState
	}
}

func main() {
	door := NewDoor("heaven")

	fmt.Println(door.CurrentState())

	door.HandleEvent(OpenEvent)
	fmt.Println(door.CurrentState())

	door.HandleEvent(CloseEvent)
	fmt.Println(door.CurrentState())
}
```

这个示例中，定义了一个核心结构体 `Door`：

```go
type Door struct {
	to    string
	state State
}
```

`Door` 结构体表示这是一扇门，`to` 属性表示这扇门通往哪里，`state` 属性标识这扇门当前处于哪种**状态**。门只有**开**和**关**两种状态，分别对应 `open` 和 `closed`。我们可以执行两个动作（**事件**）**开门**和**关门**，分别对应 `open` 和 `close`。

我们在 `main` 函数中使用 `NewDoor("heaven")` 构造了一个 `door` 对象，然后打印当前门所处的状态。接着调用 `door.HandleEvent(OpenEvent)` 实现开门操作，并打印现在门所处的状态。最后调用 `door.HandleEvent(CloseEvent)` 实现关门操作，并打印最终门所处的状态。

执行示例代码，得到输出如下：

```bash
$ go run main.go
closed
open
closed
```

以上，我们就通过 Go 程序模拟了真实世界中的门。

那么这跟有限状态机有什么关系呢？其实，门就是一种有限状态机的模型。

维基百科中对[有限状态机](https://zh.wikipedia.org/wiki/%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E6%9C%BA)的定义比较晦涩，在这里，我以有限状态机中最核心的三个特征来为你介绍到底什么是有限状态机。

有限状态机（英语：finite-state machine，缩写：FSM）是一个数学计算模型，其特征如下：

+ 状态（state）个数是有限的。
+ 任意一个时刻，只处于其中一种状态。
+ 某种条件下（触发某种 event），会从一种状态转变（transition）为另一种状态。

满足以上三个特征的对象，我们都可以称其为有限状态机。

对于 `Door` 来说，其状态只有两种，分别为 `open` 和 `closed`；任意一个时刻，门只会处在 `open` 或 `closed` 中的一种状态；如果门处于 `closed` 状态，当触发 `open` 事件时，门就会从 `closed` 状态变为 `open` 状态，反之亦然。所以 `Door` 对象就是一个有限状态机。

在我们的日常生活中，有限状态机非常多，比如过马路时的红绿灯，只有三种颜色（状态）红、黄、绿；任意一个时刻，也只会处于一种颜色（状态），其触发条件是倒计时。

程序中也有很多常见的有限状态机，比如电商的订单，有已创建、已支付、已配送、已完成、已取消、已退款等有限的状态枚举；任意一个时刻，只处于其中一种状态；触发条件则是支付、申请退款等操作。

可以发现，有限状态机中最重要的两个概念就是**状态**和**事件**。一个对象存在**有限个状态**，并在某些**事件发生**时可以实现**状态转换**，这是一个非常常见的模型，我们在写程序的过程中，可以将很多对象都抽象成有限状态机。

既然有限状态机的模型比较统一，我们是否可以专门抽象出来一个有限状体机程序，来处理这些有限状态机对象？

[looplab/fsm](https://github.com/looplab/fsm) 包就是干这个事情的，这是一个有限状态机的 Go 语言实现。接下来，我们来一起学习一下这个包的使用。

### 使用示例

#### 安装

可以通过如下命令来安装 `fsm` 包：

```bash
$ go get github.com/looplab/fsm
```

#### 简单使用

我们可以用 `fsm` 包来重写一下前文中介绍的 `Door` 对象实现：

> [https://github.com/jianghushinian/blog-go-example/blob/main/fsm/examples/simple.go](https://github.com/jianghushinian/blog-go-example/blob/main/fsm/examples/simple.go)
>

```go
package main

import (
	"context"
	"fmt"

	"github.com/looplab/fsm"
)

func main() {
	fsm := fsm.NewFSM(
		"closed",
		fsm.Events{
			{Name: "open", Src: []string{"closed"}, Dst: "open"},
			{Name: "close", Src: []string{"open"}, Dst: "closed"},
		},
		fsm.Callbacks{},
	)

	fmt.Println(fsm.Current())

	err := fsm.Event(context.Background(), "open")
	if err != nil {
		fmt.Println(err)
	}

	fmt.Println(fsm.Current())

	err = fsm.Event(context.Background(), "close")
	if err != nil {
		fmt.Println(err)
	}

	fmt.Println(fsm.Current())
}
```

示例中，通过 `fsm.NewFSM` 函数可以构造一个有限状态机对象 `fsm`，构造函数接收 3 个参数，第一个参数表示有限状态机的**当前状态**（或者叫初始状态）；第二个参数是一个 `fsm.Events{}` 对象，它底层类型是一个 `slice`，即可以注册多个事件，比如 `{Name: "open", Src: []string{"closed"}, Dst: "open"}` 表示，当前状态为 `closed` 的情况下，如果触发 `open` 事件，则状态机的状态将转换成 `open`，注意，这里面 `Name` 对应的 `open` 表示事件，`Dst` 对应的 `open` 表示状态；第三个参数是一个回调函数列表 `fsm.Callbacks{}`，暂时设为空。

接下来，我们先用 `fmt.Println(fsm.Current())` 输出 `fsm` 的当前状态；接着，触发 `open` 事件并输出 `fsm` 的最新状态；最后，触发 `close` 事件，并输出 `fsm` 的最终状态。

执行示例代码，得到输出如下：

```bash
$ go run examples/simple.go 
closed
open
closed
```

可以看到，我们使用 `fsm` 包，实现了 `Door` 状态机。

对比之下，我们可以发现，`fsm` 包是有限状态机的高度抽象。在使用 `fsm` 包时，我们无需像在使用 `Door` 时一样，手动编写一个 `*Door.HandleEvent` 方法来处理事件实现状态转换。而是可以直接在构造有限状态机时，通过类似 `{Name: "open", Src: []string{"closed"}, Dst: "open"}` 的方式，来定义事件触发时的状态转换规则。这样，当调用 `fsm.Event(ctx, "open")` 触发事件时，`fsm` 包就会根据预置的规则自动帮我们完成状态转换，将对象从原状态（`Src`）转换成目标状态（`Dst`）。

这样做的好处是，我们将状态转换规则进行了预置，在代码逻辑中，我们只需关注何时该触发某个事件即可，无需手动转换状态。这会大大减少复杂业务代码中出现 Bug 的概率，并且也提升了代码的可维护性。

#### 在结构体中使用

此外，`fsm` 包还有另一个常见用法，它可以作为结构体字段来使用。

示例如下：

> [https://github.com/jianghushinian/blog-go-example/blob/main/fsm/examples/struct/struct.go](https://github.com/jianghushinian/blog-go-example/blob/main/fsm/examples/struct/struct.go)
>

```go
package main

import (
    "context"
    "fmt"

    "github.com/looplab/fsm"
)

type Door struct {
    To  string
    FSM *fsm.FSM
}

func NewDoor(to string) *Door {
    d := &Door{
        To: to,
    }

    d.FSM = fsm.NewFSM(
        "closed",
        fsm.Events{
            {Name: "open", Src: []string{"closed"}, Dst: "open"},
            {Name: "close", Src: []string{"open"}, Dst: "closed"},
        },
        fsm.Callbacks{
            "enter_state": func(_ context.Context, e *fsm.Event) { d.enterState(e) },
        },
    )

    return d
}

func (d *Door) enterState(e *fsm.Event) {
    fmt.Printf("The door to %s is %s\n", d.To, e.Dst)
}

func main() {
    door := NewDoor("heaven")

    err := door.FSM.Event(context.Background(), "open")
    if err != nil {
        fmt.Println(err)
    }

    err = door.FSM.Event(context.Background(), "close")
    if err != nil {
        fmt.Println(err)
    }
}
```

此处，我们使用 `Door` 结构体重新实现了有限状态机，将 `FSM` 对象作为 `Door` 结构体的一个属性，这样，`Door` 结构体看起来更加符合业务。

并且，这里我们还为有限状态机定义了一个回调函数：

```go
fsm.Callbacks{
    "enter_state": func(_ context.Context, e *fsm.Event) { d.enterState(e) },
},
```

`enter_state` 是事件触发后的回调函数，定义了任意一个事件结束后触发的函数，即当触发 `FSM.Event(ctx, event)` 时会调用此函数。

执行示例代码，得到输出如下：

```bash
$ go run examples/struct/struct.go 
The door to heaven is open
The door to heaven is closed
```

可以发现，无论是触发 `open` 事件，还是触发 `close` 事件，`enter_state` 定义的回调函数都会被调用。

事实上，`fsm` 包不止提供了这一个回调函数，它共计为我们提供了 8 个回调函数。

完整回调函数使用示例如下：

> [https://github.com/jianghushinian/blog-go-example/blob/main/fsm/examples/struct/struct.go](https://github.com/jianghushinian/blog-go-example/blob/main/fsm/examples/struct/struct.go)
>

```go
package main

import (
	"context"
	"fmt"

	"github.com/fatih/color"
	"github.com/looplab/fsm"
)

type Door struct {
	To  string
	FSM *fsm.FSM
}

func NewDoor(to string) *Door {
	d := &Door{
		To: to,
	}

	d.FSM = fsm.NewFSM(
		"closed",
		fsm.Events{
			{Name: "open", Src: []string{"closed"}, Dst: "open"},
			{Name: "close", Src: []string{"open"}, Dst: "closed"},
		},
		fsm.Callbacks{
			// NOTE: closed => open
			// 在 open 事件发生之前触发（这里的 open 是指代 open event）
			"before_open": func(_ context.Context, e *fsm.Event) {
				color.Magenta("| before open\t | %s | %s |", e.Src, e.Dst)
			},
			// 任一事件发生之前触发
			"before_event": func(_ context.Context, e *fsm.Event) {
				color.HiMagenta("| before event\t | %s | %s |", e.Src, e.Dst)
			},
			// 在离开 closed 状态时触发
			"leave_closed": func(_ context.Context, e *fsm.Event) {
				color.Cyan("| leave closed\t | %s | %s |", e.Src, e.Dst)
			},
			// 离开任一状态时触发
			"leave_state": func(_ context.Context, e *fsm.Event) {
				color.HiCyan("| leave state\t | %s | %s |", e.Src, e.Dst)
			},
			// 在进入 open 状态时触发（这里的 open 是指代 open state）
			"enter_open": func(_ context.Context, e *fsm.Event) {
				color.Green("| enter open\t | %s | %s |", e.Src, e.Dst)
			},
			// 进入任一状态时触发
			"enter_state": func(_ context.Context, e *fsm.Event) {
				color.HiGreen("| enter state\t | %s | %s |", e.Src, e.Dst)
			},
			// 在 open 事件发生之后触发（这里的 open 是指代 open event）
			"after_open": func(_ context.Context, e *fsm.Event) {
				color.Yellow("| after open\t | %s | %s |", e.Src, e.Dst)
			},
			// 任一事件结束后触发
			"after_event": func(_ context.Context, e *fsm.Event) {
				color.HiYellow("| after event\t | %s | %s |", e.Src, e.Dst)
			},
		},
	)

	return d
}

func main() {
	door := NewDoor("heaven")

	color.White("--------- closed to open ---------")
	color.White("| event\t\t | src\t  | dst\t |")
	color.White("----------------------------------")

	err := door.FSM.Event(context.Background(), "open")
	if err != nil {
		fmt.Println(err)
	}
	color.White("----------------------------------")
}
```

执行示例代码，得到输出如下：

![fsm](fsm.png)

这是我们触发 `open` 事件，将 `Door` 状态机从 `closed` 状态转换成 `open` 状态的完整生命周期回调函数执行记录。

先不要觉得多，记不住，从而有抵触情绪。我忙你依次来分析一下这些回调函数你就理解了。

首先，这些回调函数执行顺序与定义顺序无关，所以以上示例代码无论如何调整回调函数定义顺序，其执行结果仍是一样的。

接着，其实你可以发现，我用不同颜色，区分了每一个回调函数的输出结果。细心观察，你还可以察觉到每两个连续的回调函数的输出颜色是用一个浅色和一个高亮色来区分的。虽然有 8 个回调函数，但其实可以分为 4 类，分别是 `before`、`leave`、`enter` 以及 `after`，所以每两个挨着的同色系输出属于同一类回调函数。

+ `before` 表示在某个**事件**触发**之前**执行的回调函数：
    - `before_open` 表示在 `open` 事件发生之前触发。
    - `before_event` 表示任意一个事件发生之前触发。
    - 如果同时定义了 `before_<EVENT>` 和 `before_event`，则 `before_<EVENT>` 先于 `before_event` 触发。
+ `leave` 表示在**离开**某一**状态**时执行的回调函数：
    - `leave_closed` 表示在离开 `closed` 状态时触发。
    - `leave_state` 表示离开任意一个状态时都会触发。
    - 如果同时定义了 `leave_<OLD_STATE>` 和 `leave_state`，则 `leave_<OLD_STATE>` 先于 `leave_state` 触发。
+ `enter` 表示在**进入**某一**状态**时执行的回调函数：
    - `enter_open` 表示在进入 `open` 状态时触发。
    - `enter_state` 表示进入任意一个状态时都会触发。
    - 如果同时定义了 `enter_<NEW_STATE>` 和 `enter_state`，则 `enter_<NEW_STATE>` 先于 `enter_state` 触发。
+ `after` 表示在某个**事件**触发**之后**执行的回调函数：
    - `after_open` 表示在 `open` 事件发生之后触发。
    - `after_event` 表示任意一个事件发生之后触发。
    - 如果同时定义了 `after_<EVENT>` 和 `after_event`，则 `after_<EVENT>` 先于 `after_event` 触发。

我们通过回调函数**执行时机**，将这 8 个回调函数分为了 4 大类。如果站在**状态**和**事件**的角度，则可以分为两类，有些回调函数是在事件触发时执行的，如 `before_xxx`、`after_xxx`，另外一些回调函数则是在状态发生转换时执行的，如 `leave_xxx`、`enter_xxx`。

这些回调函数，可以在事件触发或状态转换的生命周期内，辅助我们实现一些特有的业务逻辑。

其实，`fsm` 还为我们提供了两种定义回调函数的简写形式，比如：

```go
"closed": func(_ context.Context, e *fsm.Event) {
    color.Green("| enter closed\t | %s | %s |", e.Src, e.Dst)
},
```

等价于：

```go
"enter_closed": func(_ context.Context, e *fsm.Event) {
    color.Green("| enter closed\t | %s | %s |", e.Src, e.Dst)
},
```

即 `<NEW_STATE>` 是 `enter_<NEW_STATE>` 的简写形式。

再比如：

```go
"close": func(_ context.Context, e *fsm.Event) {
    color.Yellow("| after close\t | %s | %s |", e.Src, e.Dst)
},
```

等价于：

```go
"after_close": func(_ context.Context, e *fsm.Event) {
    color.Yellow("| after close\t | %s | %s |", e.Src, e.Dst)
},
```

即 `<EVENT>` 是 `after_<EVENT>` 的简写形式。

如果我们定义一个不存在的事件/状态，`fsm` 表现如何呢？

```go
"unknown": func(_ context.Context, e *fsm.Event) {
    color.Red("unknown event\t | %s | %s |", e.Src, e.Dst)
},
```

这个示例结果就交给你自行去探索了。

#### 项目实战

以上我向你介绍了有限状态机的概念，以及在 Go 中如何利用 `fsm` 包实现有限状态机。如果你看后还觉得不过瘾，想了解一下在真实的企业级项目中，是如何使用有限状态机的，那么你可以参考 OneX 项目 `nightwatch` 组件的源码（[https://github.com/onexstack/onex/tree/feature/onex-v2/internal/nightwatch/watcher/user](https://github.com/onexstack/onex/tree/feature/onex-v2/internal/nightwatch/watcher/user)），来学习如何在项目中落地 `fsm`。

### 总结

本文以一个示例开始，我向你介绍了什么是有限状态机。接着我向你推荐了 Go 中 `fsm` 包，并使用它实现了一个 `Door` 有限状态机。通过对比，我们能够发现，使用 `fsm` 来实现有限状态机好处是，可以将状态转换规则提前预置，然后在代码逻辑中，只需关注何时该触发某个事件即可，无需手动转换状态。我认为这也是 `fsm` 的优势所在，定义好了状态流转规则，状态转换就不会出现未知异常，如果将状态转换的代码写在复杂的业务逻辑中，则很容易出现 Bug，尤其在代码多次迭代过程中，很容易漏掉某些 case。使用 `fsm` 则可以有效避免这些问题。

对于 `fsm` 的更多使用示例，可以参考官方 examples 代码：[https://github.com/looplab/fsm/tree/main/examples](https://github.com/looplab/fsm/tree/main/examples) 。

此外，挖一个坑，如果后续有时间，我将对 `fsm` 源码进行深度剖析与解读，敬请期待！

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/fsm) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ 有限状态机定义：[https://zh.wikipedia.org/wiki/有限状态机](https://zh.wikipedia.org/wiki/有限状态机)
+ JavaScript与有限状态机：[https://www.ruanyifeng.com/blog/2013/09/finite-state_machine_for_javascript.html](https://www.ruanyifeng.com/blog/2013/09/finite-state_machine_for_javascript.html)
+ OneX 有限状态机：[https://github.com/onexstack/onex/tree/feature/onex-v2/internal/nightwatch/watcher/user](https://github.com/onexstack/onex/tree/feature/onex-v2/internal/nightwatch/watcher/user)
+ fsm GitHub 源码：[https://github.com/looplab/fsm](https://github.com/looplab/fsm)
+ fsm Documentation：[https://pkg.go.dev/github.com/looplab/fsm@v1.0.3](https://pkg.go.dev/github.com/looplab/fsm@v1.0.3)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/fsm](https://github.com/jianghushinian/blog-go-example/tree/main/fsm)
+ 本文永久地址：[https://jianghushinian.cn/2025/05/25/fsm/](https://jianghushinian.cn/2025/05/25/fsm/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

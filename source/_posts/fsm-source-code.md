---
title: 'Go 语言 fsm 源码解读，这一次让你彻底学会有限状态机'
date: 2025-06-01 16:48:08
tags:
- Go
- FSM
- 有限状态机
categories:
- Go
---

我在文章《[在 Go 中如何使用有限状态机优雅解决程序中状态转换问题](https://jianghushinian.cn/2025/05/25/fsm/)》中讲解了有限状态机的概念，并介绍了 Go 中有限状态机 [fsm](https://github.com/looplab/fsm) 包的使用。本篇文章，我将更进一步，直接通过解读源码的方式，让你深刻理解 fsm 是如何实现的，这一次你将彻底掌握有限状态机。

<!-- more -->

### 源码解读

废话不多说，我们直接上代码。

#### 结构体

首先 fsm 包定义了一个结构体 `FSM` 用来表示状态机。

> [https://github.com/looplab/fsm/blob/main/fsm.go#L40](https://github.com/looplab/fsm/blob/main/fsm.go#L40)
>

```go
// FSM 是持有「当前状态」的状态机。
type FSM struct {
	// FSM 当前状态
	current string

	// transitions 将「事件和原状态」映射到「目标状态」。
	transitions map[eKey]string

	// callbacks 将「回调类型和目标」映射到「回调函数」。
	callbacks map[cKey]Callback

	// transition 是内部状态转换函数，可以直接使用，也可以在异步状态转换时调用。
	transition func()
	// transitionerObj 用于调用 FSM 的 transition() 函数。
	transitionerObj transitioner

	// stateMu 保护对当前状态的访问。
	stateMu sync.RWMutex
	// eventMu 保护对 Event() 和 Transition() 两个函数的调用。
	eventMu sync.Mutex

	// metadata 可以用来存储和加载可能跨事件使用的数据
	// 使用 SetMetadata() 和 Metadata() 方法来存储和加载数据。
	metadata map[string]interface{}
	// metadataMu 保护对元数据的访问。
	metadataMu sync.RWMutex
}
```

我们知道，有限状态机中最重要的三个特征如下：

+ <font style="color:rgb(33, 33, 33);">状态（state）个数是有限的。</font>
+ <font style="color:rgb(33, 33, 33);">任意一个时刻，只处于其中一种状态。</font>
+ <font style="color:rgb(33, 33, 33);">某种条件下（触发某种 event），会从一种状态转变（transition）为另一种状态。</font>

<font style="color:rgb(33, 33, 33);">所以，</font>`<font style="color:rgb(33, 33, 33);">FSM</font>`<font style="color:rgb(33, 33, 33);"> 结构体中一定包含与这些特征有关的字段。</font>

`current` 表示状态机的当前状态。

`transitions` 用于记录状态转换规则，即定义触发某一事件时，允许从某一种状态，转换成另一种状态。它是一个 `map` 对象，其 `key` 为 `eKey` 类型：

```go
// eKey is a struct key used for storing the transition map.
type eKey struct {
	// event is the name of the event that the keys refers to.
	event string

	// src is the source from where the event can transition.
	src string
}
```

`eKey` 类型用来记录事件和原状态。`map` 的 `value` 为 `string` 类型，用来记录目标状态。

`callbacks` 用于记录事件触发时的回调函数。它也是一个 `map` 对象，其 `key` 为 `cKey` 类型：

```go
// cKey is a struct key used for keeping the callbacks mapped to a target.
type cKey struct {
	// target is either the name of a state or an event depending on which
	// callback type the key refers to. It can also be "" for a non-targeted
	// callback like before_event.
	target string

	// callbackType is the situation when the callback will be run.
	callbackType int
}
```

`cKey` 类型用来记录目标和回调类型，其中目标可以是**状态**或**事件**名称，回调类型可选值如下：

```go
const (
	// 未设置回调
	callbackNone int = iota
	// 事件触发前执行的回调
	callbackBeforeEvent
	// 离开旧状态前执行的回调
	callbackLeaveState
	// 进入新状态是执行的回调
	callbackEnterState
	// 事件完成时执行的回调
	callbackAfterEvent
)
```

回调类型决定了回调函数的执行时机。

map 的 value 为回调函数，其声明类型如下：

```go
// Callback is a function type that callbacks should use. Event is the current
// event info as the callback happens.
type Callback func(context.Context, *Event)
```

还记得回调函数是如何注册的吗？

```go
fsm.Callbacks{
	// 任一事件发生之前触发
	"before_event": func(_ context.Context, e *fsm.Event) {
		color.HiMagenta("| before event\t | %s | %s |", e.Src, e.Dst)
	},
}
```

这里注册的 `before_event` 回调函数签名就是 `Callback` 类型。

当然这里还使用了 `fsm.Callbacks` 类型来注册，想必你已经猜到了 `fsm.Callbacks` 的类型：

```go
// Callbacks is a shorthand for defining the callbacks in NewFSM.
type Callbacks map[string]Callback
```

接下来的 `transition` 和 `transitionerObj` 两个属性是用来实现状态转换的，暂且留到后续使用时再来研究。

这里还有两个互斥锁，分别用来保护对当前状态的访问（`stateMu`），和保证事件触发时的操作并发安全（`eventMu`）。

最后 FSM 还提供了 `metadata` 和 `metadataMu` 两个属性，这俩属性用于管理元数据信息，后文中我会演示其使用场景。

现在，我们可以总结一下 `FSM` 结构体定义：

![FSM](fsm.jpg)

接下来，我将对 `FSM` 结构体所实现的方法进行讲解。

#### 方法

我们先来看一下 `FSM` 结构体都提供了哪些方法和能力：

![FSM](fsm-methods.jpg)

这里列出了 `FSM` 结构体实现的所有方法，并且做了分类，你先有个感官上的认识，接下来我们依次解读。

##### 构造函数

我们最先要分析的源码，当然是 `FSM` 结构体的构造函数了，其实现如下：

```go
func NewFSM(initial string, events []EventDesc, callbacks map[string]Callback) *FSM {
	// 构造有限状态机 FSM
	f := &FSM{
		transitionerObj: &transitionerStruct{},        // 状态转换器，使用默认实现
		current:         initial,                      // 当前状态
		transitions:     make(map[eKey]string),        // 存储「事件和原状态」到「目标状态」的转换规则映射
		callbacks:       make(map[cKey]Callback),      // 回调函数映射表
		metadata:        make(map[string]interface{}), // 元信息
	}

	// 构建 f.transitions map，并且存储所有的「事件」和「状态」集合
	allEvents := make(map[string]bool) // 存储所有事件的集合
	allStates := make(map[string]bool) // 存储所有状态的集合
	for _, e := range events {         // 遍历事件列表，提取并存储所有事件和状态
		for _, src := range e.Src {
			f.transitions[eKey{e.Name, src}] = e.Dst
			allStates[src] = true
			allStates[e.Dst] = true
		}
		allEvents[e.Name] = true
	}

	// 提取「回调函数」到「事件和原状态」的映射关系，并注册到 callbacks
	for name, fn := range callbacks {
		var target string    // 目标：状态/事件
		var callbackType int // 回调类型（决定了调用顺序）

		// 根据回调函数名称前缀分类
		switch {
		// 事件触发前执行
		case strings.HasPrefix(name, "before_"):
			target = strings.TrimPrefix(name, "before_")
			if target == "event" { // 全局事件前置钩子（任何事件触发都会调用，如用于日志记录场景）
				target = "" // 将 target 置空
				callbackType = callbackBeforeEvent
			} else if _, ok := allEvents[target]; ok { // 在特定事件前执行
				callbackType = callbackBeforeEvent
			}
		// 离开当前状态前执行
		case strings.HasPrefix(name, "leave_"):
			target = strings.TrimPrefix(name, "leave_")
			if target == "state" { // 全局状态离开钩子
				target = ""
				callbackType = callbackLeaveState
			} else if _, ok := allStates[target]; ok { // 离开旧状态前执行
				callbackType = callbackLeaveState
			}
		// 进入新状态后执行
		case strings.HasPrefix(name, "enter_"):
			target = strings.TrimPrefix(name, "enter_")
			if target == "state" { // 全局状态进入钩子
				target = ""
				callbackType = callbackEnterState
			} else if _, ok := allStates[target]; ok { // 进入新状态后执行
				callbackType = callbackEnterState
			}
		// 事件完成后执行
		case strings.HasPrefix(name, "after_"):
			target = strings.TrimPrefix(name, "after_")
			if target == "event" { // 全局事件后置钩子
				target = ""
				callbackType = callbackAfterEvent
			} else if _, ok := allEvents[target]; ok { // 事件完成后执行
				callbackType = callbackAfterEvent
			}
		// 处理未加前缀的回调（简短版本）
		default:
			target = name                       // 状态/事件
			if _, ok := allStates[target]; ok { // 如果 target 为某个状态，则 callbackType 会置为与 enter_[target] 相同
				callbackType = callbackEnterState
			} else if _, ok := allEvents[target]; ok { // 如果 target 为某个事件，则 callbackType 会置为与 after_[target] 相同
				callbackType = callbackAfterEvent
			}
		}

		// 记录 callbacks map
		if callbackType != callbackNone {
			// key: callbackType（用于决定执行顺序） + target（如果是全局钩子，则 target 为空，否则，target 为状态/事件）
			// val: 事件触发时需要执行的回调函数
			f.callbacks[cKey{target, callbackType}] = fn
		}
	}

	return f
}
```

构造函数内部代码比较多，我们可以将它的核心逻辑分为 3 块，分别是：构造有限状态机 `FSM`、记录事件（event）和状态（state）、注册回调函数。

构造有限状态机 `FSM` 部分的代码比较简单：

```go
// 构造有限状态机 FSM
f := &FSM{
	transitionerObj: &transitionerStruct{},        // 状态转换器，使用默认实现
	current:         initial,                      // 当前状态
	transitions:     make(map[eKey]string),        // 存储「事件和原状态」到「目标状态」的转换规则映射
	callbacks:       make(map[cKey]Callback),      // 回调函数映射表
	metadata:        make(map[string]interface{}), // 元信息
}
```

使用函数参数 `initial` 作为状态机的当前状态，几个 `map` 类型的属性，都赋予了默认值。

接下来的部分代码逻辑用于记录事件（event）和状态（state）：

```go
// 构建 f.transitions map，并且存储所有的「事件」和「状态」集合
allEvents := make(map[string]bool) // 存储所有事件的集合
allStates := make(map[string]bool) // 存储所有状态的集合
for _, e := range events {         // 遍历事件列表，提取并存储所有事件和状态
	for _, src := range e.Src {
		f.transitions[eKey{e.Name, src}] = e.Dst
		allStates[src] = true
		allStates[e.Dst] = true
	}
	allEvents[e.Name] = true
}
```

这里 `allEvents` 和 `allStates` 都是集合类型（Set），分别用于记录所有注册的事件和状态。

最后这一部分代码用来注册回调函数：

```go
for name, fn := range callbacks {
	var target string    // 目标：状态/事件
	var callbackType int // 回调类型（决定了调用顺序）

	// 根据回调函数名称前缀分类
	switch {
	// 事件触发前执行
	case strings.HasPrefix(name, "before_"):
		target = strings.TrimPrefix(name, "before_")
		if target == "event" { // 全局事件前置钩子（任何事件触发都会调用，如用于日志记录场景）
			target = "" // 将 target 置空
			callbackType = callbackBeforeEvent
		} else if _, ok := allEvents[target]; ok { // 在特定事件前执行
			callbackType = callbackBeforeEvent
		}
	...
	}

	// 记录 callbacks map
	if callbackType != callbackNone {
		// key: callbackType（用于决定执行顺序） + target（如果是全局钩子，则 target 为空，否则，target 为状态/事件）
		// val: 事件触发时需要执行的回调函数
		f.callbacks[cKey{target, callbackType}] = fn
	}
}
```

这里遍历了 `callbacks` 列表，并根据回调函数名称前缀分类，然后注册到 `f.callbacks` 属性的 `map` 对象中。

> NOTE:
>
> 代码注释中的“钩子”就代表回调函数，只不过是另一种叫法罢了。
>

我们再来回顾一下回调函数是如何注册的：

```go
fsm.Callbacks{
    "before_event": func(_ context.Context, e *fsm.Event) { ... },
}
```

这个参数被传入构造函数后，会进入 `strings.HasPrefix(name, "before_")` 这个 case，然后 `if target == "event"` 成立，此时 `target` 将会被置空，回调类型 `callbackType` 将被赋值为 `callbackBeforeEvent`。如果我们注册的是 `before_closed` 回调函数，则 `target` 值为 `closed`。对于 `target` 不同处理，**将决定最后回调函数的执行顺序**。我们暂且不继续深入，留个悬念，后续解读回调函数相关的源码，你就能白为什么了。

不过，我还要特别强调一下 `default` 分支的 case：

```go
default:
	target = name                       // 状态/事件
	if _, ok := allStates[target]; ok { // 如果 target 为某个状态，则 callbackType 会置为与 enter_[target] 相同，即二者等价
		callbackType = callbackEnterState
	} else if _, ok := allEvents[target]; ok { // 如果 target 为某个事件，则 callbackType 会置为与 after_[target] 相同，即二者等价
		callbackType = callbackAfterEvent
	}
}
```

还记得在上一篇文章中我提到过，注册 `closed` 事件等价于 `enter_closed` 事件吗？就是在 `default` 这个 case 中实现的。

![FSM Event](fsm-event1.jpg)

对于构造函数的讲解就到这里，里面一些具体的代码细节你可能现在有点发懵，没关系，接着往下看，你的疑惑都将被解开。

##### 当前状态

接着，我们来看一下与当前状态相关的这几个方法源码是如何实现的，它们的代码其实都很简单，我就不一一解读了，我把源码贴在这里，你一看就能明白：

```go
// Current 返回 FSM 的当前状态。
func (f *FSM) Current() string {
	f.stateMu.RLock()
	defer f.stateMu.RUnlock()
	return f.current
}

// Is 判断 FSM 当前状态是否为指定状态。
func (f *FSM) Is(state string) bool {
	f.stateMu.RLock()
	defer f.stateMu.RUnlock()
	return state == f.current
}

// SetState 将 FSM 从当前状态转移到指定状态。
// 此调用不触发任何回调函数（如果定义）。
func (f *FSM) SetState(state string) {
	f.stateMu.Lock()
	defer f.stateMu.Unlock()
	f.current = state
}

// Can 判断 FSM 在当前状态下，是否可以触发指定事件，如果可以，则返回 true。
func (f *FSM) Can(event string) bool {
	f.eventMu.Lock()
	defer f.eventMu.Unlock()
	f.stateMu.RLock()
	defer f.stateMu.RUnlock()
	_, ok := f.transitions[eKey{event, f.current}]
	return ok && (f.transition == nil)
}

func (f *FSM) Cannot(event string) bool {
	return !f.Can(event)
}

// AvailableTransitions 返回当前状态下可用的转换列表。
func (f *FSM) AvailableTransitions() []string {
	f.stateMu.RLock()
	defer f.stateMu.RUnlock()
	var transitions []string
	for key := range f.transitions {
		if key.src == f.current {
			transitions = append(transitions, key.event)
		}
	}
	return transitions
}
```

##### 状态转换

与状态转换相关的方法可以说是 `FSM` 最重要的方法了。

我们先来看 `Event` 方法的实现：

```go
// Event 通过指定事件名称触发状态转换
func (f *FSM) Event(ctx context.Context, event string, args ...interface{}) error {
	f.eventMu.Lock() // 事件互斥锁锁定

	// 为了始终解锁事件互斥锁（eventMu），此处添加了 defer 防止状态转换完成后执行 enter/after 回调时仍持有锁；
	// 因为这些回调可能触发新的状态转换，故在下方代码中需要显式解锁
	var unlocked bool // 标记是否已经解锁
	defer func() {
		if !unlocked { // 如果下方的逻辑已经显式操作过解锁，defer 中无需重复解锁
			f.eventMu.Unlock()
		}
	}()

	f.stateMu.RLock() // 获取状态读锁
	defer f.stateMu.RUnlock()

	// NOTE: 之前的转换尚未完成
	if f.transition != nil {
		// 上一次状态转换还未完成，返回"前一个转换未完成"错误
		return InTransitionError{event}
	}

	// NOTE: 事件 event 在当前状态 current 下是否适用，即是否在 transitions 表中
	dst, ok := f.transitions[eKey{event, f.current}]
	if !ok { // 无效事件
		for ekey := range f.transitions {
			if ekey.event == event {
				// 事件和当前状态不对应
				return InvalidEventError{event, f.current}
			}
		}
		// 未定义的事件
		return UnknownEventError{event}
	}

	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	// 构造一个事件对象
	e := &Event{f, event, f.current, dst, nil, args, false, false, cancel}

	// NOTE: 执行 before 钩子
	err := f.beforeEventCallbacks(ctx, e)
	if err != nil {
		return err
	}

	// NOTE: 当前状态等于目标状态，无需转换
	if f.current == dst {
		f.stateMu.RUnlock()
		defer f.stateMu.RLock()
		f.eventMu.Unlock()
		unlocked = true
		// NOTE: 执行 after 钩子
		f.afterEventCallbacks(ctx, e)
		return NoTransitionError{e.Err}
	}

	// 定义状态转换闭包函数
	transitionFunc := func(ctx context.Context, async bool) func() {
		return func() {
			if ctx.Err() != nil {
				if e.Err == nil {
					e.Err = ctx.Err()
				}
				return
			}

			f.stateMu.Lock()
			f.current = dst    // 状态转换
			f.transition = nil // NOTE: 标记状态转换完成
			f.stateMu.Unlock()

			// 显式解锁 eventMu 事件互斥锁，允许 enterStateCallbacks 回调函数触发新的状态转换操作（避免死锁）
			// 对于异步状态转换，无需显式解锁，锁已在触发异步操作时释放
			if !async {
				f.eventMu.Unlock()
				unlocked = true
			}
			// NOTE: 执行 enter 钩子
			f.enterStateCallbacks(ctx, e)
			// NOTE: 执行 after 钩子
			f.afterEventCallbacks(ctx, e)
		}
	}

	// 记录状态转换函数（这里标记为同步转换）
	f.transition = transitionFunc(ctx, false)

	// NOTE: 执行 leave 钩子
	if err = f.leaveStateCallbacks(ctx, e); err != nil {
		if _, ok := err.(CanceledError); ok {
			f.transition = nil // NOTE: 如果通过 ctx 取消了，则标记为 nil，无需转换
		} else if asyncError, ok := err.(AsyncError); ok { // NOTE: 如果是 AsyncError，说明是异步转换
			// 为异步操作创建独立上下文，以便异步状态转换正常工作
			// 这个新的 ctx 实际上已经脱离了原始 ctx，原 ctx 取消不会影响当前 ctx
			// 不过新的 ctx 保留了原始 ctx 的值，所有通过 ctx 传递的值还可以继续使用
			ctx, cancel := uncancelContext(ctx)
			e.cancelFunc = cancel                    // 绑定新取消函数
			asyncError.Ctx = ctx                     // 传递新上下文
			asyncError.CancelTransition = cancel     // 暴露取消接口
			f.transition = transitionFunc(ctx, true) // NOTE: 标记为异步转换状态
			// NOTE: 如果是异步转换，直接返回，不会同步调用 f.doTransition()，需要用户手动调用 f.Transition() 来触发状态转换
			return asyncError
		}
		return err
	}

	// Perform the rest of the transition, if not asynchronous.
	f.stateMu.RUnlock()
	defer f.stateMu.RLock()
	err = f.doTransition() // NOTE: 执行状态转换逻辑，即调用 f.transition()
	if err != nil {
		return InternalError{}
	}

	return e.Err
}
```

因为 `Event` 是核心方法，所以源码会比较多，我们一起来梳理下核心逻辑。

首先，`Event` 方法会判断上一次的状态转换是否完成：

```go
// NOTE: 之前的转换尚未完成
if f.transition != nil {
	// 上一次状态转换还未完成，返回"前一个转换未完成"错误
	return InTransitionError{event}
}
```

是否转换完成的标志是 `f.transition` 是否为 `nil`，如果上一次状态转换尚未完成，则返回一个 Sentinel Error。

接着，需要判断当前触发的事件是否有效：

```go
// NOTE: 事件 event 在当前状态 current 下是否适用，即是否在 transitions 表中
dst, ok := f.transitions[eKey{event, f.current}]
if !ok { // 无效事件
	for ekey := range f.transitions {
		if ekey.event == event {
			// 事件和当前状态不对应
			return InvalidEventError{event, f.current}
		}
	}
	// 未定义的事件
	return UnknownEventError{event}
}
```

前文中我们说过 `f.transitions` 用于记录状态转换规则，即定义触发某一事件时，允许从某一种状态，转换成另一种状态。

如果在 `f.transitions` 表中查不到任何一条与当前状态和事件对应的数据，则表示无效事件，同样会返回指定的 Sentinel Error。

这些检查都通过后，就会构造一个事件对象：

```go
ctx, cancel := context.WithCancel(ctx)
defer cancel()
// 构造一个事件对象
e := &Event{f, event, f.current, dst, nil, args, false, false, cancel}
```

接下来，就到了状态转换的核心逻辑了。而所有的回调函数，也是在这个时候开始触发执行的。

在执行状态转换之前，首先要执行的就是 `before` 类回调函数：

```go
// NOTE: 执行 before 钩子
err := f.beforeEventCallbacks(ctx, e)
if err != nil {
    return err
}
```

执行完 `before` 类回调函数，会再对状态做一次检查：

```go
// NOTE: 当前状态等于目标状态，无需转换
if f.current == dst {
	f.stateMu.RUnlock()
	defer f.stateMu.RLock()
	f.eventMu.Unlock()
	unlocked = true
	// NOTE: 执行 after 钩子
	f.afterEventCallbacks(ctx, e)
	return NoTransitionError{e.Err}
}
```

如果状态机的当前状态等于目标状态，则无需状态转换，那么直接执行 `after` 类回调函数就行了，最终返回指定的 Sentinel Error。

否则，需要进行状态转换。此时，状态转换也不会直接进行，而是会定义一个状态转换闭包函数并赋值给 `f.transition`：

```go
	// 定义状态转换闭包函数
	transitionFunc := func(ctx context.Context, async bool) func() {
		return func() {
			...
		}
	}

	// 记录状态转换函数（这里标记为同步转换）
	f.transition = transitionFunc(ctx, false)
```

状态转换函数第二个参数用来标记同步转换还是异步转换，这里标记为同步转换。对于异步转换逻辑，我们后面再来讲解。

接下来会先执行 `leave` 类的回调函数：

```go
// NOTE: 执行 leave 钩子
if err = f.leaveStateCallbacks(ctx, e); err != nil {
    ...
}
```

这是调用的第二个回调函数。

最后，终于到了执行状态转换的逻辑了：

```go
err = f.doTransition() // NOTE: 执行状态转换逻辑，即调用 f.transition()
if err != nil {
    return InternalError{}
}
```

这里调用了 `f.doTransition()` 函数，其定义如下：

```go
// doTransition wraps transitioner.transition.
func (f *FSM) doTransition() error {
	return f.transitionerObj.transition(f)
}
```

可以发现，其内部正式调用了 `f.transitionerObj` 属性的 `transition` 方法。

还记得 `f.transitionerObj` 属性是何时赋值吗？在 `NewFSM` 构造函数中，其赋值如下：

```go
// 构造有限状态机 FSM
f := &FSM{
    transitionerObj: &transitionerStruct{},        // 状态转换器，使用默认实现
    current:         initial,                      // 当前状态
    transitions:     make(map[eKey]string),        // 存储「事件和原状态」到「目标状态」的转换规则映射
    callbacks:       make(map[cKey]Callback),      // 回调函数映射表
    metadata:        make(map[string]interface{}), // 元信息
}
```

所以我们需要看一下 `transitionerStruct` 的具体实现：

```go
// transitioner 是 FSM 的状态转换函数接口。
type transitioner interface {
	transition(*FSM) error
}

// 状态转换接口的默认实现
type transitionerStruct struct{}

// Transition completes an asynchronous state change.
//
// The callback for leave_<STATE> must previously have called Async on its
// event to have initiated an asynchronous state transition.
func (t transitionerStruct) transition(f *FSM) error {
	if f.transition == nil {
		return NotInTransitionError{}
	}
	f.transition()
	return nil
}
```

`f.transitionerObj` 属性声明的是 `transitioner` 接口类型，而 `transitionerStruct` 结构体则是这个接口的默认实现。

`transitionerStruct.transition` 方法内部最终还是在调用 `f.transition()` 方法。

而 `f.transition` 方法，也就是前文中定义的那个闭包函数：

```go
// 定义状态转换闭包函数
transitionFunc := func(ctx context.Context, async bool) func() {
    return func() {
        if ctx.Err() != nil {
            if e.Err == nil {
                e.Err = ctx.Err()
            }
            return
        }

        f.stateMu.Lock()
        f.current = dst    // 状态转换
        f.transition = nil // NOTE: 标记状态转换完成
        f.stateMu.Unlock()

        // 显式解锁 eventMu 事件互斥锁，允许 enterStateCallbacks 回调函数触发新的状态转换操作（避免死锁）
        // 对于异步状态转换，无需显式解锁，锁已在触发异步操作时释放
        if !async {
            f.eventMu.Unlock()
            unlocked = true
        }
        // NOTE: 执行 enter 钩子
        f.enterStateCallbacks(ctx, e)
        // NOTE: 执行 after 钩子
        f.afterEventCallbacks(ctx, e)
    }
}

// 记录状态转换函数（这里标记为同步转换）
f.transition = transitionFunc(ctx, false)
```

闭包函数的 `async` 参数用来标记同步或异步，我们暂且不关心异步，这里只关注同步逻辑。

其实，这里的核心逻辑就是完成状态转换：

```go
f.current = dst    // 状态转换
f.transition = nil // NOTE: 标记状态转换完成
```

状态转换完成后，将 `f.transition` 标记为 `nil`。所以根据这个属性的值，就能判断上一次状态转换是否完成。

状态转换完成后，依次执行 `enter` 和 `after` 类回调函数：

```go
// NOTE: 执行 enter 钩子
f.enterStateCallbacks(ctx, e)
// NOTE: 执行 after 钩子
f.afterEventCallbacks(ctx, e)
```

根据 `Event` 方法的源码走读，我们可以总结出状态转换的核心流程如下：

![FSM Event](fsm-event2.jpg)

本小节最后再贴一下 `Transition` 方法的源码：

```go
// Transition wraps transitioner.transition.
func (f *FSM) Transition() error {
	f.eventMu.Lock()
	defer f.eventMu.Unlock()
	return f.doTransition()
}
```

##### 回调函数

现在，我们来看一下回调函数的具体实现：

```go
// beforeEventCallbacks calls the before_ callbacks, first the named then the
// general version.
func (f *FSM) beforeEventCallbacks(ctx context.Context, e *Event) error {
	if fn, ok := f.callbacks[cKey{e.Event, callbackBeforeEvent}]; ok {
		fn(ctx, e)
		if e.canceled {
			return CanceledError{e.Err}
		}
	}
	if fn, ok := f.callbacks[cKey{"", callbackBeforeEvent}]; ok {
		fn(ctx, e)
		if e.canceled {
			return CanceledError{e.Err}
		}
	}
	return nil
}

// leaveStateCallbacks calls the leave_ callbacks, first the named then the
// general version.
func (f *FSM) leaveStateCallbacks(ctx context.Context, e *Event) error {
	if fn, ok := f.callbacks[cKey{f.current, callbackLeaveState}]; ok {
		fn(ctx, e)
		if e.canceled {
			return CanceledError{e.Err}
		} else if e.async { // NOTE: 异步信号
			return AsyncError{Err: e.Err}
		}
	}
	if fn, ok := f.callbacks[cKey{"", callbackLeaveState}]; ok {
		fn(ctx, e)
		if e.canceled {
			return CanceledError{e.Err}
		} else if e.async {
			return AsyncError{Err: e.Err}
		}
	}
	return nil
}

// enterStateCallbacks calls the enter_ callbacks, first the named then the
// general version.
func (f *FSM) enterStateCallbacks(ctx context.Context, e *Event) {
	if fn, ok := f.callbacks[cKey{f.current, callbackEnterState}]; ok {
		fn(ctx, e)
	}
	if fn, ok := f.callbacks[cKey{"", callbackEnterState}]; ok {
		fn(ctx, e)
	}
}

// afterEventCallbacks calls the after_ callbacks, first the named then the
// general version.
func (f *FSM) afterEventCallbacks(ctx context.Context, e *Event) {
	if fn, ok := f.callbacks[cKey{e.Event, callbackAfterEvent}]; ok {
		fn(ctx, e)
	}
	if fn, ok := f.callbacks[cKey{"", callbackAfterEvent}]; ok {
		fn(ctx, e)
	}
}
```

细心观察，你会发现这几个回调函数逻辑其实套路一样，都是先匹配 `cKey` 的 `target` 值为 `e.Event` 回调函数来执行，然后再匹配 `target` 值为 `""` 的回调函数来执行。

还记得 `target` 何时才会为空吗？我们一起回顾下 `NewFSM` 中的代码段：

```go
// 根据回调函数名称前缀分类
switch {
// 事件触发前执行
case strings.HasPrefix(name, "before_"):
    target = strings.TrimPrefix(name, "before_")
    if target == "event" { // 全局事件前置钩子（任何事件触发都会调用，如用于日志记录场景）
        target = "" // 将 target 置空
        callbackType = callbackBeforeEvent
    } else if _, ok := allEvents[target]; ok { // 在特定事件前执行
        callbackType = callbackBeforeEvent
    }
// 离开当前状态前执行
case strings.HasPrefix(name, "leave_"):
    target = strings.TrimPrefix(name, "leave_")
    if target == "state" { // 全局状态离开钩子
        target = ""
        callbackType = callbackLeaveState
    } else if _, ok := allStates[target]; ok { // 离开旧状态前执行
        callbackType = callbackLeaveState
    }
```

当 `target` 的值为 `event/state` 是，就会标记为 `""`。

所以，我们可以得出结论：`xxx_event` 或 `xxx_state` 回调函数，会晚于 `xxx_<EVENT>` 或 `xxx_<STATE>` 而执行。

那么，至此我们就理清了状态转换时所有的回调函数执行顺序：

![FSM Event](fsm-event3.jpg)

而这一结论，与我们在上一篇文章中讲解的示例程序执行输出结果保持一致：

![FSM Event](fsm-event4.jpg)

此外，不知道你有没有发现，其实我在上一篇文章中挖了一个坑没有详细讲解。

在前一篇文章中，我们定义了如下状态转换规则：

```go
fsm.Events{
    {Name: "open", Src: []string{"closed"}, Dst: "open"},
    {Name: "close", Src: []string{"open"}, Dst: "closed"},
},
```

细心的你可能已经发现，其实第一条规则中，事件和目标状态，都叫 `open`；而第二条规则中，事件叫 `close`，目标状态叫 `closed`。

那么你有没有思考过，当事件和目标状态同名时，即在这里 `open` 既是 `event` 又是 `state`，那么定义如下回调函数，这个回调函数是属于 `event` 还是 `state` 呢？

```go
"open": func(_ context.Context, e *fsm.Event) {
    color.Green("| enter open\t | %s | %s |", e.Src, e.Dst)
},
```

我们知道，`<NEW_STATE>` 是 `enter_<NEW_STATE>` 的简写形式，而 `<EVENT>` 又是 `after_<EVENT>` 的简写形式。

我们还知道，这段逻辑是在 `NewFSM` 中的 `default` case 代码中实现的：

```go
// 处理未加前缀的回调（简短版本）
default:
    target = name                       // 状态/事件
    if _, ok := allStates[target]; ok { // 如果 target 为某个状态，则 callbackType 会置为与 enter_[target] 相同
        callbackType = callbackEnterState
    } else if _, ok := allEvents[target]; ok { // 如果 target 为某个事件，则 callbackType 会置为与 after_[target] 相同
        callbackType = callbackAfterEvent
    }
```

而这段代码中，优先使用 `allStates[target]` 来匹配 `target`，即 `open` 会优先当作 `state` 来处理。

至此，关于回调函数的全部逻辑才算梳理完成。

##### 元信息

`FSM` 对于元信息的操作非常简单，所有涉及元信息操作的方法源码如下：

```go
// Metadata 返回存储在元信息中的值
func (f *FSM) Metadata(key string) (interface{}, bool) {
	f.metadataMu.RLock()
	defer f.metadataMu.RUnlock()
	dataElement, ok := f.metadata[key]
	return dataElement, ok
}

// SetMetadata 存储 key、val 到元信息中
func (f *FSM) SetMetadata(key string, dataValue interface{}) {
	f.metadataMu.Lock()
	defer f.metadataMu.Unlock()
	f.metadata[key] = dataValue
}

// DeleteMetadata 从元信息中删除指定 key 对应的数据
func (f *FSM) DeleteMetadata(key string) {
	f.metadataMu.Lock()
	delete(f.metadata, key)
	f.metadataMu.Unlock()
}
```

至于元信息有什么用，我将用一个示例进行讲解。

### 使用示例

对于 `FSM` 的元信息和异步状态转换操作，仅通过阅读源码，可能无法体会其使用场景。本小节将分别使用两个示例对其进行演示，以此来加深你的理解。

#### 元信息使用

对于有限状态机中元信息的使用，我写了一个使用示例：

> [https://github.com/jianghushinian/blog-go-example/blob/main/fsm/examples/data/data.go](https://github.com/jianghushinian/blog-go-example/blob/main/fsm/examples/data/data.go)
>

```go
package main

import (
	"context"
	"fmt"

	"github.com/looplab/fsm"
)

// NOTE: 将 FSM 作为生产者消费者使用

func main() {
	fsm := fsm.NewFSM(
		"idle",
		fsm.Events{
			// 生产者
			{Name: "produce", Src: []string{"idle"}, Dst: "idle"},
			// 消费者
			{Name: "consume", Src: []string{"idle"}, Dst: "idle"},
			// 清理数据
			{Name: "remove", Src: []string{"idle"}, Dst: "idle"},
		},
		fsm.Callbacks{
			// 生产者
			"produce": func(_ context.Context, e *fsm.Event) {
				dataValue := "江湖十年"
				e.FSM.SetMetadata("message", dataValue)
				fmt.Printf("produced data: %s\n", dataValue)
			},
			// 消费者
			"consume": func(_ context.Context, e *fsm.Event) {
				data, ok := e.FSM.Metadata("message")
				if ok {
					fmt.Printf("consume data: %s\n", data)
				}
			},
			// 清理数据
			"remove": func(_ context.Context, e *fsm.Event) {
				e.FSM.DeleteMetadata("message")
				if _, ok := e.FSM.Metadata("message"); !ok {
					fmt.Println("removed data")
				}
			},
		},
	)

	fmt.Printf("current state: %s\n", fsm.Current())

	err := fsm.Event(context.Background(), "produce")
	if err != nil {
		fmt.Printf("produce err: %s\n", err)
	}

	fmt.Printf("current state: %s\n", fsm.Current())

	err = fsm.Event(context.Background(), "consume")
	if err != nil {
		fmt.Printf("consume err: %s\n", err)
	}

	fmt.Printf("current state: %s\n", fsm.Current())

	err = fsm.Event(context.Background(), "remove")
	if err != nil {
		fmt.Printf("remove err: %s\n", err)
	}

	fmt.Printf("current state: %s\n", fsm.Current())
}
```

在这个示例中，将 `FSM` 作为了生产者消费者来使用。而数据的传递，正是通过元信息（`FSM.metadata`）来实现的。 

+ `FSM.SetMetadata` 用于设置元信息。
+ `FSM.Metadata` 用于获取元信息。
+ `FSM.DeleteMetadata` 则用于清理元信息。

执行示例代码，得到输出如下：

```bash
$ go run examples/data/data.go
current state: idle
produced data: 江湖十年
produce err: no transition
current state: idle
consume data: 江湖十年
consume err: no transition
current state: idle
removed data
remove err: no transition
current state: idle
```

可以发现，在数据的传递过程中，我们得到了 `no transition` 错误，而这个错误其实我们之前有解读过，是在 `Event` 方法如下代码段中产生的：

```go
// NOTE: 当前状态等于目标状态，无需转换
if f.current == dst {
    f.stateMu.RUnlock()
    defer f.stateMu.RLock()
    f.eventMu.Unlock()
    unlocked = true
    // NOTE: 执行 after 钩子
    f.afterEventCallbacks(ctx, e)
    return NoTransitionError{e.Err}
}
```

因为 `FSM` 的状态始终是 `idle`，尚未发生状态转换，所以会返回 `NoTransitionError` 这个 Sentinel Error。

所以，我们只需要忽略这个 `NoTransitionError`，那么就能把状态机 `FSM` 当作生产者消费者来使用。

当然要实现生产者消费者功能我们有很多其他的选择，这个示例主要是作为演示，让我们能够清晰的知道 `FSM` 提供的元信息功能如何使用。

#### 异步示例

在 `FSM` 源码解读的过程中，我有意避而不谈异步状态转换。是因为没有示例的讲解，直接阅读源码，不太容易理解。

我在这里为你演示一个示例，让你来体会一下异步状态转换的用法：

> 
> https://github.com/jianghushinian/blog-go-example/blob/main/fsm/examples/async/async_transition.go

```go
package main

import (
	"context"
	"errors"
	"fmt"

	"github.com/looplab/fsm"
)

// NOTE: 异步状态转换

func main() {
	// 构造有限状态机
	f := fsm.NewFSM(
		"start",
		fsm.Events{
			{Name: "run", Src: []string{"start"}, Dst: "end"},
		},
		fsm.Callbacks{
			// 注册 leave_<OLD_STATE> 回调函数
			"leave_start": func(_ context.Context, e *fsm.Event) {
				e.Async() // NOTE: 标记为异步，触发事件时不进行状态转换
			},
		},
	)

	// NOTE: 触发 run 事件，但不会完整状态转换
	err := f.Event(context.Background(), "run")

	// NOTE: Sentinel Error `fsm.AsyncError` 标识异步状态转换
	var asyncError fsm.AsyncError
	ok := errors.As(err, &asyncError)
	if !ok {
		panic(fmt.Sprintf("expected error to be 'AsyncError', got %v", err))
	}

	// NOTE: 主动执行状态转换操作
	if err = f.Transition(); err != nil {
		panic(fmt.Sprintf("Error encountered when transitioning: %v", err))
	}

	// NOTE: 当前状态
	fmt.Printf("current state: %s\n", f.Current())
}
```

示例中，在构造有限状态机对象 `f` 时，为其注册了 `leave_start` 回调函数，这个回调函数是异步状态转换的关键所在。其内部通过 `e.Async()` 将事件标记为异步，这样在事件触发时，就不会执行状态转换逻辑。

接着，代码中触发 `run` 事件。不过由于 `e.Async()` 的操作，事件触发时不会进行状态转换，而是返回 Sentinel Error `fsm.AsyncError`，这个错误用于标识这是一个异步操作，尚未进行状态转换。

接下来，我们主动调用 `f.Transition()` 来执行状态转换操作。

最终，打印 `FSM` 当前状态。

执行示例代码，得到输出如下：

```bash
$ go run examples/async/async_transition.go 
current state: end
```

这个玩法，将触发事件和状态转换操作进行了分离，使得我们可以主动控制状态转换的时机。

这个示例的关键步骤是在 `leave_start` 回调函数中的 `e.Async()` 逻辑，将当前事件标记为了异步。

首先，`Event` 对象其实也是一个结构体，它有一个属性 `async`，`e.Async()` 逻辑如下：

```go
func (e *Event) Async() {
	e.async = true
}
```

而 `leave_start` 回调函数，是在调用 `*FSM.Event` 方法时触发的：

```go
// NOTE: 执行 leave 钩子
if err = f.leaveStateCallbacks(ctx, e); err != nil {
    if _, ok := err.(CanceledError); ok {
        f.transition = nil // NOTE: 如果通过 ctx 取消了，则标记为 nil，无需转换
    } else if asyncError, ok := err.(AsyncError); ok { // NOTE: 如果是 AsyncError，说明是异步转换
        // 为异步操作创建独立上下文，以便异步状态转换正常工作
        // 这个新的 ctx 实际上已经脱离了原始 ctx，原 ctx 取消不会影响当前 ctx
        // 不过新的 ctx 保留了原始 ctx 的值，所有通过 ctx 传递的值还可以继续使用
        ctx, cancel := uncancelContext(ctx)
        e.cancelFunc = cancel                    // 绑定新取消函数
        asyncError.Ctx = ctx                     // 传递新上下文
        asyncError.CancelTransition = cancel     // 暴露取消接口
        f.transition = transitionFunc(ctx, true) // NOTE: 标记为异步转换状态
        // NOTE: 如果是异步转换，直接返回，不会同步调用 f.doTransition()，需要用户手动调用 f.Transition() 来触发状态转换
        return asyncError
    }
    return err
}
```

`f.leaveStateCallbacks` 就是在执行 `leave_start` 回调函数，其实现如下：

```go
func (f *FSM) leaveStateCallbacks(ctx context.Context, e *Event) error {
	if fn, ok := f.callbacks[cKey{f.current, callbackLeaveState}]; ok {
		fn(ctx, e)
		if e.canceled {
			return CanceledError{e.Err}
		} else if e.async { // NOTE: 异步信号
			return AsyncError{Err: e.Err}
		}
	}
	...
	return nil
}
```

这里最关键的一步就是在 `else if e.async` 时，返回 Sentinel Error `AsyncError`。

而对 `f.leaveStateCallbacks(ctx, e)` 的调用一旦返回 `AsyncError`，就说明是要进入异步状态转换逻辑。

此时会为 `f.transition` 重新赋值，并标记为异步状态转换：

```go
f.transition = transitionFunc(ctx, true) // NOTE: 标记为异步转换状态
// NOTE: 如果是异步转换，直接返回，不会同步调用 f.doTransition()，需要用户手动调用 f.Transition() 来触发状态转换
return asyncError
```

并且返回 `asyncError`，这次 `Event` 事件触发就完成了。不过并没有接着去执行 `f.transition()` 逻辑。所以就实现了异步操作。

到这里，异步转换状态的逻辑，我就帮你梳理完成了。这块可能不太好理解，但是你跟着我的思路，执行一遍示例代码，然后深入到源码，按照流程再梳理一遍，相信就就一定能理解了。

### 总结

上一篇文章我们一起学习了如何利用有限状态机 `FSM` 实现程序中的状态转换。

本篇文章我带你完整阅读了有限状态机的核心源码，为你理清了 `FSM` 的设计思路和它提供的能力。让你能够知其然，也能知其所以然。

并且我还针对不太常用的元信息操作和异步状态转换，提供了使用示例。其实官方 [examples](https://github.com/looplab/fsm/tree/main/examples) 中提供了好几个示例，你可以自行看一下，学完了本文源码，再去看示例就是小菜一碟的事情了。

值得注意的是，因为所有的状态转换核心逻辑都加了互斥锁，所以 `FSM` 是并发安全的。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/fsm) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ 有限状态机定义：[https://zh.wikipedia.org/wiki/有限状态机](https://zh.wikipedia.org/wiki/有限状态机)
+ JavaScript与有限状态机：[https://www.ruanyifeng.com/blog/2013/09/finite-state_machine_for_javascript.html](https://www.ruanyifeng.com/blog/2013/09/finite-state_machine_for_javascript.html)
+ OneX 有限状态机：[https://github.com/onexstack/onex/tree/feature/onex-v2/internal/nightwatch/watcher/user](https://github.com/onexstack/onex/tree/feature/onex-v2/internal/nightwatch/watcher/user)
+ fsm GitHub 源码：[https://github.com/looplab/fsm](https://github.com/looplab/fsm)
+ fsm Documentation：[https://pkg.go.dev/github.com/looplab/fsm@v1.0.3](https://pkg.go.dev/github.com/looplab/fsm@v1.0.3)
+ 在 Go 中如何使用有限状态机优雅解决程序中状态转换问题：[https://jianghushinian.cn/2025/05/25/fsm/](https://jianghushinian.cn/2025/05/25/fsm/)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/fsm](https://github.com/jianghushinian/blog-go-example/tree/main/fsm)
+ 本文永久地址：[https://jianghushinian.cn/2025/06/01/fsm-source-code/](https://jianghushinian.cn/2025/06/01/fsm-source-code/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

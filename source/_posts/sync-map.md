---
title: 'Go 并发控制：sync.Map 详解'
date: 2025-02-10 22:34:12
tags:
- Go
- 并发编程
categories:
- Go
---

我们知道，Go 中的 `map` 类型是非并发安全的，所以 Go 就在 `sync` 包中提供了 `map` 的并发原语 `sync.Map`，允许并发操作，本文就带大家详细解读下 `sync.Map` 的原理。

<!-- more -->

### 使用示例

`sync.Map` 提供了基础类型 `map` 的常用功能，使用示例如下：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var s sync.Map

	// 存储键值对
	s.Store("name", "江湖十年")
	s.Store("age", 20)
	s.Store("location", "Beijing")

	// 读取值
	if value, ok := s.Load("name"); ok {
		fmt.Println("name:", value)
	}

	// 删除一个键
	s.Delete("age")

	// 遍历 sync.Map
	s.Range(func(key, value interface{}) bool {
		fmt.Printf("%s: %s\n", key, value)
		return true // 继续遍历
	})
}
```

`sync.Map` 是一个结构体，和其他常见的并发原语一样零值可用。`Store` 方法用于存储一个键值对，`Load` 方法根据给定的键读取对应的值，`Delete` 方法则可以删除一个键值对，`Range` 方法则用于遍历 `sync.Map` 中存储的所有键值对。

以上便是 `sync.Map` 的基本使用方法，更多功能将会在稍后的源码解读部分介绍。

### 源码解读

我们可以在 `sync.Map` 文档中看到其定义和实现的 exported 方法：

> [https://pkg.go.dev/sync@go1.23.0#Map](https://pkg.go.dev/sync@go1.23.0#Map)
>

```go
type Map
    func (m *Map) Clear()
    func (m *Map) CompareAndDelete(key, old any) (deleted bool)
    func (m *Map) CompareAndSwap(key, old, new any) (swapped bool)
    func (m *Map) Delete(key any)
    func (m *Map) Load(key any) (value any, ok bool)
    func (m *Map) LoadAndDelete(key any) (value any, loaded bool)
    func (m *Map) LoadOrStore(key, value any) (actual any, loaded bool)
    func (m *Map) Range(f func(key, value any) bool)
    func (m *Map) Store(key, value any)
    func (m *Map) Swap(key, value any) (previous any, loaded bool)
```

`sync.Map` 结构体不仅提供了 `map` 的基本功能，还提供了如 `CompareAndSwap`、`LoadOrStore` 等复合操作，随着我们对源码的深入探究，你将学会这里所有方法的原理和用途。

#### 核心结构体

`sync.Map` 主要由以下几个核心结构体组成：

> [https://github.com/golang/go/blob/go1.23.0/src/sync/map.go](https://github.com/golang/go/blob/go1.23.0/src/sync/map.go)
>

```go
type Map struct {
    mu   sync.Mutex               // 保护 dirty map 的互斥锁
    read atomic.Pointer[readOnly] // 只读 map，原子操作无需加锁
    dirty map[any]*entry          // 可变 map，包含所有数据，操作时需要加锁
    misses int                    // 记录 read map 的 miss 计数
}

type readOnly struct {
    m       map[any]*entry // 存储数据的只读 map
    amended bool           // 是否有新的 key 只存在于 dirty map
}

type entry struct {
    p atomic.Pointer[any] // 存储值的指针，支持原子操作
}

var expunged = new(any) // 用于标记从 dirty map 中被删除的元素
```

`sync.Map` 结构体字段不多，不过 `sync.Map` 源码整体上来说逻辑还是比较复杂的，并且细节很多，我先为你大致梳理下 `sync.Map` 的设计思路，方便后续理解源码。

我们可以粗略的将 `sync.Map` 的设计类比成缓存系统，当我们要读取 `map` 中某个 `key` 对应的值时，`sync.Map` 会优先从缓存中读取，如果缓存未命中，则继续从 DB 中读取。`sync.Map` 还会在特定条件满足的情况下同步缓存和 DB 中的数据。

有了缓存系统的类比，我再为你讲解 `sync.Map` 的源码，就更容易理解了。

在 `sync.Map` 结构体中，`mu` 字段是一个互斥锁，用于保护对 `dirty` 属性的操作；`dirty` 字段是一个 `map` 类型，可以类比为缓存系统中的 DB，所以理论上 `dirty` 中的数据应该是全量的；`read` 字段是一个原子类型的指针，指向 `readOnly` 结构体，`readOnly` 内部其实也是使用 `map` 来存储数据，你可以将 `read` 类比为缓存；`misses` 字段则用于计数，记录缓存未命中的次数，当我们要读取 `map` 中某个 `key` 对应的值时，优先从 `read` 读取，如果 `read` 中不存在，则 `misses` 值加 1，然后继续从 `dirty` 中读取，当 `misses` 值达到某个阈值时，`sync.Map` 就会将 `dirty` 提升为 `read`。

`readOnly` 结构体中 `m` 字段用于存储 `map` 数据；`amended` 用于标记是否有新的 `key` 只存在于 `dirty` 中，而不在 `read` 中。当我们要读取 `sync.Map` 中某个 `key` 对应的值时，会优先从 `read.m` 中读取，如果 `read.m` 中不存在，`read.amended` 则决定了是否需要继续从 `dirty` 中读取，`read.amended` 为 `false` 时，表示 `read.m` 和 `dirty` 中的数据一致，所以无需继续从 `dirty` 中读取；`read.amended` 为 `true` 时，表示 `read.m` 中的数据要落后于 `dirty`，`read.m` 存在数据缺失，所以可以尝试继续从 `dirty` 中读取，如果 `dirty` 中还是没有，才最终确定 `sync.Map` 中确实没有此 `key`。

`entry` 结构体代表一个 `map` 中的 `value`，即在 `sync.Map` 中存储的 `key` 对应的值（`value`），所有存入 `sync.Map` 中的值都会被包装成 `entry` 对象，这样可以方便标记键值对的状态。我们可以看到这里还有一个全局变量 `expunged`，值为一个任意的指针，用于标记 `sync.Map` 中的某个键值对已经从 `dirty` 中被删除。一个 `entry` 对象可能有 3 种状态，当其内部的字段 `p` 值为一个对象的指针时，表示正常状态，即这个键值对正常存在于 `sync.Map` 中；当 `p` 的值为 `nil` 时，表示这个键值对已经被删除；当 `p` 的值为 `expunged` 时，表示这个键值对已经被彻底删除。关于这两个删除状态，你可以简单的将 `nil` 类比为数据库中的逻辑删除，`expunged` 理解为物理删除，稍后阅读完源码，你就能理解其用意了。

接下来我将对 `sync.Map` 提供的方法进行一一讲解。

#### 核心方法

首先，我们来看 `sync.Map` 提供的几个比较核心的方法 `Store`、`Load`、`Delete`、`Range` 和 `Clear`。

##### Store

`Store` 方法可以将指定的键值对存入 `sync.Map`。

`Store` 方法源码如下：

```go
func (m *Map) Store(key, value any) {
	_, _ = m.Swap(key, value)
}
```

不过 `Store` 方法实际上会将操作代理到 `Swap` 方法。

`Swap` 方法能够交换给定 `key` 对应的值，并返回之前的旧值（如果存在的话），返回值 `loaded` 表示 `key` 是否存在。即 `Swap` 用于存储键值对，它可以设置一个新的键值对，也可以更新一个已存在的键值对。

`Swap` 方法源码如下：

```go
func (m *Map) Swap(key, value any) (previous any, loaded bool) {
	// 先尝试从 read map 获取 key 对应的 entry
	read := m.loadReadOnly()
	if e, ok := read.m[key]; ok { // 如果 key 存在
		// 尝试交换值
		if v, ok := e.trySwap(&value); ok { // 如果交换成功
			if v == nil { // 说明已被删除，但没有彻底删除（expunged）
				return nil, false // 新增键值对成功
			}
			return *v, true // 交换成功
		}
	}

	// 未找到或需要修改 dirty map，需要加锁处理
	m.mu.Lock()
	read = m.loadReadOnly()
	if e, ok := read.m[key]; ok { // 如果 key 在 read map 中
		if e.unexpungeLocked() {
			// 如果 entry 之前被 expunged，意味着 dirty map 不为 nil 且该 entry 不在 dirty map 中
			m.dirty[key] = e
		}
		// 进行值交换
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else if e, ok := m.dirty[key]; ok { // 如果 key 在 dirty map 中
		// 进行值交换
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else { // key 即不在 read map 中，也不在 dirty map 中
		if !read.amended {
			// 添加第一个新 key 到 dirty map，需要先标记 read map 为不完整
			m.dirtyLocked()
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
		// 在 dirty map 中创建新 entry
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
	return previous, loaded
}
```

`Swap` 方法会先尝试从 `read` 中获取给定 `key` 对应的值 `entry`，此时无需加锁。

如果 `key` 存在，说明是更新操作，那么调用 `e.trySwap(&value)` 尝试交换 `key` 对应的值。如果返回的 `ok` 为 `true` 说明交换成功，那么此时有两种情况，① 如果返回的 `v` 等于 `nil` 说明这个 `key` 以前**存在过**，但是已经被删除了，不过还没有被彻底删除（`expunged`），此时返回的 `loaded` 值为 `false` 标识 `key` 不存在，是**新增键值对成功**；② 如果 `v` 的值不为 `nil`，说明 `key` 存在，是**更新操作**，返回的 `loaded` 值设为 `true`。如果返回的 `ok` 为 `false`，说明这个 `key` 以前**存在过**，但是已经**被彻底删除了**（`expunged`），交换失败，程序继续往下执行。

这里也可以发现，`read` 其实并不完全是“只读”，在更新的情况下，其实是可以写的（调用 `e.trySwap(&value)` 会直接修改 `read.m` 中的值，稍后我们将会看到源码实现）。对于同一个键值对，`read.m` 和 `dirty` 指向的是同一个 `entry` 指针对象，所以，其实这里修改 `read` 时 `dirty` 也会同步修改。`read` 真正代表的其实是**无锁操作**，只有修改 `dirty` 的时候才需要加锁。

如果 `key` 在 `read` 中未找到，则接下来会涉及对 `dirty` 的操作，所以需要加锁。

接着，这里会再次调用 `m.loadReadOnly()` 重新加载 `read`，并检查 `read` 中是否存在 `key`，这也是保证并发安全的常见操作，双重检查（double-checking）。

如果这次检查发现 `key` 在 `read` 中，那么调用 `e.swapLocked(&value)` 将 `entry` 的值更新成新的值，并且这里也会通过 `m.dirty[key] = e` 同时更新 `dirty`（调用 `e.unexpungeLocked()` 能确保 `entry` 未被标记为 `expunged`）。

如果 `key` 不在 `read` 中，而在 `dirty` 中存在，那么直接将其更新成新的值。

如果 `key` 即不在 `read` 中，也不在 `dirty` 中，则**说明是新插入的键值对**，在 `dirty` 中创建并保存新的 `entry`。此外，这里有一个判断 `if !read.amended`，如果成立，即 `read.amended` 值为 `false`，意味着现在的 `read` 中数据是完整的，此时 `dirty` 有可能为 `nil`（后续讲解，你将知道何时会出现这种情况），调用 `m.dirtyLocked()` 能够确保 `dirty` 不为 `nil`，如果为 `nil` 则将其初始化，并且这里会将 `read.amended` 值标记为 `true`，即标记 `read.m` 是不完整的（因为这里只将 `newEntry(value)` 赋值给了 `dirty`，并没有赋值给 `read`），以后如果在 `read.m` 里查询不到 `key`，就会去 `dirty` 中继续查找。

等这一切都做完后，键值对就一定被存储到了 `sync.Map` 中，调用 `m.mu.Unlock()` 解锁，并返回存储键值对的结果。

接下来我们再看下 `Swap` 方法中的几个依赖方法是如何实现的。

首先是用于加载 `read` 的 `*Map.loadReadOnly` 方法实现：

```go
func (m *Map) loadReadOnly() readOnly {
	if p := m.read.Load(); p != nil {
		return *p
	}
	return readOnly{}
}
```

`read` 属性是 `atomic.Pointer` 类型，其 `Load` 方法用于加载指针对象，`*p` 则可以获取指针中的值。

调用 `e.trySwap(&value)` 方法可以尝试将 `entry` 中的值更新，其实现如下：

```go
func (e *entry) trySwap(i *any) (*any, bool) {
	for {
		p := e.p.Load()
		if p == expunged {
			return nil, false
		}
		if e.p.CompareAndSwap(p, i) {
			return p, true
		}
	}
}
```

这里使用 `for` 循环来执行 CAS 操作，直到操作成功返回。

如果 `entry` 的值为 `expunged`，说明此 `entry` 已被彻底删除，那么无法交换其值。**这是因为，如果某个 `entry` 对象的值被标记为 `expunged`，那么它一定是在 `read` 中有，而在 `dirty` 中没有，所以不要交换，不然 `read` 的数据就比 `dirty` 多了，这是不允许发生的（后续看到什么情况下 `entry` 被赋值为 `expunged` 你就更加清晰这里我在说什么了）。**如果允许交换，调用 `CompareAndSwap` 方法可以原子的交换值，交换成功返回 `true`，失败则进行下一轮循环。

调用 `e.unexpungeLocked()` 能确保 `entry` 未被标记为 `expunged`，其实现如下：

```go
func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return e.p.CompareAndSwap(expunged, nil)
}
```

这里仅有一行代码，如果 `entry` 的值为 `expunged`，则将其交换为 `nil`。

调用 `e.swapLocked(&value)` 方法可以更新 `entry` 对象中的值，其实现如下：

```go
func (e *entry) swapLocked(i *any) *any {
	return e.p.Swap(i)
}
```

调用 `m.dirtyLocked()` 能够确保 `dirty` 被正确初始化，其实现如下：

```go
func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read := m.loadReadOnly()
	m.dirty = make(map[any]*entry, len(read.m))
	for k, e := range read.m {
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}
```

如果 `dirty` 不为 `nil`，则无需处理，直接返回。至于 `dirty` 何时会为 `nil` 呢？这里留个疑问，后续阅读 `Load` 方法源码时，你就知道了。

否则，需要初始化一个新的 `dirty`，然后全量遍历 `read.m`，如果 `!e.tryExpungeLocked()` 成立，则将 `entry` 存入 `dirty`。这里使用遍历的方式将 `read.m` 拷贝到 `dirty`，而没有直接全量赋值，目的是为了遍历每一个对象，从而过滤出已经标记为删除的数据（值为 `nil` 和 `expunged` 的都会被清理），防止 `read` 无限增长下去（因为 `dirty` 会在特定情况下提升为 `read`，所以这里 `dirty` 过滤掉了被删除的数据，下次提升为 `read` 时，这个新的 `read` 就比旧的 `read` 中垃圾数据更少）。

这里用到的 `e.tryExpungeLocked()` 实现如下：

```go
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := e.p.Load()
	for p == nil {
		if e.p.CompareAndSwap(nil, expunged) {
			return true
		}
		p = e.p.Load()
	}
	return p == expunged
}
```

`*entry.tryExpungeLocked` 方法的作用是过滤掉值为 `nil` 或 `expunged` 的 `entry`，即已经标记为删除的键值对。

这里同样使用了 CAS 操作，将 `entry` 值为 `nil` 的, 转换为 `expunged`，表示彻底删除。

如果操作成功，直接返回，所以我们可以发现， `sync.Map` 中一个值的删除过程是，如果在 `read` 中先置为了 `nil`，之后在某次从 `read` 复制全量数据到 `dirty` 时，会将其置为 `expunged` 状态，并且不会被拷贝到 `dirty`。所以此时 `read` 和 `dirty` 中就存在数据不一致了，在 `read` 中这个值被标记为彻底删除 `expunged`，在 `dirty` 没有此键值对。所以我们在前文说：**如果某个 `entry` 对象的值被标记为 `expunged`，那么它一定是在 `read` 中有，而在 `dirty` 中没有。**

你可以回顾一下前文中讲的 `e.unexpungeLocked()` 方法能确保 `entry` 未被标记为 `expunged`，如果 `entry` 的值为 `expunged`，则将其改回 `nil`。并通过 `m.dirty[key] = e` 将这个 `entry` 赋值给 `dirty`，然后修改其值为最新值。

所以 `e.unexpungeLocked()` 方法和 `e.tryExpungeLocked()` 方法其实是有连系的，二者是相反操作，它们通过修改 `entry` 的值来标识一个键值对的状态。其实 `sync.Map` 之所以这么设计，也是为了减少 `entry` 被彻底删除的可能性，我们可以认为，某个键值对被加入进来，之后删除掉，那么很有可能下次还会被重新加入进来，所以才搞了个 `nil` 作为逻辑删除，而非彻底从 `map` 中删除，也许可以救一下。而 `expunged` 则用来处理某个被删除的对象在 `read` 中存在，而在 `dirty` 中不存在的情况，当读取 `read` 发现其已经被标记为彻底删除，那么就没比较加锁再去 `dirty` 中查找了。

并且，我们还可以发现，因为调用 `dirtyLocked` 可能全量遍历 `read.m`，所以就可能会导致 `sync.Map` 的性能退化。

调用 `newEntry(value)` 可以创建一个新的 `entry` 对象，其实现如下：

```go
func newEntry(i any) *entry {
	e := &entry{}
	e.p.Store(&i)
	return e
}
```

至此，`Store` 方法涉及的所有源码，就都讲解清楚了。

需要特别强调的是，以上提到的方法中，带有 `Locked` 结尾的方法，都需要外部持有锁才可以调用，以保证并发安全。

##### Load

`Load` 方法用于从 `sync.Map` 中读取给定 `key` 对应的值，如果 `key` 不存在，则返回值为 `nil`，且第二个返回值 `ok` 为 `false`。

`Load` 方法源码如下：

```go
func (m *Map) Load(key any) (value any, ok bool) {
	read := m.loadReadOnly() // 加载只读 map
	e, ok := read.m[key]     // 尝试从 read map 中查找 key

	// 如果 read map 中找不到 key，并且 dirty map 可能包含新键（amended 为 true），则进入慢路径
	if !ok && read.amended {
		m.mu.Lock() // 操作 dirty 需要加锁
		// 在获取锁的过程中，dirty 可能已经被提升为 read，因此需要重新加载 read map 进行 double-checking
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			// 如果在 read map 仍然找不到，则尝试从 dirty map 查找
			e, ok = m.dirty[key] // 从 dirty map 中查找 key
			// 无论 key 是否存在，记录一次 miss：
			// 在 dirty map 提升为 read map 之前，这个 key 的查询都会走慢路径
			m.missLocked() // miss 计数值 + 1
		}
		m.mu.Unlock()
	}

	// 如果最终 key 仍然不存在，则返回 nil
	if !ok {
		return nil, false
	}
	return e.load() // 返回找到的值
}
```

同 `Store` 方法一样，`Load` 方法也会优先加载只读的 `map` 对象 `read`，并尝试从 `read` 中查找 `key`。

如果 `ok` 说明找到了，那么可以直接调用 `e.load()` 加载并返回找到的值。这是一个快速路径，不需要加锁。

如果 `!ok` 说明没找到，并且 `read.amended` 为 `true` 时，说明 `dirty` 中可能包含新的键值对，则需要加锁，进入到慢路径查找。

先进行 double-checking，在获取锁的过程中，`dirty` 可能已经被提升为 `read`，因此需要重新加载 `read`。如果这次检查 `read` 还不满足，则尝试从 `dirty` 中查找，并且 `misses` 计数值加 1。

查找结束后进行解锁操作，然后根据 `ok` 的值返回结果，如果 `!ok` 说明没找到，返回 `nil`。否则说明查找到了 `key`，调用 `e.load()` 返回找到的值。

接下来我们同样列出 `Load` 方法中几个依赖方法的具体实现。

调用 `m.missLocked()` 方法可以将 `misses` 计数值加 1，其实现如下：

```go
func (m *Map) missLocked() {
	m.misses++
	if m.misses < len(m.dirty) {
		return
	}
	m.read.Store(&readOnly{m: m.dirty})
	m.dirty = nil
	m.misses = 0
}
```

可以发现，当 `misses` 的次数达到 `dirty` 中全量数据的长度时（所以 `misses` 的阈值就是 `dirty` 中全量数据的长度），**`dirty` 会被提升为 `read`（这里直接使用 `dirty` 作为 `readOnly` 的 `m` 属性值），这一步相当于刷新了缓存，更新成最新数据，并且 `dirty` 被置为了 `nil`，`misses` 计数值清零。**所以这里也解释了前文中的疑问，`dirty` 何时会为 `nil` 呢？就是在此时。那么这个目的又是什么呢？还记得前文中讲过调用 `m.dirtyLocked()` 方法时，如果 `dirty` 值为 `nil`，则会遍历 `read.m` 的数据，然后过滤掉被删除的值，将正常值存入 `dirty` 吗，这就是为了有一个机会，能够清理被标记删除的对象，以防止 `sync.Map` 无限增长。

这里有个细节，构造 `readOnly` 对象时 `amended` 没有赋值，默认值为 `false`，标识 `read` 和 `dirty` 现在数据是等价的，`read` 是全量数据（其实 `dirty` 中此时数据为 `nil`，所以此时以 `read` 数据为准）。后续查找 `key` 时，如果 `read` 中没找到，就无需再加锁继续查找 `dirty` 了。

调用 `e.load()` 能够加载 `entry` 对象中记录的具体值，其实现如下：

```go
func (e *entry) load() (value any, ok bool) {
	p := e.p.Load()
	if p == nil || p == expunged {
		return nil, false
	}
	return *p, true
}
```

这里无需过多解释，就是从 `atomic.Pointer` 类型中获取包含的值。

那么现在 `Load` 方法也讲解完成了。

其实到这里我们可以总结出，`sync.Map` 之所以设计出 `read` + `dirty` 两层结构，其实就是以空间换时间的思路，做了两层的数据冗余，对于 `read` 的操作都是原子性的，无需加锁，而 `read` 中数据存在滞后的情况下，再加锁访问 `dirty`。

##### Delete

`Delete` 方法用于从 `sync.Map` 中删除指定的键值对。

`Delete` 方法源码如下：

```go
func (m *Map) Delete(key any) {
	m.LoadAndDelete(key) // 直接调用 LoadAndDelete 进行删除，忽略其返回值
}
```

在 `Delete` 方法内部，直接调用了 `LoadAndDelete` 方法。`LoadAndDelete` 方法会删除 `key` 对应的值，并返回删除前的值（如果存在），第二个返回值 `loaded` 标识该 `key` 是否存在。

`LoadAndDelete` 方法源码如下：

```go
func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
	// 先从 read map 读取，避免加锁
	read := m.loadReadOnly() // 加载只读 map
	e, ok := read.m[key]
	if !ok && read.amended {
		// 若 read map 没有 key，且 dirty map 可能包含该 key，则加锁检查 dirty map
		m.mu.Lock()
		read = m.loadReadOnly() // double-checking
		e, ok = read.m[key]
		if !ok && read.amended {
			// 在 dirty map 查找 key
			e, ok = m.dirty[key]
			delete(m.dirty, key) // 删除 dirty map 中的 key
			// 记录一次 miss，表示该 key 走了慢路径，直到 dirty map 提升为 read map
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
		// 若 key 存在，调用 entry.delete() 删除值并返回
		return e.delete()
	}
	return nil, false // read map 和 dirty map 都无此 key
}
```

有了前面两个方法的讲解，现在阅读 `LoadAndDelete` 方法的源码就轻松多了。

同样的套路，优先加载只读 `map`，先尝试从 `read` 中查找 `key`。若 `read` 中没有该 `key`，且 `dirty` 中可能包含该 `key`，则加锁检查 `dirty`。同样做了 double-checking 处理，再次查询 `read` 依然不满足，才尝试从 `dirty` 中查找。这里的操作是先查找再删除 `dirty` 中的 `key`，这也正是 `Load` 和 `Delete` 两个操作。此外，这里还会记录一次 `misses`。

`Load` 和 `Delete` 操作都结束后进行解锁操作，之后根据 `ok` 的值，返回结果，如果 `ok` 则说明查找到了 `key`，调用 `e.delete()` 删除值并返回，否则说明未查到该 `key`，返回 `nil`。

`LoadAndDelete` 方法中依赖的 `*entry.delete` 方法实现如下：

```go
func (e *entry) delete() (value any, ok bool) {
	for {
		p := e.p.Load()
		if p == nil || p == expunged {
			return nil, false
		}
		if e.p.CompareAndSwap(p, nil) {
			return *p, true
		}
	}
}
```

当 `p` 的值为 `nil` 或 `expunged` 时，说明该 `key` 已经被删除了，所以返回的 `ok` 值为 `false`，否则通过 CAS 操作对其进行删除，删除成功返回 `true`。

**注意这里的删除只会将 `entry` 中存储的值置为 `nil`，并不会彻底删除 `entry` 对象。**

##### Range

`Range` 方法可以遍历 `sync.Map` 中所有的键值对，并依次调用给定的 `f(key, value)` 函数，如果 `f` 返回 `false`，则停止遍历。

`Range` 方法源码如下：

```go
func (m *Map) Range(f func(key, value any) bool) {
	// 读取当前的 read map
	read := m.loadReadOnly()

	// 如果 read map 是完整的（未修改），直接遍历
	if read.amended {
		// 若 read.amended == true，说明 dirty map 存在新增 key，需要提升为 read
		m.mu.Lock()
		read = m.loadReadOnly()
		if read.amended {
			// 提升 dirty map 为 read
			read = readOnly{m: m.dirty}
			copyRead := read
			m.read.Store(&copyRead)
			// 清空 dirty map 并重置 miss 计数
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}

	// 遍历 read map
	for k, e := range read.m {
		v, ok := e.load()
		if !ok {
			continue
		}
		// 调用用户提供的函数 f
		if !f(k, v) {
			break // f 返回 false 时停止遍历
		}
	}
}
```

对于 `Range` 操作来说，只需要遍历 `map`，即这是一个读取操作，所以会读取当前的 `read` 值，如果 `read` 是完整的（`read.amended == false`），那么可以直接遍历 `read`，并且每次 `for` 循环中都会调用用户提供的函数 `f`，如果 `f` 返回 `false`，则停止遍历。

如果 `read.amended` 值为 `true`，说明 `dirty` 中存在新增的 `key`，那么就将其提升为 `read`，然后再进行 `read` 遍历。因为需要操作 `dirty`，所以会加锁，并且进行了 double-checking，在提升 `dirty` 为 `read` 后，会清空 `dirty` 并重置 `misses` 计数。所以说调用 `Range` 方法也是一个可能导致后面 `sync.Map` 会出现性能退化的原因。

这里将 `dirty` 提升为 `read` 的逻辑跟在 `Load` 方法中调用 `m.missLocked()` 的逻辑类似。

##### Clear

`Clear` 方法会删除所有键值对，使 `sync.Map` 变为空。

`Clear` 方法源码如下：

```go
func (m *Map) Clear() {
	read := m.loadReadOnly() // 加载只读 map
	// 如果 read map 为空且 dirty map 也未修改（amended 为 false），则无需执行清理
	if len(read.m) == 0 && !read.amended {
		// 如果 map 已经为空，则避免分配新的 readOnly。
		return
	}

	m.mu.Lock() // 加锁
	defer m.mu.Unlock()

	// double-checking
	// 重新加载 read map，避免在获取锁的过程中发生变化
	read = m.loadReadOnly()
	if len(read.m) > 0 || read.amended {
		// 清空 read map，存储一个新的空 readOnly 结构
		m.read.Store(&readOnly{})
	}

	// 清空 dirty map
	clear(m.dirty)
	// 复位 miss 计数，防止下一次操作时刚清空的 dirty map 立即被提升到 read map
	m.misses = 0
}
```

`Clear` 方法首先会加载 `read`，并判断 `read.m` 是否为空，如果为空且 `dirty` 中也不存在新的 `key`，则无需进一步处理，直接返回。否则，需要加锁后进一步处理。在 double-checking 后会再次判断 `read.m` 是否为空，如果不为空或 `read.amended` 为 `true`，则清空 `read`。最终，再清空 `dirty` 并重置 `misses` 计数。

#### 复合操作方法

`sync.Map` 不仅提供了 `map` 的常规操作方法，它还实现了几个复合操作方法 `LoadAndDelete`、`LoadOrStore`、`CompareAndSwap` 和 `CompareAndDelete`。所谓复合方法，就是一个方法同时支持两个以上的操作，而且这几个复合方法都能够见名知意。

其中 `LoadAndDelete` 方法前文中已经讲解，那么接下来就介绍下其他几个方法。

##### LoadOrStore

`LoadOrStore` 方法用来获取或保存一个键值对，如果 `key` 存在，则返回其当前值，否则存储并返回给定的 `value`，`loaded` 返回值表示是否是从 `map` 中加载的值（`true` 表示 `key` 已存在，`false` 表示存储了新值）。

`LoadOrStore` 方法源码如下：

```go
func (m *Map) LoadOrStore(key, value any) (actual any, loaded bool) {
	// 优先检查只读 map，避免加锁，提高并发性能
	read := m.loadReadOnly() // 加载只读 map
	if e, ok := read.m[key]; ok {
		// 尝试加载或存储值
		actual, loaded, ok := e.tryLoadOrStore(value)
		if ok {
			return actual, loaded
		}
	}

	// 加锁处理
	m.mu.Lock()
	defer m.mu.Unlock()

	// 进入慢路径

	// 重新加载 read map 以防数据变化
	read = m.loadReadOnly()
	if e, ok := read.m[key]; ok { // key 在 read map 中
		// 解除 expunged 状态并放入 dirty map
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
		// 尝试加载或存储
		actual, loaded, _ = e.tryLoadOrStore(value)
	} else if e, ok := m.dirty[key]; ok { // key 仅存在于 dirty map 中
		actual, loaded, _ = e.tryLoadOrStore(value)
		m.missLocked() // 记录一次 miss
	} else { // 该 key 完全不存在
		if !read.amended {
			// 这是 dirty map 第一次存入新 key，需要确保它已分配，并标记 read map 为不完整
			m.dirtyLocked()
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
		// 在 dirty map 中存储新值
		m.dirty[key] = newEntry(value)
		actual, loaded = value, false
	}

	return actual, loaded
}
```

`LoadOrStore` 方法先尝试从 `read` 中获取给定 `key` 对应的值，如果存在，则调用 `e.tryLoadOrStore(value)` 尝试加载或存储值，并且根据返回值 `ok` 来决定是否返回。

其中 `*entry.tryLoadOrStore` 方法实现如下：

```go
func (e *entry) tryLoadOrStore(i any) (actual any, loaded, ok bool) {
	p := e.p.Load()
	if p == expunged {
		// entry 被标记为 expunged，不可修改
		return nil, false, false
	}
	if p != nil {
		// entry 已经存在，loaded 为 true，返回当前值
		return *p, true, true
	}

	ic := i
	// 循环执行 CAS 操作，直到成功返回
	for {
		// 如果当前值为 nil，则尝试存储新值
		if e.p.CompareAndSwap(nil, &ic) {
			return i, false, true // 存储成功，loaded 为 false
		}

		// 每次循环都要执行一次检查
		p = e.p.Load()
		if p == expunged {
			// entry 被 expunged，返回失败
			return nil, false, false
		}
		if p != nil {
			// entry 已经存在，loaded 为 true，返回当前值
			return *p, true, true
		}
	}
}
```

如果 `entry` 已经被标记为 `expunged`，`tryLoadOrStore` 不会修改 `entry` 的值，并且返回值 `ok` 置为 `false`。如果 `entry` 中的值不为 `nil`，说明是正常状态的值，直接返回其 `value`。否则 `entry` 中的值被标记为 `nil`，`tryLoadOrStore` 会继续通过 CAS 操作原子地存储或加载一个值。

如果当前 `entry` 中的值为 `nil`，那么 CAS 操作就会成功，返回 `loaded` 为 `false`。在并发情况下，当前 `entry` 中的值可能会被其他 goroutine 修改成功，所以 CAS 操作如果失败，则还需要对 `entry` 中的值状态再次做检查。`for` 循环最终会在某个 case 操作成功后返回。

然后我们再回到 `LoadOrStore` 方法中的逻辑继续向下看。

如果 `LoadOrStore` 方法在 `read` 中没有找到给定的 `key`，或者 `e.tryLoadOrStore(value)` 失败，则需要加锁后做进一步检查。如果这一次在 `read` 中找到了给定的 `key`，则将 `entry` 记录到 `dirty` 中，并调用 `e.tryLoadOrStore(value)` 尝试加载或存储值。值得注意的是，这里会先调用 `e.unexpungeLocked()` 确保 `dirty` 未被标记为 `expunged`。如果 `key` 不在 `read` 中，而在 `dirty` 中存在，那么直接将其更新成新的值，并记录一次 `misses`。如果 `key` 即不在 `read` 中，也不在 `dirty` 中，则**说明是新插入的键值对**，在 `dirty` 中创建并保存新的 `entry`。

这段逻辑与 `Swap` 方法中的逻辑非常相似，你可以对比学习。

##### CompareAndSwap

`CompareAndSwap` 方法仅在给定的 `key` 存在且对应的值等于传入的 `old` 时（`old` 必须是可比较的类型），将其替换为 `new`。

`CompareAndSwap` 方法源码如下：

```go
func (m *Map) CompareAndSwap(key, old, new any) (swapped bool) {
	// 先尝试从 read map 查找 key
	read := m.loadReadOnly()
	if e, ok := read.m[key]; ok { // 如果 key 在 read map 中
		// 直接在 entry 上尝试 CompareAndSwap
		return e.tryCompareAndSwap(old, new)
	} else if !read.amended {
		// 如果 key 不存在，并且 read map 没有标记为 amended，说明 dirty map 也不会有 key
		return false
	}

	// 进入 dirty map 处理，需要加锁
	m.mu.Lock()
	defer m.mu.Unlock()
	read = m.loadReadOnly()

	if e, ok := read.m[key]; ok { // 如果 key 在 read map 中
		swapped = e.tryCompareAndSwap(old, new)
	} else if e, ok := m.dirty[key]; ok { // 如果 key 在 dirty map 中
		swapped = e.tryCompareAndSwap(old, new)
		// 这里虽然 CompareAndSwap 不会修改 map 的 key 集合，
		// 但因为涉及锁的操作，需要增加一次 miss 计数，
		// 这样 dirty map 之后会被提升为 read，提高访问效率。
		m.missLocked()
	}
	return swapped
}
```

如果给定的 `key` 在 `read` 中存在，则直接调用 `e.tryCompareAndSwap(old, new)` 尝试比较并交换。

`*entry.tryCompareAndSwap` 方法实现如下：

```go
func (e *entry) tryCompareAndSwap(old, new any) bool {
	// 加载当前 entry 存储的值
	p := e.p.Load()
	// 如果值为 nil（条目已删除）或 expunged（条目已被彻底删除）或当前值不等于 old，则返回 false
	if p == nil || p == expunged || *p != old {
		return false
	}

	// 复制 new 以优化逃逸分析，避免在比较失败时不必要的堆分配
	nc := new
	// 循环执行 CAS 操作，直到成功返回
	for {
		// 尝试使用原子操作 CompareAndSwap 替换值
		if e.p.CompareAndSwap(p, &nc) {
			return true
		}

		// 每次循环都要执行一次检查
		// 重新加载最新的值，确保并发情况下仍满足条件
		p = e.p.Load()
		// 如果值发生变化，或者条目已删除，则返回 false
		if p == nil || p == expunged || *p != old {
			return false
		}
	}
}
```

`tryCompareAndSwap` 方法会比较 `entry` 中的值是否等于给定的 `old` 值，如果相等且它的值未被删除（未被标记为 `nil` 或 `expunged`），则将其替换为 `new` 值。如果它的值已被删除或不等于 `old` 值，则会返回 `false`，且不对 `entry` 做任何修改。

回到 `CompareAndSwap` 方法逻辑中，如果给定的 `key` 不在 `read` 中且 `read.amended` 为 `true`，则需要加锁继续到 `dirty` 中查找，且 `misses` 计数值加 1。

这个方法的整体逻辑还是比较简单的。

##### CompareAndDelete

`CompareAndDelete` 方法仅在给定的 `key` 存在且对应的值等于 `old` 时删除该键值对。

`CompareAndDelete` 方法源码如下：

```go
func (m *Map) CompareAndDelete(key, old any) (deleted bool) {
	// 先尝试从 read map 查找 key
	read := m.loadReadOnly()
	e, ok := read.m[key]

	// 如果 key 不在 read map 中，但 read.amended == true，说明可能在 dirty map
	if !ok && read.amended {
		m.mu.Lock() // 进入 dirty map 处理，需要加锁
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// 不直接从 m.dirty 删除 key，而是仅执行“compare”逻辑。
			// 当 dirty map 未来被提升为 read 时，该 entry 会被标记 expunged。
			//
			// 记录一次 miss，这样该 key 未来会走慢路径，直到 dirty map 变为 read。
			m.missLocked()
		}
		m.mu.Unlock()
	}

	// 如果 key 存在于 read map 或 dirty map
	for ok {
		p := e.p.Load()
		// 如果值已被删除（nil）、expunged（彻底删除）或者值不等于 old，则返回 false
		if p == nil || p == expunged || *p != old {
			return false
		}
		// CAS 交换值为 nil，成功删除
		if e.p.CompareAndSwap(p, nil) {
			return true
		}
	}
	return false
}
```

这个方法主要逻辑也比较简单，我就不一行一行的讲解了。值得一提的是，这里的删除操作与 `Delete` 方法一样也是通过 CAS 操作将 `entry` 中的值改为 `nil` 来实现的。

至此，`sync.Map` 中的所有源码咱们就都讲解完成了。

### 总结

`sync.Map` 的用法非常简单，它提供了 `map` 的基础功能，并且还扩展了几个复合功能。

要读懂 `sync.Map` 的源码，可以将其类比为缓存系统，先对其设计思路做到心中有数，再阅读源码，不然很容易陷入到细节而不知所措。

你需要记住 `sync.Map` 的主体逻辑是，所有方法都优先尝试无锁操作 `read`，如果 `read` 条件不满足，再尝试加锁操作 `dirty`。`read` 和 `dirty` 存储键值对时，指向的是同一个 `entry` 对象。对于 `entry` 实例，你需要理解其 3 种状态，`nil`、`expunged` 和正常指针对象，它们之间是如何转换的。

所有对 `read` 的操作都无需加锁，对 `dirty` 的操作才需要加锁，再结合 `entry` 的 `nil` 和 `expunged` 两种状态，`sync.Map` 可以尽可能实现无锁操作。不过，`Range` 方法和 `m.missLocked()` 操作都有可能使 `dirty` 为 `nil`，这会导致 `sync.Map` 后续操作出现性能退化，需要格外小心。此外，其实严格来讲，`read` 并不代表只读 `map`，部分更新、删除动作也都会操作 `read`，只有新插入一个键值对时才一定要操作 `ditry`。

这篇文章内容有点长，有些文字写的也很啰嗦，但这都是笔者有意而为之，目的是让你加深印象，多读几遍，理解更加深刻。只有明白了 `sync.Map` 的原理，才能将其用好。最后强调一点，谨慎起见，必要时还是要进行性能测试来验证 `sync.Map` 是否能满足我们的业务需求。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ sync.Map Documentation：[https://pkg.go.dev/sync@go1.23.0#Map](https://pkg.go.dev/sync@go1.23.0#Map)
+ sync.Map GitHub 源码：[https://github.com/golang/go/blob/go1.23.0/src/sync/map.go](https://github.com/golang/go/blob/go1.23.0/src/sync/map.go)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/sync/map](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

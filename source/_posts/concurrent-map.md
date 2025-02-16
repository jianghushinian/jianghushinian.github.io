---
title: 'Go 并发编程：如何实现一个并发安全的 map'
date: 2025-02-16 14:22:57
tags:
- Go
- 并发编程
categories:
- Go
---

上周发布的文章「[Go 并发控制：sync.Map 详解](https://jianghushinian.cn/2025/02/10/sync-map/)」有读者反馈说我写的太难了，上来就挑战源码，对新手不够友好。所以这篇文章算作补充，从入门到进阶的顺序讲解一下在 Go 中如何自己实现一个并发安全的 `map`。

<!-- more -->

### 内置 map

首先，我们来测试一下 Go 语言内置 `map` 并发安全性，示例如下：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/main.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/main.go)
>

```go
package main

import "fmt"

// NOTE: Go 内置 map 不支持并发读写

func main() {
	m := make(map[string]int)

	// 并发读写 map
	go func() {
		for {
			m["k"] = 1
			fmt.Println("set k:", 1)
		}
	}()

	go func() {
		for {
			v, _ := m["k"]
			fmt.Println("read k:", v)
		}
	}()

	select {}
}
```

这里开了两个 goroutine 分别用来读写 `map` 对象变量 `m`，并且每个 goroutine 中都采用无限循环来不停的读写 `m`，这样就会造成大量并发操作。程序最后使用空的 `select{}` 语句来实现阻塞。

执行示例代码，你将得到 `panic`，输出日志太长我就不贴出来了，主要报错信息为 `fatal error: concurrent map read and map write`。这提示我们存在并发读写 `map` 的操作，所以说 Go 内置 `map` 不是并发安全的。

### 实现并发安全 map

要实现 `map` 的并发安全，我们首先想到的就是互斥锁 `sync.Mutex`。不过 Go 语言还在 `sync.Mutex` 的基础上提供了读写锁 `sync.RWMutex`，能够更大的程度的提升**读多写少场景**的并发性能，所以我们选用读写锁来实现并发安全的 `map`。

实现实例如下：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/mutex_map.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/mutex_map.go)
>

```go
type RWMutexMap struct {
	rw sync.RWMutex
	m  map[string]int
}

func NewRWMutexMap() *RWMutexMap {
	return &RWMutexMap{
		m: make(map[string]int),
	}
}

func (m *RWMutexMap) Set(key string, v int) {
	m.rw.Lock()
	defer m.rw.Unlock()
	m.m[key] = v
}

func (m *RWMutexMap) Get(key string) (int, bool) {
	m.rw.RLock()
	defer m.rw.RUnlock()
	v, ok := m.m[key]
	return v, ok
}

func (m *RWMutexMap) Del(key string) {
	m.rw.Lock()
	defer m.rw.Unlock()
	delete(m.m, key)
}
```

这里我们定义 `RWMutexMap` 结构用来表示并发安全的 `map`，其包含两个属性，字段 `rw` 即为读写锁，用来保证并发安全，`m` 字段就是普通的 `map`，用来存取数据。

`Set` 方法和 `Del` 方法属于写操作，所以在修改 `m` 字段数据时都加了写锁 `m.rw.Lock()`，并在操作完成后释放写锁 `defer m.rw.Unlock()`。而 `Get` 方法属于读操作，所以在获取 `m` 字段数据时加了读锁 `m.rw.RLock()`，并在获取完成后释放读锁 `defer m.rw.RUnlock()`。

这里有必要解释一下读锁和写锁的区别：

写锁就是互斥锁，不可以被多个 goroutine 同时持有，比如在调用写锁 `m.rw.Lock()` 时，如果有读锁或写锁已经被其他 goroutine 持有，则当前 goroutine 会被阻塞。

读锁类似可重入锁，多个 goroutine 可以同时持有读锁，比如在调用读锁 `m.rw.RLock()` 时，如果有其他 goroutine 也持有读锁，则当前 goroutine 还可以继续获取这把读锁，但此时如果有 goroutine 尝试获取写锁则会被阻塞。

所以说读写锁对读多写少的场景更加适用，读写锁允许多个 goroutine 同时持有读锁，但只允许一个 goroutine 持有写锁。

此时，我们可以写出同样的测试代码，来测试这个使用读写锁实现的并发安全 `map` 的效果：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/mutex_map_test.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/mutex_map_test.go)
>

```go
// NOTE: 这个测试不会终止
func TestRWMutexMap(t *testing.T) {
	m := NewRWMutexMap()

	// 并发读写 map
	go func() {
		for {
			m.Set("k", 1)
			fmt.Println("set k:", 1)
		}
	}()

	go func() {
		for {
			v, _ := m.Get("k")
			fmt.Println("read k:", v)
		}
	}()

	select {}
}
```

这个示例不会终止，会交替打印 `read` 和 `set` 的日志。

没错，实现一个并发安全的 `map` 就是如此的简单，这么一对比，`sync.Map` 的实现确实很复杂。

### 分片 map

我们知道加锁会使程序的性能急剧下降，读写锁能够在读多写少的场景中降低互斥锁对性能的影响。那么我们实现的并发安全 `map` 是否还有可优化空间呢？答案是肯定的。

我们可以采用分片的思想，将一个大的 `map` 拆成多个更小的 `map`，然后让每个小 `map` 持有各自的锁，以此来减小锁的粒度，减少并发情况下加锁的次数。

实现示例如下：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/sharding_map.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/sharding_map.go)
>

```go
package main

import (
	"hash/maphash"
	"sync"
)

var seed = maphash.MakeSeed()

func hashKey(key string) uint64 {
	return maphash.String(seed, key)
}

type ShardingMap struct {
	locks  []sync.RWMutex
	shards []map[string]int
}

func NewShardingMap(size int) *ShardingMap {
	sm := &ShardingMap{
		locks:  make([]sync.RWMutex, size),
		shards: make([]map[string]int, size),
	}
	for i := 0; i < size; i++ {
		sm.shards[i] = make(map[string]int)
	}
	return sm
}

func (m *ShardingMap) getShardIdx(key string) uint64 {
	hash := hashKey(key)
	return hash % uint64(len(m.shards))
}

func (m *ShardingMap) Set(key string, value int) {
	idx := m.getShardIdx(key)
	m.locks[idx].Lock()
	defer m.locks[idx].Unlock()
	m.shards[idx][key] = value
}

func (m *ShardingMap) Get(key string) (int, bool) {
	idx := m.getShardIdx(key)
	m.locks[idx].RLock()
	defer m.locks[idx].RUnlock()
	value, ok := m.shards[idx][key]
	return value, ok
}

func (m *ShardingMap) Del(key string) {
	idx := m.getShardIdx(key)
	m.locks[idx].Lock()
	defer m.locks[idx].Unlock()
	delete(m.shards[idx], key)
}
```

`ShardingMap` 表示一个采用分片机制的并发安全 `map`，它有两个属性 `locks` 和 `shards`，它们都是切片类型，`locks` 用来保存每个分片的读写锁，`shards` 用来保存每个分片的数据。

通过函数 `NewShardingMap` 可以创建一个 `ShardingMap` 对象，它接收一个参数 `size` 用来初始化分片数量，即 `locks` 和 `shards` 的切片长度。在构造 `ShardingMap` 对象时会为 `shards` 中的每个 `map` 进行初始化，而由于 `sync.RWMutex` 零值可用，所以不必对 `locks` 中的每把读写锁进行初始化操作。

`getShardIdx` 方法非常重要，会被所有方法使用，它可以计算给定 `key` 所在的分片索引。首先它调用辅助函数 `hashKey` 对给定 `key` 计算出一个 `hash` 值，然后用这个 `hash` 值对分片长度做取模运算，这样就能得到一个值必然会落到某个分片的索引。这也是分片 `map` 的精髓所在，给定一个 `key` 为其计算一个分片索引值，然后将其存入对应的分片，这个过程不用加锁。如果 `hashKey` 函数实现的足够好，那么每个分片中分布的数据量也是比较均匀的。

接下来就是 `map` 的基本操作方法 `Set`、`Get` 和 `Del` 的实现了，它们内部都会先调用 `m.getShardIdx(key)` 获取分片索引，然后对指定的分片进行加锁，最后再操作对应分片中的数据。这样，当多个 goroutine 并发读写 `map` 时，只要对应的 `key` 不在同一个分片中，那么就能降低加锁次数，从而提升程序性能。只有对应的 `key` 计算后在同一个分片中，多个 goroutine 之间才需要抢锁。

看到这，你应该能够想到，如果我们要用 `map` 存储大量数据，则使用分片机制的并发安全 `map` 实现效果更好。

### 符合 Go 哲学的并发安全 map？

文章写到这里，其实 Go 中常见的实现并发安全 `map` 的方式基本就介绍完了。不过，这时候好像缺了点什么，Go 并发之道大家可是更推荐使用 channel 而非互斥锁，那么我们何不用 channel 来实现一个符合 Go 哲学的并发安全 `map` 呢？

实现示例如下：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/channel_map.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/channel_map.go)
>

```go
type ChannelMap struct {
	cmd chan command
	m   map[string]int
}

type command struct {
	action string // "get", "set", "delete"
	key    string
	value  int
	result chan<- result
}

type result struct {
	value int
	ok    bool
}

func NewChannelMap() *ChannelMap {
	sm := &ChannelMap{
		cmd: make(chan command),
		m:   make(map[string]int),
	}
	go sm.run()
	return sm
}

func (m *ChannelMap) run() {
	for cmd := range m.cmd {
		switch cmd.action {
		case "get":
			value, ok := m.m[cmd.key]
			cmd.result <- result{value, ok}
		case "set":
			m.m[cmd.key] = cmd.value
		case "delete":
			delete(m.m, cmd.key)
		}
	}
}

func (m *ChannelMap) Set(key string, value int) {
	m.cmd <- command{action: "set", key: key, value: value}
}

func (m *ChannelMap) Get(key string) (int, bool) {
	res := make(chan result)
	m.cmd <- command{action: "get", key: key, result: res}
	r := <-res
	return r.value, r.ok
}

func (m *ChannelMap) Del(key string) {
	m.cmd <- command{action: "delete", key: key}
}
```

`ChannelMap` 结构体表示我们用 channel 实现的并发安全 `map`，它有两个属性，`cmd` 字段是一个 `chan command` 类型用来传递操作指令和操作结果，`m` 字段用来存储 `map` 数据。

`command` 结构是用来在 channel 中传递信息的，`action` 表示操作指令，比如 `get`、`set` 和 `delete` 操作，`key` 用来记录我们在调用 `Set`、`Get` 和 `Del` 方法时要操作的 `map` 中的 `key`，而 `value` 则用来记录调用 `Set` 方法时指定的 `value` 值，最后一个 `result` 字段用来记录调用 `Set`、`Get` 和 `Del` 方法时返回的结果。

构造函数 `NewChannelMap` 可以创建一个 `ChannelMap` 对象，其内部不仅会初始化一个 `ChannelMap` 对象，还会调用 `go sm.run()` 启动一个新的 goroutine 来执行后台任务。

这个后台任务就是 `ChannelMap` 的核心所在，在 `run` 方法内部，使用 `for` 循环来不断的从 `m.cmd` 这个 channel 中获取操作指令，根据 `cmd.action` 我们能够判断出用户是调用了 `Set`、`Get` 和 `Del` 中的哪个方法，接着做相应的操作。比如如果是调用了 `Get` 方法，就使用 `m.m[cmd.key]` 从 `map` 中获取 `key` 对应的值 `value`，然后构造 `result` 对象，并将其通过 `cmd.result` 这个 channel 传递给调用方。

接下来看到 `Set`、`Get` 和 `Del` 这几个方法的实现，你就能彻底理解 `ChannelMap` 的设计思路了。还是拿 `Get` 方法举例，这里先是构造了一个 `chan result` 对象用来保存结果，并将它和给定的 `key` 一起构造出一个 `command` 对象传递给 `m.cmd`。当 `command` 被传递过去后，`run` 方法内部的 `for` 循环就能拿到这个 `command` 对象了，然后将结果通过我们构造的 `chan result` 对象再传递回来，一次 `Get` 操作就完成了。

其他两个方法同理，这里的核心就是通过 `m.cmd` 这个无缓冲的 channel 实现并发控制，来保证多个 goroutine 同时操作 `map` 时的并发安全。

### 如何选择

现在，我们掌握了多种并发安全 map 的实现，比如互斥锁、读写锁、分片以及 channel 实现，还有不要忘记 Go 内置的 `sync.Map` 实现。那么，我们到底该如何选择使用哪种实现呢？

我们可以对其进行一轮基准测试，来测试下不同 `map` 实现的性能差距。示例如下：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/sharding_map_test.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/sharding_map_test.go)
>

```go
// 读写次数一致
func BenchmarkSyncMap(b *testing.B) {
	var m sync.Map
	var wg sync.WaitGroup
	for i := 0; i < b.N; i++ {
		wg.Add(1)
		go func(k int) {
			defer wg.Done()
			key := fmt.Sprintf("key%d", k)
			m.Store(key, k)
			m.Load(key)
		}(i % 100000) // 使用 100000 个不同的 key
	}
	wg.Wait()
}
```

这是读写次数一致的场景下对 `sync.Map` 进行的基准测试代码，其他几种 `map` 的基准测试代码一样，我就不都贴出来了。

执行示例代码，可以得到如下输出：

```bash
$ go test -bench="Map$" -benchmem -run="^$"
goos: darwin
goarch: amd64
pkg: github.com/jianghushinian/blog-go-example/sync/concurrent-map
cpu: Intel(R) Core(TM) i5-7360U CPU @ 2.30GHz
BenchmarkChannelMap-4             551652              2670 ns/op             478 B/op          6 allocs/op
BenchmarkRWMutexMap-4            1000000              1474 ns/op             196 B/op          4 allocs/op
BenchmarkMutexMap-4              2156851               571.5 ns/op            68 B/op          4 allocs/op
BenchmarkShardingMap-4           2752004               444.5 ns/op            66 B/op          3 allocs/op
BenchmarkSyncMap-4               1765488               618.8 ns/op           115 B/op          7 allocs/op
PASS
ok      github.com/jianghushinian/blog-go-example/sync/concurrent-map   10.271s
```

根据这轮基准测试结果我们可以发现，在读写次数一致情况下，对并发安全的 `map` 进行 `100000` 次读写操作，使用 channel 实现的 `map` 性能最差，使用分片机制实现的 `map` 性能最好。

那么我们再测一测读多写少的场景。示例如下：

> [https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/sharding_map_test.go](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map/sharding_map_test.go)
>

```go
// 读多写少
func BenchmarkSyncMapReadHeavy(b *testing.B) {
	var m sync.Map
	var wg sync.WaitGroup
	for i := 0; i < b.N; i++ {
		for j := 0; j < 100; j++ {
			wg.Add(1)
			go func(k int) {
				defer wg.Done()
				m.Load(string(rune(k)))
			}(j)
		}
		for j := 0; j < 10; j++ {
			wg.Add(1)
			go func(k int) {
				defer wg.Done()
				m.Store(string(rune(k)), k)
			}(j)
		}
	}
	wg.Wait()
}
```

执行示例代码，可以得到如下输出：

```bash
$ go test -bench="MapReadHeavy$" -benchmem -run="^$"
goos: darwin
goarch: amd64
pkg: github.com/jianghushinian/blog-go-example/sync/concurrent-map
cpu: Intel(R) Core(TM) i5-7360U CPU @ 2.30GHz
BenchmarkChannelMapReadHeavy-4              8490            212698 ns/op           33944 B/op        489 allocs/op
BenchmarkRWMutexMapReadHeavy-4             30123             37994 ns/op            5342 B/op        230 allocs/op
BenchmarkMutexMapReadHeavy-4               33726             42250 ns/op            5352 B/op        230 allocs/op
BenchmarkShardingMapReadHeavy-4            31478             36449 ns/op            5320 B/op        230 allocs/op
BenchmarkSyncMapReadHeavy-4                35464             36445 ns/op            5640 B/op        250 allocs/op
PASS
ok      github.com/jianghushinian/blog-go-example/sync/concurrent-map   10.816s
```

从这一轮基准测试结果来看，依然是使用 channel 实现的 `map` 性能最差，而使用分片机制实现的 `map` 和 `sync.Map` 性能不相上下。

所以，这几个并发安全 `map` 的实现该怎么选择，你心目中有了答案吗？

我的答案是，不要看我的基准测试结果，很可能换一下数据量重新测试你会得到不一样的结果。更不要人云亦云，比如不要被一些网上的言论所惑，并不像有些人鼓吹的那样，Go 并发编程只用 channel，你看使用 channel 实现的并发安全 `map` 性能反而最差。我想表达的是，你应该根据你预估的数据量来跑基准测试，以你自己的基准测试结果为选择依据。即使是 channel 也不是万能的，Go 中每一种并发原语都有它的适用场景，实践才是检验真理的唯一标准。

### 总结

Go 中内置的 `map` 不是并发安全的，而 `map` 是一种很常用的类型，为此 Go 在 `sync` 包中又内置了 `sync.Map` 来支持并发安全。

除此以外，我们还可以使用 Go 的并发原语自己来实现并发安全的 `map`，比如使用互斥锁、读写锁、分片机制以及 channel。每一种 `map` 的实现都有其特点和适用场景，我们应该根据自己的数据量对这些实现进行基准测试，以此来选择合适的 `map` 实现。

另外，为了降低读者阅读代码时的心智负担，本文中没有介绍泛型版本的并发安全 `map` 实现。本文原理已经讲解清楚了，对于泛型版本的实现，就留作作业交给你自己去完成了。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ sync.Map Documentation：[https://pkg.go.dev/sync@go1.23.0#Map](https://pkg.go.dev/sync@go1.23.0#Map)
+ sync.Map GitHub 源码：[https://github.com/golang/go/blob/go1.23.0/src/sync/map.go](https://github.com/golang/go/blob/go1.23.0/src/sync/map.go)
+ Go 并发控制：sync.Map 详解：[https://jianghushinian.cn/2025/02/10/sync-map/](https://jianghushinian.cn/2025/02/10/sync-map/)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map](https://github.com/jianghushinian/blog-go-example/tree/main/sync/map/concurrent-map)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

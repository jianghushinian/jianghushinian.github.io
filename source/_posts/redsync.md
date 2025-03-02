---
title: '在 Go 中如何使用分布式锁解决并发问题？'
date: 2025-03-02 22:01:22
tags:
- Go
- 并发编程
categories:
- Go
---

在分布式系统中，协调多个服务实例之间的共享资源访问是一个经典的挑战。传统的单机锁（如 `sync.Mutex`）无法实现跨进程工作，此时就需要用到分布式锁了。本文将介绍 Go 语言生态中基于 Redis 实现的分布式锁库 `redsync`，并探讨其使用方法和实现原理。

<!-- more -->

### 分布式锁

首先我们来探讨下为什么需要分布式锁？当我们编写的程序出现资源竞争的时候，就需要使用互斥锁来保证并发安全。而我们的服务很有可能不会单机部署，而是采用多副本的集群部署方案。无论哪种方案运行程序，我们都需要合适的工具来解决并发问题。在解决单个进程间多个协程之间的并发资源抢占问题时，我们往往采用 `sync.Mutex`。而在解决多个进程间的并发资源抢占问题时，就需要采用分布式锁了，这就引出了我们今天要讲解的 `redsync`。

#### 为什么是 `redsync`

在 Go 中分布式锁的开源实现有很多，为什么选择介绍和使用 `redsync` 呢？简单一句话：`redsync` 是 [Redis 官方](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/) 唯一推荐的 Go Redis 分布式锁解决方案，遵循 [Redlock 算法](https://redis.io/topics/distlock)。它允许在多个独立 Redis 节点上创建高可用的锁，适用于需要强一致性的分布式场景。

我们可以对比下 `sync.Mutex` 和 `redsync` 之间的区别，让你有个感性的认识。

| 特性 | sync.Mutex | redsync |
| :---: | :---: | :---: |
| 适用范围 | 单个进程内的多个 goroutine | 多个进程（允许跨机器） |
| 依赖 | 无 | Redis |
| 性能 | 高（无网络开销） | 较低（涉及网络通信） |
| 实现复杂度 | 简单 | 较复杂（需处理网络、超时等问题） |
| 典型场景 | 内存共享资源保护 | 分布式系统共享资源保护 |


二者分别适用于不同的并发场景，选择时需要根据实际需求（单机还是分布式）来决定。

### `redsync` 快速上手

`redsync` 虽然内部实现上比较复杂，但别被吓到，它的用法超级简单。

示例代码如下：

```go
package main

import (
	"context"

	"github.com/go-redsync/redsync/v4"                  // 引入 redsync 库，用于实现基于 Redis 的分布式锁
	"github.com/go-redsync/redsync/v4/redis/goredis/v9" // 引入 redsync 的 goredis 连接池
	goredislib "github.com/redis/go-redis/v9"           // 引入 go-redis 库，用于与 Redis 服务器通信
)

func main() {
	// 创建一个 Redis 客户端
	client := goredislib.NewClient(&goredislib.Options{
		Addr:     "localhost:36379", // Redis 服务器地址
		Password: "nightwatch",
	})

	// 使用 go-redis 客户端创建一个 redsync 连接池
	pool := goredis.NewPool(client)

	// 创建一个 redsync 实例，用于管理分布式锁
	rs := redsync.New(pool)

	// 创建一个名为 "test-redsync" 的互斥锁（Mutex）
	mutex := rs.NewMutex("test-redsync")

	// 创建一个上下文（context），一般用于控制锁的超时和取消
	ctx := context.Background()

	// 获取锁，如果获取失败（例如锁已被其他进程持有），会返回错误
	if err := mutex.LockContext(ctx); err != nil {
		panic(err) // 如果获取锁失败，程序会 panic
	}

	// TODO 执行业务逻辑
	// ...

	// 释放锁，如果释放失败（例如锁已过期或不属于当前进程），会返回错误
	if _, err := mutex.UnlockContext(ctx); err != nil {
		panic(err) // 如果释放锁失败，程序会 panic
	}
}
```

因为 `redsync` 依赖 Redis，所以我们首先需要创建一个 Redis 客户端对象 `client`，调用 `goredis.NewPool(client)` 会基于这个 `client` 创建一个 `redsync` 的连接池，有了这个连接池 `pool` 就可以调用 `redsync.New(pool)` 创建一个 `redsync` 实例来申请分布式锁了。

`redsync` 提供了 `NewMutex` 方法可以创建一个分布式锁，它接收一个 `name` 参数作为锁的名字，这个名字会作为 Redis 中的 `key`。

拿到锁对象 `mutex` 以后，调用 `mutex.LockContext(ctx)` 就可以加锁，加锁后便可以访问竞态资源了，资源访问完成后，调用 `mutex.UnlockContext(ctx)` 便可以释放锁。

可以发现，`redsync` 用法和 `sync.Mutex` 非常相似，核心就是 `Lock/Unlock` 两个操作。redsync 的使用无非多了一步连接 Redis 的过程。

### 配置选项

不知道你有没有想过一个问题，我们在使用 `sync.Mutex` 时，如果某个 gorutine 加锁后不释放掉，那么其他 gorutine 就无法获取锁，而在分布式场景中，如果一个进程获取了 Redis 分布式锁，然后在未释放锁之前进程挂掉了，其他进程要如何获取锁呢，难道要一直等待下去吗？

这里就要引出一个使用分布式锁很重要的问题，那就是一定要设置一个过期时间，这样才能保证即使拿到锁的进程挂掉了，只要锁的过期时间已到，锁也一定会被自动释放掉，只有这样，其他进程才有机会获取锁。

而我们上面的示例中，之所以可以不设置锁的过期时间，原因是 `redsync` 内部设置了默认值。以下是 `redsync` 中 `NewMutex` 方法的源码：

```go
// NewMutex returns a new distributed mutex with given name.
func (r *Redsync) NewMutex(name string, options ...Option) *Mutex {
	m := &Mutex{
		name:   name,
		expiry: 8 * time.Second,
		tries:  32,
		delayFunc: func(tries int) time.Duration {
			return time.Duration(rand.Intn(maxRetryDelayMilliSec-minRetryDelayMilliSec)+minRetryDelayMilliSec) * time.Millisecond
		},
		genValueFunc:  genValue,
		driftFactor:   0.01,
		timeoutFactor: 0.05,
		quorum:        len(r.pools)/2 + 1,
		pools:         r.pools,
	}
	for _, o := range options {
		o.Apply(m)
	}
	if m.shuffle {
		randomPools(m.pools)
	}
	return m
}
```

这里 `Mutex` 对象的第二个字段 `expiry` 就是分布式锁的过期时间，这里默认为设为 8 秒。`tries` 字段是获取锁的重试次数，即尝试获取锁失败 32 次以后，才会返回加锁失败，因为分布式场景下失败是很正常的情况，所以 32 次并不是一个很夸张的值。`delayFunc` 字段是每次失败后重试的间隔时间。其他字段我就不一一讲解了，绝大多数我们都用不到。

根据代码我们很容易想到这几个字段是通过选项模式来设置的。

+ `WithExpiry(time.Duration)`：设置锁的自动过期时间（建议大于业务执行时间）。
+ `WithTries(int)`：设置最大重试次数。
+ `WithRetryDelay(time.Duration)`：设置重试间隔。

使用示例：

```go
mutex := rs.NewMutex("test-redsync",
    redsync.WithExpiry(30*time.Second),
    redsync.WithTries(3),
    redsync.WithRetryDelay(500*time.Millisecond),
)
```

### 看门狗

我们现在知道使用分布式锁一定要设置一个过期时间了，但是这会带来另外一个问题：如果我们的业务代码还没执行完，锁就过期自动释放了，那么此时另外一个进程成功拿到这把锁，也来访问竞态资源，那分布式锁不就失去意义了吗？

这就引出了使用分布式锁的另一个重要问题，锁自动续期。我举一个代码示例，你就懂了：

```go
package main

import (
	"context"
	"log/slog"
	"time"

	"github.com/go-redsync/redsync/v4"                  // 引入 redsync 库，用于实现基于 Redis 的分布式锁
	"github.com/go-redsync/redsync/v4/redis/goredis/v9" // 引入 redsync 的 goredis 连接池
	goredislib "github.com/redis/go-redis/v9"           // 引入 go-redis 库，用于与 Redis 服务器通信
)

func main() {
	// 创建一个 Redis 客户端
	client := goredislib.NewClient(&goredislib.Options{
		Addr:     "localhost:36379", // Redis 服务器地址
		Password: "nightwatch",
	})

	// 使用 go-redis 客户端创建一个 redsync 连接池
	pool := goredis.NewPool(client)

	// 创建一个 redsync 实例，用于管理分布式锁
	rs := redsync.New(pool)

	// 创建一个名为 "test-redsync" 的互斥锁（Mutex）
	mutex := rs.NewMutex("test-redsync", redsync.WithExpiry(5*time.Second))

	// 创建一个上下文（context），一般用于控制锁的超时和取消
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 获取锁，如果获取失败（例如锁已被其他进程持有），会返回错误
	if err := mutex.LockContext(ctx); err != nil {
		panic(err) // 如果获取锁失败，程序会 panic
	}

	// 看门狗，实现锁自动续约
	stopCh := make(chan struct{})
	ticker := time.NewTicker(2 * time.Second) // 每隔 2s 续约一次
	defer ticker.Stop()
	go func() {
		for {
			select {
			case <-ticker.C:
				// 续约，延长锁的过期时间
				if ok, err := mutex.ExtendContext(ctx); !ok || err != nil {
					slog.Error("Failed to extend mutex", "err", err, "status", ok)
				} else {
					slog.Info("Successfully extend mutex")
				}
			case <-stopCh:
				slog.Info("Exiting mutex watchdog")
				return
			}
		}
	}()

	// 执行业务逻辑
	time.Sleep(6 * time.Second)

	// 通知看门狗停止自动续期
	stopCh <- struct{}{}

	// 释放锁，如果释放失败（例如锁已过期或不属于当前进程），会返回错误
	if _, err := mutex.UnlockContext(ctx); err != nil {
		panic(err) // 如果释放锁失败，程序会 panic
	}
}
```

这个示例延续了前文中的示例代码，你需要重点关注的是如下这部分逻辑：

```go
// 看门狗，实现锁自动续约
stopCh := make(chan struct{})
ticker := time.NewTicker(2 * time.Second) // 每隔 2s 续约一次
defer ticker.Stop()
go func() {
    for {
        select {
        case <-ticker.C:
            // 续约，延长锁的过期时间
            if ok, err := mutex.ExtendContext(ctx); !ok || err != nil {
                slog.Error("Failed to extend mutex", "err", err, "status", ok)
            } else {
                slog.Info("Successfully extend mutex")
            }
        case <-stopCh:
            slog.Info("Exiting mutex watchdog")
            return
        }
    }
}()
```

`redsync` 提供了 `mutex.ExtendContext(ctx)` 方法可以延长锁的过期时间。假设我们申请的分布式锁过期时间是 5 秒，而业务代码执行时间是未知的，那么我们在拿到锁以后，可以单独开启一个 goroutine 来定时延长锁的过期时间，当业务代码执行完成以后，主 goroutine 通过 `stopCh <- struct{}{}` 向子 goroutine 发送停止信号，那么子 goroutine 中的 `<-stopCh` case 就会收到通知，子 goroutine 便会退出，也就停止了锁自动续期。

通过为分布式锁设置过期时间，再配合子 goroutine 自动续期的功能，我们就能保证，持有锁的进程挂掉时不会影响其他进程获取锁，并且还能实现业务执行完成后才释放锁。而这个实现分布式锁自动续期的程序，我们通常把它叫做“看门狗”。

我再额外啰嗦一句，关于分布式锁的续期时常和间隔周期的问题，一般来说，续期的时间可以设置为等于过期时间，即锁的过期时间设为 5 秒，那么每次也只续期 5 秒，`redsync` 内部也是这么做的，至于间隔多久续期一次，这个时间肯定是要小于过期时间 5 秒的，通常设为锁过期时间的 1/3 或 1/2 都可以。

### `redsync` 原理

我上面讲解的 `redsync` 用法基本上能覆盖业务开发中的大部分场景了，对于 `redsync` 更多的功能我就不过多介绍了，有了现有的知识，你遇到了问题也可以自己查阅文档学习。
下面我想讲点更有价值的东西，我们自己来实现一个微型的 Redis 分布式锁，以此来加深你对 `redsync` 的理解。

#### 如何实现一个 Redis 分布式锁

要基于 Redis 实现一个最小化的分布式锁，我们可以定义一个结构体 `MiniRedisMutex` 作为锁对象：

```go
type MiniRedisMutex struct {
	name   string        // 会作为分布式锁在 Redis 中的 key
	expiry time.Duration // 锁过期时间
	conn   redis.Cmdable // Redis Client
}
```

它仅包含必要的字段，`name` 是锁的名称，`expiry` 是分布式锁必须要有的过期时间，`conn` 用来存储 Redis 客户端连接。

我们可以定义一个构造函数 `NewMutex` 来创建分布式锁对象：

```go
func NewMutex(name string, expiry time.Duration, conn redis.Cmdable) *MiniRedisMutex {
	return &MiniRedisMutex{name, expiry, conn}
}
```

接下来就要实现加锁和解锁这两个功能。

加锁方法 `Lock` 实现如下：

```go
func (m *MiniRedisMutex) Lock(ctx context.Context, value string) (bool, error) {
	reply, err := m.conn.SetNX(ctx, m.name, value, m.expiry).Result()
	if err != nil {
		return false, err
	}
	return reply, nil
}
```

`Lock` 方法接收两个参数，`ctx` 用来控制取消，`value` 则会作为锁的值。

`Lock` 方法内部逻辑非常简单，直接调用 Redis 的 `SetNX` 命令来排他的设置一个键值对，锁名称 `name` 作为 Redis 的 `key`，锁的值 `value` 作为 Redis 的 `value`，并指定过期时间为 `expiry`，这就是分布式锁的加锁原理。

这里有两个关键点需要你注意：

+ 使用 `SetNX` 命令：这里之所以使用 `SetNX` 命令而不是普通的 `Set` 命令，是因为加锁操作需要**排他性**。我们知道，`SetNX` 命令的全称是 `SET if Not eXists`，即通过 `SetNX` 命令设置键值对时，如果 `key` 不存在，设置其 `value`，若 `key` 已存在，则不执行任何操作。这刚好符合互斥性，是实现分布式互斥锁的关键所在。
+ `value` 唯一性：虽然 `SetNX` 命令能够实现互斥，但是 Redis 的 `value` 还是要保证唯一性。这一点我们接着往下看你就明白了。

释放锁方法 `Unlock` 实现如下：

```go
// 释放锁的 lua 脚本，保证并发安全
var deleteScript = `
	local val = redis.call("GET", KEYS[1])
	if val == ARGV[1] then
		return redis.call("DEL", KEYS[1])
	elseif val == false then
		return -1
	else
		return 0
	end
`

// Unlock 释放锁
func (m *MiniRedisMutex) Unlock(ctx context.Context, value string) (bool, error) {
	// 执行 lua 脚本，Redis 会保证其并发安全
	status, err := m.conn.Eval(ctx, deleteScript, []string{m.name}, value).Result()
	if err != nil {
		return false, err
	}
	if status == int64(-1) {
		return false, ErrLockAlreadyExpired
	}
	return status != int64(0), nil
}
```

在释放锁的逻辑中，我们不是简单的将指定的 Redis 键值对删除即可，而是调用 `m.conn.Eval` 方法执行了一段 lua 脚本的方式来释放锁。

在这段 lua 脚本中，我们先是从 Redis 中获取指定 `key` 为 `m.name` 的键值对，然后判断其 `value` 是否等于 `Unlock` 方法传入的 `value` 参数值，如果相等，则从 Redis 中删除指定的键值对，表示释放锁，否则什么也不做。

之所以要对 `value` 进行判断，是因为我们要保证这把锁是当前进程所持有的锁，而不是其他进程持有的锁。那么以什么为依据来说明这把锁是当前进程持有的呢？这就是我们要保证 `value` 唯一的原因，每个进程在加锁的时候，需要生成一个随机的 `value` 作为自己的锁的标识，那么释放时，就可以通过这个 `value` 来判断是否是自己持有的锁。而这样做的目的，是为了避免一个进程抢到锁后，还在执行业务逻辑时，锁被另外一个进程给释放了。

遗憾的是，这段释放锁的逻辑，Redis 没有提供像 `SetNX` 一样的快捷命令，所以我们只能将其放在 lua 脚本中执行，才能保证并发安全。

至此，一个微型的 Redis 分布式锁的核心功能咱们就讲解完成了。

以下是 `MiniRedisMutex` 分布式锁完整的代码实现：

```go
package miniredislock

import (
	"context"
	"errors"
	"time"

	"github.com/redis/go-redis/v9"
)

var ErrLockAlreadyExpired = errors.New("miniredislock: failed to unlock, lock was already expired")

// MiniRedisMutex 一个微型的 Redis 分布式锁
type MiniRedisMutex struct {
	name   string        // 会作为分布式锁在 Redis 中的 key
	expiry time.Duration // 锁过期时间
	conn   redis.Cmdable // Redis Client
}

// NewMutex 创建 Redis 分布式锁
func NewMutex(name string, expiry time.Duration, conn redis.Cmdable) *MiniRedisMutex {
	return &MiniRedisMutex{name, expiry, conn}
}

// Lock 加锁
func (m *MiniRedisMutex) Lock(ctx context.Context, value string) (bool, error) {
	reply, err := m.conn.SetNX(ctx, m.name, value, m.expiry).Result()
	if err != nil {
		return false, err
	}
	return reply, nil
}

// 释放锁的 lua 脚本，保证并发安全
var deleteScript = `
	local val = redis.call("GET", KEYS[1])
	if val == ARGV[1] then
		return redis.call("DEL", KEYS[1])
	elseif val == false then
		return -1
	else
		return 0
	end
`

// Unlock 释放锁
func (m *MiniRedisMutex) Unlock(ctx context.Context, value string) (bool, error) {
	// 执行 lua 脚本，Redis 会保证其并发安全
	status, err := m.conn.Eval(ctx, deleteScript, []string{m.name}, value).Result()
	if err != nil {
		return false, err
	}
	if status == int64(-1) {
		return false, ErrLockAlreadyExpired
	}
	return status != int64(0), nil
}
```

其实，这段代码的主要逻辑，都是我从 `redsync` 源码中提取出来。所以 `redsync` 其实也是这样实现的，只不过它内部增加了很多可靠性和边缘场景等逻辑代码，最核心的加锁和解锁逻辑是一样的。

#### 微型分布式锁使用

下面我们来写一个示例程序，演示下如何使用这个微型的分布式锁：

```go
package main

import (
	"fmt"
	"time"

	goredislib "github.com/redis/go-redis/v9"
	"golang.org/x/net/context"

	"github.com/jianghushinian/blog-go-example/redsync/miniredislock"
)

func main() {
	// 创建一个 Redis 客户端
	client := goredislib.NewClient(&goredislib.Options{
		Addr:     "localhost:36379", // Redis 服务器地址
		Password: "nightwatch",
	})
	defer client.Close()

	// 创建一个名为 "test-miniredislock" 的互斥锁
	mutex := miniredislock.NewMutex("test-miniredislock", 5*time.Second, client)

	ctx := context.Background()
	// 互斥锁的值应该是一个随机值
	value := "random-string"

	// 获取锁
	_, err := mutex.Lock(ctx, value)
	if err != nil {
		panic(err)
	}

	// 执行业务逻辑
	fmt.Println("do something...")
	time.Sleep(3 * time.Second)

	// 释放自己持有的锁
	_, err = mutex.Unlock(ctx, value)
	if err != nil {
		panic(err)
	}
}
```

这个示例的具体逻辑我就不逐行讲解了，相信你一看便懂。也希望你能够自己在本机上跑起来这段代码，真正用一下分布式锁，以此加深理解。

最后我再留一个作业，你可以尝试一下实现锁的续期方法 `Extend`。

### 总结

分布式锁可以确保分布式系统中并发安全的访问竞态资源，`redsync` 作为 Go 中最流行的 Redis 分布式锁方案，非常值得我们学习和使用。

`redsync` 的用法非常简单，加锁和解锁操作与 `sync.Mutex` 也非常类似，没有太多的学习成本。不过，为了避免持有锁的进程挂掉时，其他进程还有机会获取锁，我们需要实现看门狗的功能。

我还带你从零实现了一个微型的 Redis 分布式锁，希望你不仅会用 redsync 分布式锁，还能理解其原理，这样在自己的业务开发中，如果遇到问题，我们才能更加得心应手。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/redsync) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Distributed Locks with Redis：[https://redis.io/docs/latest/develop/use/patterns/distributed-locks/](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/)
+ redsync Documentation：[https://pkg.go.dev/github.com/go-redsync/redsync](https://pkg.go.dev/github.com/go-redsync/redsync)
+ redsync GitHub 源码：[https://github.com/go-redsync/redsync](https://github.com/go-redsync/redsync)
+ go-redis GitHub 源码：[https://github.com/redis/go-redis](https://github.com/redis/go-redis)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/redsync](https://github.com/jianghushinian/blog-go-example/tree/main/redsync)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

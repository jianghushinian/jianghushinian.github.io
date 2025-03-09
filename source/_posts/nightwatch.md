---
title: '如何基于 Go 语言设计一个简洁优雅的分布式任务系统'
date: 2025-03-09 22:11:21
tags:
- Go
- 分布式
- 架构设计
categories:
- Go
---

在当今云计算与微服务盛行的时代，分布式任务系统已成为支撑大规模业务的核心基础设施。今天就来为大家分享下如何基于 Go 语言从零设计和实现一个架构简洁且扩展性强的分布式任务系统。

<!-- more -->

### 前置概念

本文会设计并实现一个分布式任务系统，这里我们要先明确两个概念。

+ 分布式：在我们将要实现的分布式任务系统中，分布式是指我们的服务可以部署多个副本，这样才能确保服务更加稳定。
+ 任务：这里的任务是指异步任务，可能是定时或需要周期性运行的任务。

有了这两个前置概念，我们再来分析下在 Go 中如何实现分布式和如何处理异步任务。

### 异步任务

在 Go 中，要处理异步任务有多种方式，比如原生支持的 `time.Sleep`、`time.Timer` 或 `time.Ticke`，再比如一些第三方包 `go-co-op/gocron/v2`、`robfig/cron/v3` 或 `bamzi/jobrunner` 等。本项目在调研过后决定采用 `robfig/cron/v3` 包（以下简称 `cron`）来处理异步任务，原因如下：

+ `cron` 是一个非常流行的包，支持标准 `crontab` 表达式（并且可精确到秒），支持时区、任务链等高级功能。
+ 提供秒级精度的任务调度。
+ 轻量级，且<font style="color:rgb(33, 33, 33);">能轻松应对各种复杂的定时任务场景</font>。

对于 `cron` 包的使用，可以参考我的另一篇文章「[在 Go 中使用 cron 执行定时任务](https://mp.weixin.qq.com/s/bCgGnw9QYTTBuoEktTYZcw)」，里面有详细说明。

### 分布式

既然我们的任务系统是分布式的，那么必然要考虑并发安全问题。当多个副本同时读写系统资源时，很容易产生竞太问题。在分布式场景中，解决竞太问题最常用的手段当然是分布式锁。

Go 中的分布式锁解决方案也很多，常见的有基于 etcd、Redis、ZooKeeper 等中间件来实现的，因为 Redis 在系统中更加常用，所以本项目采用基于 Redis 实现分布式锁的解决方案。Go 中有两个比较常用的第三方包 `bsm/redislock` 和 `go-redsync/redsync` 都是基于 Redis 的分布式锁实现。本项目在调研过后决定采用 `go-redsync/redsync` 包（以下简称 `redsync`），原因如下：

+ `redsync` 遵循 Redis 官方推荐的 Redlock 算法，支持多节点，容忍部分节点故障，避免单点问题。
+ 通过多数派机制确保锁的全局唯一性，降低锁冲突风险。
+ `redsync` 是 [Redis 官方](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/) 唯一推荐的 Go Redis 分布式锁解决方案，由 Redis 社区背书，长期维护，可靠性高。

对于 `redsync` 包的使用，可以参考我的另一篇文章「[在 Go 中如何使用分布式锁解决并发问题？](https://mp.weixin.qq.com/s/8HJXRcCoyZqeC_b6l-TD0w)」，里面有详细说明。

### 分布式任务系统

现在我们对分布式任务系统中的分布式和任务都有了明确的认识，并且找到了解决方案。那么接下来就可以设计并实现分布式任务系统了。

#### 功能介绍

我们要实现的分布式任务系统叫 nightwatch，nightwatch 是守夜、值班的意思，那么这套系统的功能也就一目了然了，就是用来 24 小时不停机的执行异步任务的。

nightwatch 要实现的主要功能如下：

现在我们有一个系统，用户可以在 Web 页面通过表单提交一个“任务”到关系型数据库的任务表中。然后 nightwatch 系统会定时的扫描任务表，取出待执行的任务，并根据任务中的配置，到 Kubernetes 中拉起 Job 资源对象，真正的执行任务。此外，nightwatch 还会取出已经开始执行的任务，然后去 Kubernetes 中获取当前任务对应的 Job 实时状态，并回写到数据库中。直到 Kubernetes 中的 Job 执行完成（或失败），nightwatch 会标记 Job 在数据库表中的任务状态为完成（或失败）。当任务状态为完成（或失败），则任务任务终止，nightwatch 不再扫描出这种状态的数据。

系统整体架构如下：

![nightwatch-component](nightwatch-component.jpg)

nightwatch 是系统中一个非常核心的组件，用来控制任务的执行，并同步任务状态。

#### 架构设计

现在我们知道了 nightwatch 的作用，那么就可以设计其实现架构了。

nightwatch 架构设计如下：

![nightwatch-architecture](nightwatch-architecture.jpg)

首先，我们需要思考一个问题，分布式锁应该在何时使用？

在分布式任务系统中，我们有两种方式使用分布式锁来保证并发安全。一种是在执行具体的定时任务时，多个副本之间进行竞争，谁抢到锁，谁就可以执行任务，未抢到锁的副本可以选择性的跳过此次执行周期。另一种是在 nigthwatch 启动时，就开始抢锁，多个副本之间谁抢到锁，谁就去执行任务调度，未抢到锁的副本则进行周期性的尝试抢锁操作，如果当前执行任务调度的副本被终止，那么其他副本就有机会抢到锁，并执行任务调度。

这两种方式个各自有不同的使用场景，第一种方式的优势是能够实现多副本之间的负载均衡，多个副本都在工作，都有可能抢到锁并执行任务，不过这种方式不能严格控制执行任务的间隔时间，比较适合对间隔时间要求不严格的任务。第二种方式实际上只有一个副本在执行任务调度，其他副本是空载状态，是主备设计，这种方式的好处是能够严格控制任务执行的间隔时间。

nigthwatch 采用第二种方式来使用分布式锁保证并发安全。所以在 nigthwatch 的架构设计中，在启动 nigthwatch 时，先将所有的定时任务注册到任务调度器中，接着就会进行抢锁操作，只有抢到锁的副本才能够执行任务调度。未抢到锁，则使用一个循环周期性的尝试抢锁，直到抢锁成功。对于抢到锁的副本，当注册的任务定时策略达到时，任务调度器就会执行任务。架构图中的 task 就是我们要实现的异步任务，也是主要业务逻辑，task 组件会从数据库表中读取任务，然后在 Kubernetes 中启动 Job，并同步数据库和 Kubernetes 资源之间的状态。

#### 目录结构

我们现在已经设计好了 nigthwatch 的架构，可以动手进行开发实现了。

以下是 nigthwatch 项目的目录和文件：

```bash
$ tree nightwatch
nightwatch # 项目目录
├── README.md # README 文件
├── assets # 项目相关的资源目录
│   ├── docker-compose.yaml # 用于启动项目依赖的 MariaDB 和 Redis
│   └── schema.sql # 测试数据 SQL
├── cmd # 项目启动入口
│   └── main.go
├── go.mod
├── go.sum
├── internal # 项目内部包
│   ├── logger.go # 定制日志
│   ├── nightwatch.go # nightwatch 的实现和启动入口
│   └── watcher # 任务接口和实现
│       ├── all # 任务注册入口
│       │   └── all.go
│       ├── config.go # 任务配置
│       ├── task # 任务实现，一个可以定时同步 MariaDB 和 Kubernetes 任务状态的示例程序
│       │   ├── task.go
│       │   └── watcher.go
│       └── watcher.go # 任务接口
└── pkg # 项目公共包
    ├── db # 数据库实例
    │   ├── mysql.go
    │   └── redis.go
    ├── meta # 元信息
    │   └── where.go # MariaDB where 查询条件元信息封装
    ├── model # 任务模型
    │   └── task.go
    ├── store # 数据库操作接口
    │   ├── helper.go
    │   ├── store.go
    │   └── task.go
    └── util # 工具包
        └── reflect
            └── reflect.go

14 directories, 21 files
```

这里主要的目录和文件我都标明了其用途，不必完全记住，你先有个印象，大概知道整个项目的结构。

#### 调用链路

为了便于你理解代码，我画了一张 nigthwatch 项目的调用链路图：

![nightwatch-flow](nightwatch-flow.jpg)

这个调用链路图指明了 nigthwatch 项目中所有目录之间的代码调用关系。根据这张图，可以看出这是一个非常简洁的架构。`cmd` 中的入口函数 `main` 会调用 `internal` 中的 `nigthwatch` 包，`nigthwatch` 是分布式系统实现的关键所在，这里实现了任务的注册和调度，`watcher` 定义了任务的接口，`task` 就是任务的具体实现，`task` 的业务逻辑中会依赖 `store` 层来读写数据库，所以 `store` 会依赖 `model` 和 `db`。

#### 代码实现

接下来就进入到真正的编码阶段了。

首先我们需要为 nigthwatch 项目的业务设计一张任务表，建表 SQL 语句如下：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/assets/schema.sql](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/assets/schema.sql)
>

```sql
CREATE TABLE IF NOT EXISTS `task` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(45) NOT NULL DEFAULT '' COMMENT '任务名称',
  `namespace` varchar(45) NOT NULL DEFAULT '' COMMENT 'k8s namespace 名称',
  `info` TEXT NOT NULL COMMENT '任务 k8s 相关信息',
  `status` varchar(45) NOT NULL DEFAULT '' COMMENT '任务状态',
  `user_id` bigint(20) NOT NULL DEFAULT '0' COMMENT '用户 ID',
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_name_namespace` (`name`, `namespace`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='任务表';
```

为了简化你理解项目的成本，这里仅定义了最小需要字段。

同时我们可以插入两条测试数据，用来后续项目功能的验证：

```sql
INSERT INTO `task` (`id`, `name`, `namespace`, `info`, `status`, `user_id`) VALUES (1, 'demo-task-1', 'default', '{"image":"alpine","command":["sleep"],"args":["60"]}', 'Normal', 1);
INSERT INTO `task` (`id`, `name`, `namespace`, `info`, `status`, `user_id`) VALUES (2, 'demo-task-2', 'demo', '{"image":"busybox","command":["sleep"],"args":["3600"]}', 'Normal', 2);
```

拿 `ID` 为 `1` 的 `task` 数据举例，任务名是 `demo-task-1`，`namespace` 是 `default`，镜像是 `alpine`，执行命令是 `sleep`，命令参数是 `60`，状态为 `Normal` 表示待执行。当 nigthwatch 服务扫描到这条数据时，就会在 Kubernetes 中 `default` 这个 `namespace` 下创建一个 `name` 为 `demo-task-1` 的 Job，其启动镜像为 `alpine`，启动命令为 `sleep 60`，即睡眠 `60` 秒然后退出。

现在有了数据库表和测试数据，我们来看看 nigthwatch 代码是如何实现的。

入口文件 `cmd/main.go` 实现如下：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/cmd/main.go](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/cmd/main.go)
>

```go
package main

import (
	"flag"
	"log/slog"
	"path/filepath"
	"time"

	genericapiserver "k8s.io/apiserver/pkg/server"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"

	"github.com/jianghushinian/blog-go-example/nightwatch/internal"
	"github.com/jianghushinian/blog-go-example/nightwatch/pkg/db"
)

func main() {
	slog.SetLogLoggerLevel(slog.LevelDebug)

	var kubecfg *string
	if home := homedir.HomeDir(); home != "" {
		kubecfg = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "Optional absolute path to kubeconfig")
	} else {
		kubecfg = flag.String("kubeconfig", "", "Absolute path to kubeconfig")
	}

	config, err := clientcmd.BuildConfigFromFlags("", *kubecfg)
	if err != nil {
		slog.Error(err.Error())
		return
	}
	config.QPS = 50
	config.Burst = 100
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		slog.Error(err.Error())
		return
	}

	cfg := nightwatch.Config{
		MySQLOptions: &db.MySQLOptions{
			Host:                  "127.0.0.1:33306",
			Username:              "root",
			Password:              "nightwatch",
			Database:              "nightwatch",
			MaxIdleConnections:    100,
			MaxOpenConnections:    100,
			MaxConnectionLifeTime: time.Duration(10) * time.Second,
		},
		RedisOptions: &db.RedisOptions{
			Addr:         "127.0.0.1:36379",
			Username:     "",
			Password:     "nightwatch",
			Database:     0,
			MaxRetries:   3,
			MinIdleConns: 0,
			DialTimeout:  5 * time.Second,
			ReadTimeout:  3 * time.Second,
			WriteTimeout: 3 * time.Second,
			PoolSize:     10,
		},
		Clientset: clientset,
	}

	nw, err := cfg.New()
	if err != nil {
		slog.Error(err.Error())
		return
	}

	stopCh := genericapiserver.SetupSignalHandler()
	nw.Run(stopCh)
}
```

因为 `main.go` 非常重要，是整个程序的入口，所以我就把完整代码都贴出来了，包括 `import` 部分，这是为了让你对项目文件之间的依赖关系有一个更清晰的认知，后续讲解的其他模块我就只会贴出核心代码。

`main` 函数的核心功能如下：

首先会初始化各种依赖包，初始化 Kubernetes `clientset` 用于后续操作 Job，初始化 MySQL 用于从中读取任务和更新任务状态，初始化 Redis 用于实现分布式锁。接着会使用这些初始化的对象创建一个配置对象 `nightwatch.Config`。然后使用 `cfg.New()` 创建一个 nightwatch 实例对象 `nw`。最后调用 `nw.Run(stopCh)` 启动服务。这里为了做优雅退出，还引用了 Kubernetes `genericapiserver` 优雅退出机制，你可以参考我的文章「[Go 程序如何实现优雅退出？来看看 K8s 是怎么做的——上篇](https://mp.weixin.qq.com/s/UR6Rf1ewthI8qU3A7Szfzg)」、「[Go 程序如何实现优雅退出？来看看 K8s 是怎么做的——下篇](https://mp.weixin.qq.com/s/oztrz9WfCDnyZC6s4b__LQ)」查看其实现原理。

这里涉及到的 Kubernetes `clientset`、MySQL 和 Redis 相关的具体配置细节我就不详细讲解了，咱们还是将主要精力聚焦在 nigthwatch 的主脉络上。

接下来看下 `cfg.New()` 代码实现如下：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/nightwatch.go](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/nightwatch.go)
>

```go
// New 通过配置构造一个 nightWatch 对象
func (c *Config) New() (*nightWatch, error) {
	rdb, err := db.NewRedis(c.RedisOptions)
	if err != nil {
		slog.Error(err.Error(), "Failed to create Redis client")
		return nil, err
	}

	logger := newCronLogger()
	runner := cron.New(
		cron.WithSeconds(),
		cron.WithLogger(logger),
		cron.WithChain(cron.SkipIfStillRunning(logger), cron.Recover(logger)),
	)

	pool := goredis.NewPool(rdb)
	lockOpts := []redsync.Option{
		redsync.WithRetryDelay(50 * time.Microsecond),
		redsync.WithTries(3),
		redsync.WithExpiry(defaultExpiration),
	}
	locker := redsync.New(pool).NewMutex(lockName, lockOpts...)

	cfg, err := c.CreateWatcherConfig()
	if err != nil {
		return nil, err
	}

	nw := &nightWatch{runner: runner, locker: locker, config: cfg}
	if err := nw.addWatchers(); err != nil {
		return nil, err
	}

	return nw, nil
}
```

`*Config.New` 方法会通过配置信息构造一个 `nightWatch` 对象并返回。这里的 `runner` 就是异步任务调度器，使用 `cron` 包实现，用来调度和执行定时任务。并且这个方法内部还实例化了一个 `redsync` 分布式锁对象 `locker`。`nightWatch` 对象就是通过 `runner`、`locker` 和 `cfg` 来构造的。

这里的核心部分是 `addWatchers` 的逻辑，其实现如下：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/nightwatch.go](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/nightwatch.go)
>

```go
// 注册所有 Watcher 实例到 nightWatch
func (nw *nightWatch) addWatchers() error {
	for n, w := range watcher.ListWatchers() {
		if err := w.Init(context.Background(), nw.config); err != nil {
			slog.Error(err.Error(), "Failed to construct watcher", "watcher", n)
			return err
		}

		spec := watcher.Every3Seconds
		if obj, ok := w.(watcher.ISpec); ok {
			spec = obj.Spec()
		}

		if _, err := nw.runner.AddJob(spec, w); err != nil {
			slog.Error(err.Error(), "Failed to add job to the cron", "watcher", n)
			return err
		}
	}

	return nil
}
```

`*nightWatch.addWatchers` 方法用来注册所有 `Watcher` 对象到调度器 `runner` 中。

`Watcher` 是一个接口，定义了异步任务应该实现的方法。`Watcher` 接口定义如下：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/watcher/watcher.go](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/watcher/watcher.go)
>

```go
type Watcher interface {
	Init(ctx context.Context, config *Config) error
	cron.Job
}

type ISpec interface {
	Spec() string
}

var (
	registryLock = new(sync.Mutex)
	registry     = make(map[string]Watcher)
)

func Register(watcher Watcher) {
	registryLock.Lock()
	defer registryLock.Unlock()

	name := reflectutil.StructName(watcher)
	if _, ok := registry[name]; ok {
		panic("duplicate watcher entry: " + name)
	}

	registry[name] = watcher
}

func ListWatchers() map[string]Watcher {
	registryLock.Lock()
	defer registryLock.Unlock()

	return registry
}
```

可以看到，要实现一个异步任务，需要实现 `Init` 方法以及 `cron.Job` 接口。`cron.Job` 接口其实只有一个方法定义如下：

```go
type Job interface {
    Run()
}
```

只要满足 `Watcher` 接口的任务，就可以通过 `Register` 函数注册到 `registry` 中。`ListWatchers` 函数则可以返回注册到 `registry` 中全部任务。而 `ListWatchers` 函数正是在前文讲解的 `*nightWatch.addWatchers` 方法中调用的。

到目前为止，任务如何被注册到 `nightWatch.runner` 的过程我们就串起来了。接下来需要关注的两个点是，调度器 `runner` 是何时启动的，以及是何时调用 `Register` 函数注册任务的。

我们先来看调度器 `runner` 是何时启动的：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/nightwatch.go](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/nightwatch.go)
>

```go
// Run 执行异步任务，此方法会阻塞直到关闭 stopCh
func (nw *nightWatch) Run(stopCh <-chan struct{}) {
	ctx := wait.ContextForChannel(stopCh)

	// 循环加锁，直到加锁成功，再去启动任务
	ticker := time.NewTicker(defaultExpiration + (5 * time.Second))
	defer ticker.Stop()
	for {
		err := nw.locker.LockContext(ctx)
		if err == nil {
			slog.Debug("Successfully acquired lock", "lockName", lockName)
			break
		}
		slog.Debug("Failed to acquire lock", "lockName", lockName, "err", err)
		<-ticker.C
	}

	// 看门狗，实现锁自动续约
	ticker = time.NewTicker(extendExpiration)
	defer ticker.Stop()
	go func() {
		for {
			select {
			case <-ticker.C:
				if ok, err := nw.locker.ExtendContext(ctx); !ok || err != nil {
					slog.Debug("Failed to extend lock", "err", err, "status", ok)
				}
			case <-ctx.Done():
				slog.Debug("Exiting lock watchdog")
				return
			}
		}
	}()

	// 启动定时任务
	nw.runner.Start()
	slog.Info("Successfully started nightwatch server")

	// 阻塞等待退出信号
	<-stopCh

	nw.stop()
}
```

在 `*nightWatch.Run` 方法中，首先会启动一个无限循环，定时执行尝试抢锁操作，直到抢锁成功。这与前文中讲解的 nightwatch 架构设计是一致的。抢到锁后，就可以执行 `nw.runner.Start()` 启动调度器，执行定时任务了。

此外，在 nightwatch 架构图中没有体现的一点是，这里为分布式锁实现了看门狗机制，用来自动续约。关于 `redsync` 分布式锁的自动续约，在我的文章「[在 Go 中如何使用分布式锁解决并发问题？](https://mp.weixin.qq.com/s/8HJXRcCoyZqeC_b6l-TD0w)」中有详细讲解。

而这个 `Run` 方法，就是在 `main` 函数中通过 `nw.Run(stopCh)` 调用的。

我们还剩下一个最后要看的核心逻辑是 `task` 在何时会调用 `Register` 注册到 `registry` 变量中。

还记得前文中讲解的 `Watcher` 接口么，`*taskWatcher` 实现了这个接口：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/watcher/task/watcher.go](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/watcher/task/watcher.go)
>

```go
var _ watcher.Watcher = (*taskWatcher)(nil)

type taskWatcher struct {
	store     store.IStore
	clientset kubernetes.Interface

	wg sync.WaitGroup
}

func (w *taskWatcher) Init(ctx context.Context, config *watcher.Config) error {
	w.store = config.Store
	w.clientset = config.Clientset
	return nil
}

func (w *taskWatcher) Spec() string {
	return "@every 30s"
}

func init() {
	watcher.Register(&taskWatcher{})
}
```

`taskWatcher` 就是 `task` 任务的具体对象，它实现了 `watcher.Watcher` 接口。可以发现，`Register` 函数是在 `init` 函数中调用的，即 `task` 包被导入时实现自动注册。

task 包会在 `nightwatch/internal/watcher/all/all.go` 文件被导入：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/watcher/all/all.go](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/watcher/all/all.go)
>

```go
package all

import (
	// 触发所有 Watcher 的 init 函数进行注册
	_ "github.com/jianghushinian/blog-go-example/nightwatch/internal/watcher/task"
)
```

这里以匿名包的方式导入 `task` 包。如果我们还有其他的任务实现，则同样可以参考 `task` 包的注册方式，在这里以匿名包形式导入，这也是 `all` 包名的由来，可以注册全部的任务。

然后会在 `nightwatch` 中再次以匿名包的方式导入 `all` 包：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/nightwatch.go](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/nightwatch.go)
>

```go
package nightwatch

import (
	...
	// 触发 init 函数
	_ "github.com/jianghushinian/blog-go-example/nightwatch/internal/watcher/all"
)
```

我们可以总结出任务的注册流程是，`nightwatch` 包导入 `all` 包，`all` 包会导入 `task` 包，`task` 包的 `init` 函数执行就会完成注册。所以在入口文件 `main.go` 导入 `nightwatch` 包的时候，就会触发任务的注册。在调用 `nw.Run(stopCh)` 启动服务时，所有的任务已经注册完成了。

`taskWatcher` 对象的核心逻辑当然就是 `Run` 方法了：

> [https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/watcher/task/watcher.go](https://github.com/jianghushinian/blog-go-example/blob/main/nightwatch/internal/watcher/task/watcher.go)
>

```go
// Run 运行 task watcher 任务
func (w *taskWatcher) Run() {
	w.wg.Add(2)

	slog.Debug("Current sync period is start")

	// NOTE: 将 Normal 状态任务在 Kubernetes 中启动
	go func() {
		defer w.wg.Done()
		ctx := context.Background()

		_, tasks, err := w.store.Tasks().List(ctx, meta.WithFilter(map[string]any{
			"status": model.TaskStatusNormal,
		}))
		if err != nil {
			slog.Error(err.Error(), "Failed to list tasks")
			return
		}

		var wg sync.WaitGroup
		wg.Add(len(tasks))
		for _, task := range tasks {
			go func(task *model.Task) {
				defer wg.Done()
				job, err := w.clientset.BatchV1().Jobs(task.Namespace).Create(ctx, toJob(task), metav1.CreateOptions{})
				if err != nil {
					slog.Error(err.Error(), "Failed to create job")
					return
				}

				task.Status = model.TaskStatusPending
				if err := w.store.Tasks().Update(ctx, task); err != nil {
					slog.Error(err.Error(), "Failed to update task status")
					return
				}
				slog.Info("Successfully created job", "namespace", job.Namespace, "name", job.Name)
			}(task)
		}
		wg.Wait()
	}()

	// NOTE: 同步中间状态的任务在 Kubernetes 中的状态到表中
	go func() {
		defer w.wg.Done()
		ctx := context.Background()

		_, tasks, err := w.store.Tasks().List(ctx, meta.WithFilterNot(map[string]any{
			// 排除这几个状态
			"status": []model.TaskStatus{model.TaskStatusNormal, model.TaskStatusSucceeded, model.TaskStatusFailed},
		}))
		if err != nil {
			slog.Error(err.Error(), "Failed to list tasks")
			return
		}

		var wg sync.WaitGroup
		wg.Add(len(tasks))
		for _, task := range tasks {
			go func(task *model.Task) {
				defer wg.Done()
				job, err := w.clientset.BatchV1().Jobs(task.Namespace).Get(ctx, task.Name, metav1.GetOptions{})
				if err != nil {
					slog.Error(err.Error(), "Failed to get task")
					return
				}

				task.Status = toTaskStatus(job)
				if err := w.store.Tasks().Update(ctx, task); err != nil {
					slog.Error(err.Error(), "Failed to update task status")
					return
				}
				slog.Info("Successfully sync job status to task", "namespace", job.Namespace, "name", job.Name, "status", task.Status)
			}(task)
		}
		wg.Wait()
	}()

	w.wg.Wait()
	slog.Debug("Current sync period is complete")
}
```

`Run` 方法就是用来实现每个 `watcher` 对象的业务逻辑。比如这里就实现了 `task` 任务的业务逻辑，它包含两个功能，在 `Run` 方法的上半部分代码中启动了第一个 goroutine 用来实现将 `Normal` 状态任务在 Kubernetes 中启动，下半部分代码中启动了第二个 goroutine 用来实现同步已运行的任务在 Kubernetes 中的 Job 状态到数据库表中。

至此，nightwatch 项目就讲解完成了。我们一起实现了一个架构简洁且扩展性强的分布式任务系统。关于 nightwatch 项目中更多的代码细节你可以跳转到我的 [GitHub 仓库](https://github.com/jianghushinian/blog-go-example/tree/main/nightwatch)中查看。

### 总结

本文带大家从技术选型到架构设计再到代码实现，一步步完成了一个简洁优雅的分布式任务系统。这套系统不仅架构简洁，扩展也非常方便，我们只需要按照 `task` 的套路实现更多的异步任务，都可以非常方便的方式注册到 nightwatch 中。

我们实现的 nightwatch 项目实际上是开源项目 [OneX](https://github.com/onexstack/onex) 项目中 onex-nightwatch 组件的 copy 实现，其架构完全参考 onex-nightwatch 实现。本文介绍的 nightwatch 项目只是 OneX 其中的一个组件，OneX 所有源代码的质量都超级高，并且相当规范，全部都是来自一线大厂的实践经验。

OneX 不仅是一个开源项目，更是一整套 Go + Kubernetes + LLMOps + MLOps 企业级落地方案，这个项目出自孔令飞老师的「[云原生 AI 实战星球](https://konglingfei.com/)」。

OneX 项目技术栈如下：

![onex](onex.png)

OneX 项目完全开源免费，如果你想学习各种最佳实践，非常建议你深入学习这个项目。如果你觉得自己一个人学习容易放弃，或者遇到问题无法解决，想找人一起讨论，也欢迎你加入到孔令飞老师的「[云原生 AI 实战星球](https://konglingfei.com/)」。这里有一起进步的小伙伴，星球是围绕 OneX 打造的，里面会详细讲解 OneX 中用到的各种技术栈，极大缩短自学 OneX 项目的时间。

星球刚刚上线，原价 ¥1599，现在早鸟价 **¥1299**，交个朋友，可以加我微信领取折扣优惠券（**限时限量**），星球值不值你可以自己先研究下 OneX 项目一探虚实。

扫码直达星球：

![onex-zsxq](onex-zsxq.png)

扫码加我微信领取星球大额优惠券（**限时限量**）：

![wechat](wechat.png)

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/nightwatch) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Onex 项目 GitHub 源码：[https://github.com/onexstack/onex](https://github.com/onexstack/onex)
+ onex-nightwatch 组件 GitHub 源码：[https://github.com/onexstack/onex/tree/master/internal/nightwatch](https://github.com/onexstack/onex/tree/master/internal/nightwatch)
+ 在 Go 中使用 cron 执行定时任务：[https://mp.weixin.qq.com/s/bCgGnw9QYTTBuoEktTYZcw](https://mp.weixin.qq.com/s/bCgGnw9QYTTBuoEktTYZcw)
+ 在 Go 中如何使用分布式锁解决并发问题？：[https://mp.weixin.qq.com/s/8HJXRcCoyZqeC_b6l-TD0w](https://mp.weixin.qq.com/s/8HJXRcCoyZqeC_b6l-TD0w)
+ Go 程序如何实现优雅退出？来看看 K8s 是怎么做的——上篇：[https://mp.weixin.qq.com/s/UR6Rf1ewthI8qU3A7Szfzg](https://mp.weixin.qq.com/s/UR6Rf1ewthI8qU3A7Szfzg)
+ Go 程序如何实现优雅退出？来看看 K8s 是怎么做的——下篇：[https://mp.weixin.qq.com/s/oztrz9WfCDnyZC6s4b__LQ](https://mp.weixin.qq.com/s/oztrz9WfCDnyZC6s4b__LQ)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/nightwatch](https://github.com/jianghushinian/blog-go-example/tree/main/nightwatch)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

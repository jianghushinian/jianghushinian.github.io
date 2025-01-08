---
title: '超简单！用 Go 启动 Redis 实例'
date: 2025-01-08 23:05:05
tags:
- Go
categories:
- Go
---

最近写了几篇 Go 并发编程相关的文章，想必有些读者看多了可能会有些厌倦，今天来点轻松的内容，介绍一个可以用来启动 `redis-server` 的开源库 `github.com/stvp/tempredis`。这是一个用 Go 语言开发的包，专门用于创建临时的 Redis 实例，主要用于测试目的。它可以在本地启动一个临时的 Redis 服务实例，在测试结束后自动关闭并清理，帮助开发者避免对实际的 Redis 环境进行操作。

### 主要功能

`tempredis` 包主要功能如下：

1. 快速启动临时 Redis 实例：无需手动安装和配置 Redis，只要系统上已经安装了 Redis 二进制文件（`redis-server`）。
2. 独立的测试环境：每个测试运行一个独立的 Redis 实例，避免了测试之间的相互干扰。
3. 自动清理：测试完成后，临时实例会被关闭，相关文件会被删除。

### 环境准备

首先，主机上当然要安装 Redis 二进制文件和安装 `tempredis` Go 包：

```bash
# mac 电脑直接使用 brew 即可安装 redis-server
# 其他平台可以参考官方文档：https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/
$ brew install redis
# 安装 tempredis Go 包
$ go get -u github.com/stvp/tempredis
```

### 简单介绍
可以在 [tempredis 文档](https://pkg.go.dev/github.com/stvp/tempredis) 中看到其实现的 exported 结构体和方法：

```go
type Config
    func (c Config) Socket() string
type Server
    func Start(config Config) (server *Server, err error)
    func (s *Server) Kill() (err error)
    func (s *Server) Socket() string
    func (s *Server) Stderr() string
    func (s *Server) Stdout() string
    func (s *Server) Term() (err error)
```

`Config` 结构体实际上就是一个 `map[string]string` 对象，用于设置 Redis 配置。它的 `Socket` 方法用于返回 `redis.sock` 文件位置。

`Server` 结构体代表一个 Redis 实例对象，几个方法作用分别如下：

+ `Start` 是一个函数，用于根据给定的配置 `Config` 启动 `Server`。
+ `Kill` 方法用于强制关闭 `redis-server`。
+ `Term` 方法用于优雅关闭 `redis-server`。
+ `Socket` 方法会代理到 `Config.Socket` 方法调用。
+ `Stdout` 方法阻塞等待返回 `redis-server` 执行后的标准输出。
+ `Stderr` 方法阻塞等待返回 `redis-server` 执行后的错误输出。

### 使用示例

我们已经对 `tempredis` 提供的方法有了初步的了解，那么是时候写一个小 demo 练练手了，示例代码如下：

```go
package main

import (
	"fmt"
	"log"
	"time"

	"github.com/go-redis/redis"
	"github.com/stvp/tempredis"
)

func main() {
	// 创建并启动一个临时的 Redis 实例
	server, err := tempredis.Start(tempredis.Config{
		"port": "0", // 自动分配端口
	})
	if err != nil {
		log.Fatalf("Failed to start tempredis: %v", err)
	}

	// 放在 defer 中执行，避免阻塞 main goroutine
	defer func() {
		fmt.Println("====================== stdout ======================")
		fmt.Println(server.Stdout())
		fmt.Println("====================== stderr ======================")
		fmt.Println(server.Stderr())
	}()

	// main 退出时关闭 redis-server
	defer server.Term()

	// 获取 Redis 的地址
	fmt.Println("Redis server is running at", server.Socket())

	// 连接临时的 Redis 实例
	client := redis.NewClient(&redis.Options{
		Network: "unix",
		Addr:    server.Socket(),
	})

	// 使用 Redis 实例
	client.Set("name", "jianghushinian", time.Second)
	val, err := client.Get("name").Result()
	if err != nil {
		fmt.Println("Get redis key error:", err)
		return
	}
	fmt.Println("name:", val)

	time.Sleep(time.Second)

	// 1s 后 name 已经过期
	val, err = client.Get("name").Result()
	if err != nil {
		fmt.Println("Get redis key error:", err)
		return
	}
	fmt.Println("name:", val)
}
```

代码中每个部分我都写了注释说明，不难理解。在启动临时的 Redis 实例后，可以使用任何 Go Redis 客户端进行连接，这里使用的是 `go-redis/redis`。之后就可以正常使用 Redis 的功能了。

执行示例代码，得到输出如下：

```bash
$ go run main.go
Redis server is running at /var/folders/_n/xb14yhv56fq98c_nqcg8skfc0000gp/T/tempredis1387753332/redis.sock
name: jianghushinian
Get redis key error: redis: nil
====================== stdout ======================
31048:C 02 Jan 2025 00:24:23.933 * Reading config from stdin
31048:C 02 Jan 2025 00:24:23.933 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
31048:C 02 Jan 2025 00:24:23.933 * Redis version=7.2.6, bits=64, commit=00000000, modified=0, pid=31048, just started
31048:C 02 Jan 2025 00:24:23.933 * Configuration loaded
31048:M 02 Jan 2025 00:24:23.933 * monotonic clock: POSIX clock_gettime
31048:M 02 Jan 2025 00:24:23.934 * Running mode=standalone, port=0.
31048:M 02 Jan 2025 00:24:23.934 # WARNING: The TCP backlog setting of 511 cannot be enforced because kern.ipc.somaxconn is set to the lower value of 128.
31048:M 02 Jan 2025 00:24:23.934 * Server initialized
31048:M 02 Jan 2025 00:24:23.934 * Loading RDB produced by version 7.2.6
31048:M 02 Jan 2025 00:24:23.934 * RDB age 167 seconds
31048:M 02 Jan 2025 00:24:23.934 * RDB memory usage when created 1.11 Mb
31048:M 02 Jan 2025 00:24:23.934 * Done loading RDB, keys loaded: 0, keys expired: 0.
31048:M 02 Jan 2025 00:24:23.934 * DB loaded from disk: 0.000 seconds
31048:M 02 Jan 2025 00:24:23.934 * Ready to accept connections unix
31048:signal-handler (1735748664) Received SIGTERM scheduling shutdown...
31048:M 02 Jan 2025 00:24:24.944 * User requested shutdown...
31048:M 02 Jan 2025 00:24:24.944 * Saving the final RDB snapshot before exiting.
31048:M 02 Jan 2025 00:24:24.954 * DB saved on disk
31048:M 02 Jan 2025 00:24:24.954 * Removing the unix socket file.
31048:M 02 Jan 2025 00:24:24.954 # Redis is now ready to exit, bye bye...

====================== stderr ======================

```

这个示例很好的演示了 `tempredis` 的功能和用法，剩下唯一没有演示的就是 `Server.Kill` 方法了，你可以自行尝试下。

### 总结

本文介绍了如何使用 `tempredis` 包来启动 Redis 实例，并应用于 Go 项目中。可以发现，`tempredis` 包的功能和用法都非常简单。

之所以介绍这个包，其实是因为这个包出现在 [redsync](https://github.com/go-redsync/redsync/blob/v4.13.0/examples/goredis/main.go) 项目中，这是一个用 Go 语言实现的 Redis 分布式锁，感兴趣的读者可以自行研究下。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/tempredis) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ tempredis GitHub 源码：[https://github.com/stvp/tempredis](https://github.com/stvp/tempredis)
+ tempredis Documentation：[https://pkg.go.dev/github.com/stvp/tempredis](https://pkg.go.dev/github.com/stvp/tempredis)
+ <font style="color:rgb(31, 35, 40);">Distributed mutual exclusion lock using Redis for Go：</font>[https://github.com/go-redsync/redsync](https://github.com/go-redsync/redsync)
+ Install Redis：[https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/tempredis](https://github.com/jianghushinian/blog-go-example/tree/main/tempredis)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

---
title: '使用 Uber automaxprocs 正确设置 Go 程序线程数'
date: 2025-04-13 17:37:56
tags:
- Go
categories:
- Go
---

我们知道 Go 语言没有直接对用户暴露线程的概念，而是通过 goroutine 来控制并发。不过，在 Go 程序启动时，其背后的调度器往往是多线程运行的。在 Go 语言的 GMP 调度模型中，P 决定着同时运行的 goroutine 数，我们可以通过环境变量 `GOMAXPROCS` 或者运行时函数 `runtime.GOMAXPROCS(n)` 来设置 P 的数量。默认情况下 P 的数量等于机器中可用的 CPU 核心数，不过，这也带来了一个潜在的问题，在容器运行环境下，P 的数量可能远超容器限制的 CPU 数量，从而引发性能问题。本文我们一起来看一下 Go 程序在不同运行环境中的表现，以及如何解决潜在问题。

<!-- more -->

首先，介绍下本文用来测试程序的电脑是 MacBook Pro Core i5 双核四线程的 Intel CPU。我将分别在宿主机和容器环境下进行演示。

> NOTE:
>
> 文中所出现的 GOMAXPROCS 和 P 代表的都是一个意思，即表示 GMP 模型中 P 的数量值。
>

### 宿主机环境

我们先来看在宿主机环境下执行 Go 程序，查看 P 数量的表现。

示例代码如下：

> main.go
>

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Printf("GOMAXPROCS = %d\n", runtime.GOMAXPROCS(0))
}
```

`runtime.GOMAXPROCS(n)` 接收一个 `int` 类型的参数，如果参数 `n` 小于等于 0，则返回当前 P 的数量，如果 `n` 大于 0，则设置 P 的数量为 `n` 并返回修改前 P 的数量。

执行示例代码，得到输出如下：

```bash
$ go run main.go             
GOMAXPROCS = 4
```

这个结果符合预期，输出结果等于宿主机上可用的 CPU 核心数。

### 容器环境

我们再来看下前面的示例程序在 Docker 容器环境下的表现。

为了能够在 Docker 容器中执行示例程序，首先我们将程序进行交叉编译，得到 Linux 平台可执行的二进制文件 `gomaxprocs`。

```bash
$ CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o gomaxprocs main.go
```

然后执行如下命令，在 Docker 容器中运行示例 `gomaxprocs`。

```bash
$ docker run --cpus=2 -it \
 -v $(pwd):/app -w /app alpine \
./gomaxprocs               
GOMAXPROCS = 4
```

在执行容器时，我传递了 `--cpus=2` 参数来限制容器内能够使用的 CPU 数量最大为 2。可以看到，`GOMAXPROCS` 输出结果依然为 4。这说明 Go 程序并没有清楚的识别到自己在容器中运行，其 P 的数量依然设置为宿主机的可用 CPU 核心数。这样看来容器并不能限制 Go 程序能够识别的 CPU 核心数，那么这个容器的隔离环境效果也就大大折扣。尽管 Go 程序为 P 设置了一个较大的数值，但容器执行时只会给容器内进程 2 核的 CPU 使用，而过多的 P 数量可能导致频繁的上下文切换，这将会严重影响 Go 程序的性能。

> NOTE:
>
> 使用如下命令可以查看 `docker run` 命令如何限制 CPU。 
>

```bash
$ docker run --help | grep cpu
      --cpu-period int                   Limit CPU CFS (Completely Fair Scheduler) period
      --cpu-quota int                    Limit CPU CFS (Completely Fair Scheduler) quota
      --cpu-rt-period int                Limit CPU real-time period in microseconds
      --cpu-rt-runtime int               Limit CPU real-time runtime in microseconds
  -c, --cpu-shares int                   CPU shares (relative weight)
      --cpus decimal                     Number of CPUs
      --cpuset-cpus string               CPUs in which to allow execution (0-3, 0,1)
      --cpuset-mems string               MEMs in which to allow execution (0-3, 0,1)
```

那么如何解决这个问题呢？现在是时候让 Uber `automaxprocs` 登场了。

### 使用 Uber automaxprocs

Uber 公司是著名的 Go 应用厂商，其内部大量使用 Go 语言来开发程序，并且开源了很多 Go 包，比如流行的 Go 日志处理包 [zap](https://github.com/uber-go/zap)。同时 Uber 也开源了 [automaxprocs](https://github.com/uber-go/automaxprocs) 包用于解决 Go 程序在容器内无法正确识别运行环境中 CPU 核心数的问题。

接下来，我们就使用 `automaxprocs` 来看一下效果。

首先我们通过如下命令来安装 `automaxprocs`。

```bash
$ go get -u go.uber.org/automaxprocs
```

之后就可以使用 `automaxprocs` 了。

使用方式非常简单，只需要以匿名的方式导入 `automaxprocs` 包即可。

> automaxprocs/main.go
>

```go
package main

import (
	"fmt"
	"runtime"

	_ "go.uber.org/automaxprocs"
)

func main() {
	fmt.Printf("GOMAXPROCS = %d\n", runtime.GOMAXPROCS(0))
}
```

我们先在宿主机环境中执行以上示例代码，看一下效果。

```go
$ go run automaxprocs/main.go
2025/04/10 23:37:57 maxprocs: Leaving GOMAXPROCS=4: CPU quota undefined
GOMAXPROCS = 4
```

可以发现输出的 `GOMAXPROCS` 值依然为 4，这没什么问题。

此外，这里还会打印一行日志，其中 `CPU quota undefined` 表示未检测到容器 CPU 配额，说明 `automaxprocs` 识别到 Go 程序未运行在容器环境。

现在，我们再来看下容器环境中 `automaxprocs` 的效果。同样使用交叉编译，得到 Linux 平台可执行的二进制文件 `automaxprocs`。

```bash
$ CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o automaxprocs/automaxprocs automaxprocs/main.go
```

然后执行如下命令，在 Docker 容器中运行示例 `automaxprocs`。

```bash
$ docker run --cpus=2 -it \                                                                       
 -v $(pwd):/app -w /app alpine \
./automaxprocs/automaxprocs
2025/04/10 15:41:01 maxprocs: Updating GOMAXPROCS=2: determined from CPU quota
GOMAXPROCS = 2
```

可以发现，这一次输出的 `GOMAXPROCS` 值为 2，等于我们设置的参数 `--cpus=2`，说明 `automaxprocs` 包生效了。

### Kubernetes 环境

因为在企业中，大多数情况都是以 Kubernetes 的方式来部署 Go 应用，所以我们有必要在 Kubernetes 环境中来验证一下 `automaxprocs` 包的效果。

现在我们的示例程序目录如下：

```bash
$ tree -F .
./
├── Dockerfile
├── automaxprocs/
│   ├── automaxprocs* # Linux 平台下可执行的二进制文件
│   └── main.go
├── deploy/
│   └── pod.yaml
├── go.mod
├── go.sum
├── gomaxprocs* # Linux 平台下可执行的二进制文件
└── main.go

3 directories, 8 files
```

我们将使用 `Dockerfile` 打包一个镜像，然后在 Kubernetes 环境中以 Pod 的方式来运行 Go 程序。

`Dockerfile` 内容如下：

> Dockerfile
>

```dockerfile
FROM alpine:latest
LABEL authors="jianghushinian"

WORKDIR /app

COPY . .
```

Kubernetes 的 Pod Yaml 文件内容如下：

> deploy/pod.yaml
>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-automaxprocs
  namespace: default
spec:
  containers:
    - name: without-automaxprocs
      image: automaxprocs:latest
      imagePullPolicy: IfNotPresent
      command: [ "/app/gomaxprocs" ]
      resources:
        limits:
          cpu: "1" # 对应 docker run 中 --cpus=1
    - name: with-automaxprocs
      image: automaxprocs:latest
      imagePullPolicy: IfNotPresent
      command: [ "/app/automaxprocs/automaxprocs" ]
      resources:
        limits:
          cpu: "1"
  restartPolicy: Never
```

这个 Pod 包含两个容器，`without-automaxprocs` 运行没有使用 `automaxprocs` 包的 Go 程序，`with-automaxprocs` 运行使用了 `automaxprocs` 包的 Go 程序。并且这两个容器都限制了可用的 CPU 核心数为 1。

在 Kubernetes 集群中运行 Pod，输出结果如下：

```bash
# 启动 Pod
$ kubectl apply -f deploy/pod.yaml 
pod/test-automaxprocs created

# 确认 Pod 执行完成
$ kubectl get pod                 
NAME                READY   STATUS      RESTARTS   AGE
test-automaxprocs   0/2     Completed   0          3s

# 查看 without-automaxprocs 容器执行后的输出结果
$ kubectl logs test-automaxprocs -c without-automaxprocs
GOMAXPROCS = 4

# 查看 with-automaxprocs 容器执行后的输出结果
$ kubectl logs test-automaxprocs -c with-automaxprocs 
2025/04/10 16:43:51 maxprocs: Updating GOMAXPROCS=1: determined from CPU quota
GOMAXPROCS = 1
```

以上结果说明 `automaxprocs` 包在 Kubernetes 运行的容器环境中也依然生效。

那么，你是否好奇 `automaxprocs` 包是如何实现的呢？接下来，我将带你深入到 `automaxprocs` 包源码，来探究其实现原理。

### 源码解读

既然我们在使用 `automaxprocs` 包的方式是通过匿名导入，那么很自然的就应该想到，背后是 `automaxprocs` 包的 `init` 函数在起作用。

`automaxprocs` 包的 `init` 函数实现如下：

> [https://github.com/uber-go/automaxprocs/blob/master/automaxprocs.go#L31](https://github.com/uber-go/automaxprocs/blob/master/automaxprocs.go#L31)
>

```go
func init() {
	maxprocs.Set(maxprocs.Logger(log.Printf))
}
```

看来 `automaxprocs` 包是通过 `maxprocs.Set` 函数来识别并设置正确的 P 数量的。其参数 `maxprocs.Logger(log.Printf)` 显然是用来设置日志输出的，这也是为什么我们匿名导入 `automaxprocs` 包后，每次执行程序，都会有日志输出打印 `GOMAXPROCS` 的值的原因。

`maxprocs.Set` 函数实现如下：

> [https://github.com/uber-go/automaxprocs/blob/master/maxprocs/maxprocs.go#L91](https://github.com/uber-go/automaxprocs/blob/master/maxprocs/maxprocs.go#L91)
>

```go
// 自动设置 GOMAXPROCS 以匹配容器 CPU 配额，返回撤销函数和错误
// 仅在 Linux 系统下且有 CPU 配额时生效，其他情况无操作（no-op）
func Set(opts ...Option) (func(), error) {
	// 配置初始化
    cfg := &config{
		procs:          iruntime.CPUQuotaToGOMAXPROCS, // 根据 CPU 配额计算 P 的函数
		roundQuotaFunc: iruntime.DefaultRoundFunc,     // 配额舍入策略，默认向下取整（math.Floor）
		minGOMAXPROCS:  1,                             // P 最小值为 1
	}
	// 应用自定义选项
    for _, o := range opts {
		o.apply(cfg)
	}

	// 初始化默认撤销函数
    undoNoop := func() {
		cfg.log("maxprocs: No GOMAXPROCS change to reset")
	}

	// 如果存在 GOMAXPROCS 环境变量，则说明用户想要手动设置 P 的值，所以这里直接返回，跳过自动设置
    if max, exists := os.LookupEnv(_maxProcsKey); exists {
		cfg.log("maxprocs: Honoring GOMAXPROCS=%q as set in environment", max)
		return undoNoop, nil
	}

    // 根据 CPU 配额计算 P
    // 参数 cfg.minGOMAXPROCS 表示确保返回值大于此值
    // 参数 cfg.roundQuotaFunc 用来设置配额舍入策略
    maxProcs, status, err := cfg.procs(cfg.minGOMAXPROCS, cfg.roundQuotaFunc)
	if err != nil {
		return undoNoop, err
	}

	// 配额未定义时直接返回，跳过自动设置
    // 比如非容器环境下
    if status == iruntime.CPUQuotaUndefined {
		cfg.log("maxprocs: Leaving GOMAXPROCS=%v: CPU quota undefined", currentMaxProcs())
		return undoNoop, nil
	}

	// 获取当前 P 的值，构造撤销函数
    // 调用方可以调用 undo() 来撤销对 P 值的修改
    prev := currentMaxProcs()
	undo := func() {
		cfg.log("maxprocs: Resetting GOMAXPROCS to %v", prev)
		runtime.GOMAXPROCS(prev)
	}

	// 分类处理日志记录
    switch status {
	case iruntime.CPUQuotaMinUsed: // CPU 配额值小于配置的最低值，使用 minGOMAXPROCS
		cfg.log("maxprocs: Updating GOMAXPROCS=%v: using minimum allowed GOMAXPROCS", maxProcs)
	case iruntime.CPUQuotaUsed: // 使用正常计算值
		cfg.log("maxprocs: Updating GOMAXPROCS=%v: determined from CPU quota", maxProcs)
	}

	// 设置计算后的 P 值，并返回撤销函数 undo
    runtime.GOMAXPROCS(maxProcs)
	return undo, nil
}
```

`maxprocs.Set` 函数采用了选项模式，可以自定义一些配置项。默认情况下使用 `iruntime.CPUQuotaToGOMAXPROCS` 函数来根据 CPU 配额计算 P 值。如果用户指定了 `GOMAXPROCS` 环境变量，则跳过自动设置 P 值，否则，使用 `iruntime.CPUQuotaToGOMAXPROCS` 函数进行计算并设置 P 值。最终返回 `undo` 函数，可以用来撤销对 P 值的修改。

其中 `currentMaxProcs` 函数实现如下：

```go
func currentMaxProcs() int {
	return runtime.GOMAXPROCS(0)
}
```

`iruntime.DefaultRoundFunc` 函数实现如下：

```go
func DefaultRoundFunc(v float64) int {
	return int(math.Floor(v))
}
```

接下来我们重点关注下 `iruntime.CPUQuotaToGOMAXPROCS` 函数的实现。

在 `maxprocs.Set` 函数的注释中我有提到，`Set` 函数仅在 Linux 系统下且有 CPU 配额时生效，其他情况无操作（`no-op`）。所以 `CPUQuotaToGOMAXPROCS` 函数在不同的平台必然有不同的实现。

以下是非 Linux 系统中的实现：

> [https://github.com/uber-go/automaxprocs/blob/master/internal/runtime/cpu_quota_unsupported.go](https://github.com/uber-go/automaxprocs/blob/master/internal/runtime/cpu_quota_unsupported.go)
>

```go
//go:build !linux
// +build !linux

package runtime

// CPUQuotaToGOMAXPROCS converts the CPU quota applied to the calling process
// to a valid GOMAXPROCS value. This is Linux-specific and not supported in the
// current OS.
func CPUQuotaToGOMAXPROCS(_ int, _ func(v float64) int) (int, CPUQuotaStatus, error) {
	return -1, CPUQuotaUndefined, nil
}
```

比如我的 Mac 系统中，就会执行这个实现。

而以下是 Linux 系统中的实现：

> [https://github.com/uber-go/automaxprocs/blob/master/internal/runtime/cpu_quota_linux.go#L35](https://github.com/uber-go/automaxprocs/blob/master/internal/runtime/cpu_quota_linux.go#L35)
>

```go
func CPUQuotaToGOMAXPROCS(minValue int, round func(v float64) int) (int, CPUQuotaStatus, error) {
	// 如果传进来的参数 round 为 nil，则设置为默认的向下取整策略
    if round == nil {
		round = DefaultRoundFunc
	}

    // 检测 cgroups 版本
    cgroups, err := _newQueryer()
	if err != nil {
		return -1, CPUQuotaUndefined, err
	}

	// 根据 cgroups 获取 CPU 配额
    quota, defined, err := cgroups.CPUQuota()
	if !defined || err != nil {
		return -1, CPUQuotaUndefined, err
	}

	// 计算并校验配额
    maxProcs := round(quota)
	if minValue > 0 && maxProcs < minValue {
		return minValue, CPUQuotaMinUsed, nil // 使用最低保障值
	}
    // 返回计算结果
	return maxProcs, CPUQuotaUsed, nil
}
```

Linux 系统中的 `CPUQuotaToGOMAXPROCS` 函数实现代码逻辑不多，主要逻辑都封装在 `_newQueryer()` 返回的对象中。

`_newQueryer()` 实现如下：

> [https://github.com/uber-go/automaxprocs/blob/master/internal/runtime/cpu_quota_linux.go#L66](https://github.com/uber-go/automaxprocs/blob/master/internal/runtime/cpu_quota_linux.go#L66)
>

```go
type queryer interface {
	CPUQuota() (float64, bool, error)
}

var (
	_newCgroups2 = cg.NewCGroups2ForCurrentProcess
	_newCgroups  = cg.NewCGroupsForCurrentProcess
	_newQueryer  = newQueryer
)

func newQueryer() (queryer, error) {
	// 这里先直接判断是否为 cgroups v2，如果是，直接返回，否则继续判断是否为 cgroups v1
    cgroups, err := _newCgroups2()
	if err == nil {
		return cgroups, nil
	}
    // 如果不是 cgroups v2，则判断是否为 cgroups v1
	if errors.Is(err, cg.ErrNotV2) {
		return _newCgroups()
	}
	return nil, err
}
```

可以发现，`_newQueryer()`  函数等于 `newQueryer` 函数，而 `newQueryer` 函数返回一个 `queryer` 接口，这个接口仅有一个方法 `CPUQuota` 用来计算 CPU 配额。

`queryer` 接口的实现有两个，分别是 `cg.NewCGroupsForCurrentProcess` 和 `cg.NewCGroups2ForCurrentProcess`。而这二者也正对应的 cgroups v1/v2 两个版本。

所以，代码阅读到这里，其实我们已经几乎探究到了 `automaxprocs` 包的实现原理，就是通过检查当前 Go 程序所在环境的 cgroups v1/v2 版本，来判断是当前环境是否在容器中，并以此来获取并设置正确的 P 值。这也是为什么 `automaxprocs` 包只能支持 Linux 系统的缘故，因为只有 Linux 实现了 cgroups，以此来实现容器功能。

那么接下来其实我们要看的就是 `cg.NewCGroupsForCurrentProcess` 和 `cg.NewCGroups2ForCurrentProcess` 两个函数的具体实现了。

不过，这两个函数代码实现比较多，我就不贴出来了，我把这两个函数各自相关的常量贴出来，你就能大致明白其实现原理了。

`cg.NewCGroupsForCurrentProcess` 函数涉及的部分常量如下：

> [https://github.com/uber-go/automaxprocs/blob/master/internal/cgroups/cgroups.go](https://github.com/uber-go/automaxprocs/blob/master/internal/cgroups/cgroups.go)
>

```go
const (
	// _cgroupFSType is the Linux CGroup file system type used in
	// `/proc/$PID/mountinfo`.
	_cgroupFSType = "cgroup"
	// _cgroupSubsysCPU is the CPU CGroup subsystem.
	_cgroupSubsysCPU = "cpu"
	// _cgroupSubsysCPUAcct is the CPU accounting CGroup subsystem.
	_cgroupSubsysCPUAcct = "cpuacct"
	// _cgroupSubsysCPUSet is the CPUSet CGroup subsystem.
	_cgroupSubsysCPUSet = "cpuset"
	// _cgroupSubsysMemory is the Memory CGroup subsystem.
	_cgroupSubsysMemory = "memory"

	// _cgroupCPUCFSQuotaUsParam is the file name for the CGroup CFS quota
	// parameter.
	_cgroupCPUCFSQuotaUsParam = "cpu.cfs_quota_us"
	// _cgroupCPUCFSPeriodUsParam is the file name for the CGroup CFS period
	// parameter.
	_cgroupCPUCFSPeriodUsParam = "cpu.cfs_period_us"
)

const (
	_procPathCGroup    = "/proc/self/cgroup"
	_procPathMountInfo = "/proc/self/mountinfo"
)
```

在这里，我们可以看到 `cpu.cfs_quota_us` 和 `cpu.cfs_period_us` 两个关键的值。这两个值其实是 Linux 中的两个文件，是 cgroups v1 用来限制 CPU 使用量的，而它们分别代表着 CPU 时间配额（每周期内可使用的 CPU 时间上限）和 CPU 时间周期。

我们可以在使用了 cgroups v1 的机器中用如下命令在容器中来看一下这两个文件中的实际内容：

```bash
$ docker run --cpus=2 -it --rm alpine sh
/ # cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
200000
/ # cat /sys/fs/cgroup/cpu/cpu.cfs_period_us
100000
```

CPU 时间配额和时间周期的值分别为 `200000` 和 `100000`，单位是 us，它的含义是：在每 `100` ms 的时间里，此容器中的进程能够使用 `200` ms 的 CPU 时间。即 CPU 的核心数为 `200000/100000 = 2` 核。

而 `cg.NewCGroups2ForCurrentProcess` 函数涉及的部分常量如下：

> [https://github.com/uber-go/automaxprocs/blob/master/internal/cgroups/cgroups2.go](https://github.com/uber-go/automaxprocs/blob/master/internal/cgroups/cgroups2.go)
>

```go
const (
	// _cgroupv2CPUMax is the file name for the CGroup-V2 CPU max and period
	// parameter.
	_cgroupv2CPUMax = "cpu.max"
	// _cgroupFSType is the Linux CGroup-V2 file system type used in
	// `/proc/$PID/mountinfo`.
	_cgroupv2FSType = "cgroup2"

	_cgroupv2MountPoint = "/sys/fs/cgroup"

	_cgroupV2CPUMaxDefaultPeriod = 100000
	_cgroupV2CPUMaxQuotaMax      = "max"
)
```

这里的关键值是 `cpu.max` 和 `/sys/fs/cgroup`。这是 cgroups v2 用来限制 CPU 使用量的文件。

我们可以在使用了 cgroups v2 的机器中用如下命令在容器中来看一下这个文件中的实际内容：

```bash
$ docker run --cpus=2 -it --rm alpine \
  sh -c "cat /sys/fs/cgroup/cpu.max"
200000 100000
```

不难发现，其实 cgroups v2 就是把 cgroups v1 中 `cpu.cfs_quota_us` 和 `cpu.cfs_period_us` 两个文件的内容，放到了一个文件 `cpu.max` 中，而内容和效果完全一致。

以上，便是 `automaxprocs` 包的实现原理。

### 未来展望

其实，之所以今天想写这个话题，是因为最近 Go 的一个新提案 [issues/73193](https://github.com/golang/go/issues/73193) 中提到了 Go 要在未来版本中加入自动正确设置容器内 Go 程序 P 值的逻辑。即 Go 官方亲自下场，来解决这个历史遗留问题。这个问题由来已久，早在 2019 年的 [issues/33803](https://github.com/golang/go/issues/33803) 中就有人提出，如果 Go 官方要来解决此问题，对我们广大的 Gopher 来说也算是一件好事。

这里我简单总结一下 [issues/73193](https://github.com/golang/go/issues/73193) 提案，感兴趣的读者也可以点击跳转过去查阅原文。

此提案旨在让 Go 运行时自动适配容器环境，根据 CPU 配额、CPU 亲和性和物理机核心数来综合考量，智能的设置 `GOMAXPROCS`。其优化规则如下：

1. 获取 3 个关键参数
    1. 机器的可用逻辑 CPU 核心数
    2. 通过系统调用 `sched_getaffinity()` 得到的可用 CPU 核心数（CPU 亲和性限制）
    3. 如果进程在容器中运行，计算 cgroups v1/v2 中的 CPU 核心数
2. 计算 `GOMAXPROCS` 值，取步骤 1 中最小值

当然，现在还处于提案阶段，后续在新版本的 Go 程序中到底如何实现还需要查看具体代码。先挖个坑，到时候我会再写一篇文章来分析，欢迎先关注我 [https://jianghushinian.cn/about/](https://jianghushinian.cn/about/)。

### 总结

本文讲解了如何使用 Uber `automaxprocs` 包正确设置 `GOMAXPROCS`。

默认情况下 Go 程序无法识别自己是在容器中运行，导致其自动创建的 P 数量等于宿主机中的可用逻辑 CPU 核心数，从而可能引发性能问题。

`automaxprocs` 包就是专门用来解决此问题的，并且用法非常简单，只需要使用匿名导入的方式 `import _ "go.uber.org/automaxprocs"` 一行代码即可搞定。

我还带你一起阅读了 `automaxprocs` 包的核心部分源码，来深入探究其实现原理。通过阅读源码，我们知道 `automaxprocs` 包其实是读取了 cgroups v/v2 中用来配置 CPU 配额的文件，来感知容器内可用 CPU 核数的，以此来实现在容器内自动为 Go 程序设置正确的 `GOMAXPROCS` 值。

并且，我还提到了 Go 官方下场，将在未来的 Go 版本中解决 `GOMAXPROCS` 不正确的问题，敬请期待。

最后，留一个小作业，如果你觉得 `automaxprocs` 包打印的日志不符合你的项目规范，如何自定义日志呢？这个问题就交给你自行去探索了。

```bash
2025/04/10 23:37:57 maxprocs: Leaving GOMAXPROCS=4: CPU quota undefined
```

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/automaxprocs) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ automaxprocs GitHub 源码地址：[https://github.com/uber-go/automaxprocs](https://github.com/uber-go/automaxprocs)
+ Go GOMAXPROCS 提案 issues/73193：[https://github.com/golang/go/issues/73193](https://github.com/golang/go/issues/73193)
+ Go issues/33803：[https://github.com/golang/go/issues/33803](https://github.com/golang/go/issues/33803)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/automaxprocs](https://github.com/jianghushinian/blog-go-example/tree/main/automaxprocs)
+ 本文永久地址：[https://jianghushinian.cn/2025/04/13/automaxprocs/](https://jianghushinian.cn/2025/04/13/automaxprocs/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

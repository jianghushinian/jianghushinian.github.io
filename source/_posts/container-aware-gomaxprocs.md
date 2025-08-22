---
title: 'Go 1.25 终于迎来了容器感知 GOMAXPROCS'
date: 2025-08-22 09:08:11
tags:
- Go
categories:
- Go
---

大家好，我是江湖十年。

2025 年 8 月 12 日 [Go 1.25](https://go.dev/blog/go1.25) 终于正式发布，随之一起带来的重大利好是 Go 1.25 包含新的容器感知 `GOMAXPROCS` 默认值。我曾在《[使用 Uber automaxprocs 正确设置 Go 程序线程数](https://mp.weixin.qq.com/s/5wrYaHXBpuN0WxKAaNNp-A)》一文中讲解过 Go 语言一直存在的默认线程数问题，如今终于被解决了。我本来打算最近写一篇文章介绍一下，不过就在昨天，Go 官方博客抢先一步，专门写了一篇文章《[Container-aware GOMAXPROCS](https://go.dev/blog/container-aware-gomaxprocs)》来专门介绍 Go 1.25 带来的 `GOMAXPROCS` 默认值变化。那我就不再班门弄斧自己写文章了，本文是对官方博客的中文翻译。

<!-- more -->

如下为译文：

> 原文地址：[https://go.dev/blog/container-aware-gomaxprocs](https://go.dev/blog/container-aware-gomaxprocs)
>

_Michael Pratt 和 Carlos Amedee_

_2025 年 8 月 20 日_

Go 1.25 引入了全新的容器感知 GOMAXPROCS 默认设置，为容器化工作负载提供了更合理的默认行为，避免了可能影响尾延迟的限流问题，并提升了 Go 的开箱即用生产就绪性。我们将深入探讨 Go 如何调度 goroutine，调度又是如何与容器级 CPU 控制交互，以及 Go 如何通过感知容器 CPU 控制来提升性能表现。

### GOMAXPROCS

Go 的核心优势之一是通过 goroutine 提供内置的轻量级并发能力。从语义角度看，goroutine 与操作系统线程高度相似，使我们能够编写简单的阻塞式代码。但 goroutine 比线程更轻量，创建和销毁成本更低。

虽然 Go 可以将每个 goroutine 映射到独立的操作系统线程，但 Go 通过运行时调度器实现线程复用，使线程成为可互换资源。任何 Go 管理的线程均可运行任意 goroutine，因此无需为每个新 goroutine 创建新线程，唤醒 goroutine 时也无需唤醒其他线程。

然而，调度器的引入带来了新的问题。例如，我们应使用多少个线程来运行 goroutine？如果存在 1000 个可运行的 goroutine，我们是否应该将其分配到 1000 个不同线程？

这正是 GOMAXPROCS 的作用所在。GOMAXPROCS 从语义层面上定义了 Go 应该使用的“可用并行度”，更具体地说， GOMAXPROCS 是同时用于运行 goroutine 的最大线程数。

因此，如果 GOMAXPROCS=8 并且有 1000 个可运行 goroutine 时，Go 会使用 8 个线程每次运行 8 个 goroutine。goroutine 通常运行短暂时间后会阻塞，此时 Go 会切换至同一线程运行其他 goroutine。对于非阻塞式的 goroutine，Go 会主动抢占，以确保所有 goroutines 均有机会获得执行。

从 Go 1.5 到 Go 1.24 版本， GOMAXPROCS 的值默认为机器上 CPU 核心的总数。请注意，在本文中，“核心”更精确地来说是指“逻辑 CPU”。例如，一台具有 4 个物理 CPU 且支持超线程的机器有 8 个逻辑 CPU。

这种默认值通常能良好匹配硬件“并行能力”，是因为它与硬件本身的并行能力高度契合。也就是说，如果机器有 8 个核心，而 Go 同时运行超过 8 个线程，操作系统将不得不将这些线程多路复用到 8 个核心上，这就像 Go 将 goroutines 多路复用到线程上一样。这额外的调度层并不总是问题，但它是不必要的开销。

### 容器编排

Go 的另一核心优势是容器化部署的便捷性。在 Kubernetes 等容器编排平台中，管理 Go 使用的 CPU 核心数尤为重要。像 Kubernetes 这样的容器编排平台会根据请求的资源，在可用的机器资源中调度容器。在集群资源中尽可能多地打包容器，需要平台能够预测每个调度容器的资源使用情况。我们希望 Go 能够遵守容器编排平台设定的资源利用约束。

让我们以 Kubernetes 为例，探讨 GOMAXPROCS 设置在此背景下的影响。像 Kubernetes 这样的平台提供了限制容器消耗资源的机制。Kubernetes 有 CPU 资源限制的概念，它向底层操作系统发出信号，告知特定容器或一组容器将被分配多少核心资源。设置 CPU 限制相当于创建一个 Linux [control group](https://docs.kernel.org/admin-guide/cgroup-v2.html#cpu) 的 CPU 带宽限制。

在 Go 1.25 之前，Go 对编排平台设置的 CPU 限制是未知的。相反，它会将 GOMAXPROCS 设置为部署机器上的核心数。如果有 CPU 限制，应用程序可能会尝试使用远超限制允许的 CPU。为了防止应用程序超出其限制，Linux 内核将限制（[throttle](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)）应用程序的执行。

限流机制是一种粗暴的机制，用于限制那些可能会超出其 CPU 限制的容器：它会在限制期间完全暂停应用程序的执行。限制期间通常是 100 毫秒，因此与较低 GOMAXPROCS 设置的软调度多路复用效果相比，限制可能会导致显著的尾延迟影响。即使应用程序很少具有大量并行性，Go 运行时执行的任务（如垃圾回收）仍可能引起 CPU 峰值，从而触发限制。

### 新默认设置

我们希望 Go 在可能的情况下提供高效且可靠的默认值，因此，在 Go 1.25 中，我们默认让 GOMAXPROCS 考虑其容器环境。如果一个 Go 进程在一个有 CPU 限制的容器中运行，如果限制小于核心数， GOMAXPROCS 将默认设为 CPU 限制的值。

容器编排系统可能会动态调整容器的 CPU 限制，因此 Go 1.25 也会定期检查 CPU 限制，并在其发生变化时自动调整 GOMAXPROCS 。

这两项默认值仅适用于 GOMAXPROCS 未指定的情况。设置 GOMAXPROCS 环境变量或调用 runtime.GOMAXPROCS 仍然保持之前的行为。 [runtime.GOMAXPROCS](https://pkg.go.dev/runtime#GOMAXPROCS) 文档详细介绍了新行为。

### 模型的略微差异

GOMAXPROCS 和容器 CPU 限制都会限制进程可使用的最大 CPU 量，但它们的模型略有不同。

GOMAXPROCS 是并行性限制。如果 GOMAXPROCS=8，Go 将永远不会同时运行超过 8 个 goroutine。

相比之下，CPU 限制是一个吞吐量限制。也就是说，它们限制在一段时间内使用的总 CPU 时间。默认周期是 100 毫秒。因此，“8 CPU 限制”实际上是每 100 毫秒的时间段内限制 800 毫秒的 CPU 时间。

这个限制可以通过在 100 毫秒的整个周期内连续运行 8 个线程来填满，这相当于 GOMAXPROCS=8 。另一方面，这个限制也可以通过运行 16 个线程，每个线程运行 50 毫秒，然后在另外 50 毫秒内处于空闲或阻塞状态来填满。

换句话说，CPU 限制并没有限制容器可以运行的总 CPU 数量。它只限制总 CPU 时间。

大多数应用程序在 100 毫秒的周期内具有相当一致的 CPU 使用情况，因此新的 GOMAXPROCS 默认值与 CPU 限制非常匹配，而且肯定比总核心数要好！然而，值得注意的是，由于 GOMAXPROCS 防止超出 CPU 限制平均值的短暂线程峰值，特别尖峰的工作负载可能会因为这种变化而看到延迟增加。

此外，由于 CPU 限制是吞吐量限制，它们可以有一个小数部分（例如，2.5 个 CPU）。另一方面， GOMAXPROCS 必须是正整数。因此，Go 必须将限制舍入为有效的 GOMAXPROCS 值。Go 始终向上舍入，以启用使用完整的 CPU 限制。

### CPU 请求（Requests）

Go 的新 GOMAXPROCS 默认值基于容器的 CPU 限制，但容器编排系统也提供了一个“CPU 请求”控制。虽然 CPU 限制指定了容器可能使用的最大 CPU，但 CPU 请求指定了容器始终可用的最小 CPU。

我们通常会创建具有 CPU 请求但没有 CPU 限制的容器，因为这允许容器利用超出 CPU 请求之外的机器 CPU 资源，这些资源原本由于其他容器的负载不足而闲置。不幸的是，这意味着 Go 不能根据 CPU 请求设置 GOMAXPROCS ，这会阻止利用额外的闲置资源。

当机器繁忙时，具有 CPU 请求的容器在超出其请求时仍然会受到限制（[constrained](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)）。超出请求的权重限制比 CPU 限额的硬性周期性限流要“柔和”，但来自高 GOMAXPROCS 的 CPU 峰值仍然会对应用程序行为产生不利影响。

### 是否应该设置 CPU 限制（limit）？

我们已经了解到 GOMAXPROCS 过高导致的问题，并且知道设置容器 CPU 限制可以让 Go 自动设置一个合适的 GOMAXPROCS ，因此一个显而易见的下一步就是思考是否所有容器都应该设置 CPU 限制。

在自动获取合理的 GOMAXPROCS 默认值方面，这或许是条好建议，但在决定是否设置 CPU 限制时，需要考虑许多其他因素，例如通过避免限制来优先利用闲置资源，或通过设置限制来优先保证可预测的低延迟。

当 GOMAXPROCS 显著高于有效 CPU 限制时，GOMAXPROCS 与有效 CPU 限制不匹配的最坏行为就会发生。例如，在一个拥有 128 个核心的机器上运行的小容器只分配到 2 个 CPU。在这些情况下，考虑明确设置 CPU 限制，或者，作为替代方案，明确设置 GOMAXPROCS ，是最有价值的。

### 结论

Go 1.25 通过根据容器 CPU 限制设置 GOMAXPROCS ，为许多容器工作负载提供了更合理的默认行为。这样做可以避免可能影响尾部延迟的限流，提高效率，并通常确保 Go 开箱即用即可满足生产就绪。你只需将 Go 版本设置为 1.25.0 或更高版本即可获得新的默认值。在您的 go.mod 中设置 Go 版本即可。

感谢社区中所有为促成这一成果的长期讨论（[long](https://go.dev/issue/33803) [discussions](https://go.dev/issue/73193)）做出贡献的人，特别是来自 Uber 的 [go.uber.org/automaxprocs](https://pkg.go.dev/go.uber.org/automaxprocs) 维护者的反馈，他们长期以来一直为用户提供类似的行为。

原文完。

希望此文能对你有所启发。

延伸阅读

+ Go 1.25 released：[https://go.dev/blog/go1.25](https://go.dev/blog/go1.25)
+ Go 1.25 Release Notes：[https://go.dev/doc/go1.25](https://go.dev/doc/go1.25)
+ Container-aware GOMAXPROCS 原文：[https://go.dev/blog/container-aware-gomaxprocs](https://go.dev/blog/container-aware-gomaxprocs)
+ 使用 Uber automaxprocs 正确设置 Go 程序线程数：[https://mp.weixin.qq.com/s/5wrYaHXBpuN0WxKAaNNp-A](https://mp.weixin.qq.com/s/5wrYaHXBpuN0WxKAaNNp-A)
+ 本文永久地址：[https://jianghushinian.cn/2025/08/22/container-aware-gomaxprocs](https://jianghushinian.cn/2025/08/22/container-aware-gomaxprocs)

联系我

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

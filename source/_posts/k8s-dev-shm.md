---
title: 'K8s 如何设置容器 /dev/shm 控制共享内存大小'
date: 2024-08-10 16:27:30
tags:
- Kubernetes
categories:
- Kubernetes
---

记一次生产实践过程中，使用 K8s 部署大语言模型训练代码时，Pod 未设置容器 `/dev/shm` 大小而引发异常，以及完整解决过程。

<!-- more -->

### 起因

一天下午，算法同学在组内 OA 群里发了这么一条消息：

> 多卡启用 `vLLM` 框架推理由于 `Pod` 创建时分配的共享内存大小太小导致 `NCCL` 没法利用 `shm` 创建卡间通信，这个问题看下谁那边解决下？

并附上了截图：

![K8s Pod log](k8s-pod-log.jpg)

我的第一反应是在想：算法同学自己用 Docker 运行代码是肯定没有问题的，然后他才会将代码放到 K8s 集群中使用 Pod 来运行。所以这个问题只有在使用 K8s Pod 启动时才会遇到，估计是哪里设置存在差异。

然后经过跟算法同学的简单沟通，发现果然如此。

并且我又根据截图顺藤摸瓜问出了算法同学 `docker run` 命令的参数，其中有一个 `--shm-size` 参数是用来设置容器 `/dev/shm` 共享内存大小的，Docker 默认设置为 `64M`，算法同学将其调整成了 `128M`。但是使用 K8s Pod 运行则没有去设置此参数，看来这很可能就是问题所在。

### 想当然

现在基本已经定位到问题，大概率就是使用 K8s Pod 启动算法代码时未调整容器 `/dev/shm` 大小。

所以我决定先从解决此问题入手。

既然 Docker 启动命令提供了 `--shm-size` 参数来调整容器 `/dev/shm` 共享内存大小，那么我很自然的就想到了 K8s Pod spec 中也会提供相应的参数来对其进行设置。

于是，我使用如下两条命令来查看 K8s Pod spec 的文档，看看有没有相关参数：

```bash
$ kubectl explain pod.spec
$ kubectl explain pod.spec.containers
```

但是，使用这两条命令却查询无果。

然后我又准备去网上查查看。

先去官网搜一下：

![K8s search shm](k8s-shm.png)

仅搜出来 4 个结果？

我顿感不妙，并且看样子感觉这几个结果并没啥用，都不用点进去了。

出师不利，看来我还是太想当然了。

### 顺利解决

于是我决定还是用 Google 搜索下看看吧。

果然，还是 Google 强大，很容易我就在搜索结果的 [stackoverflow](https://stackoverflow.com/questions/46085748/define-size-for-dev-shm-on-container-engine/46434614) 链接中找到了答案：

> Mounting an `emptyDir` to /dev/shm and setting the medium to `Memory` did the trick!

```yaml
spec:
  volumes:
  - name: dshm
    emptyDir:
      medium: Memory
  containers:
  - image: gcr.io/project/image
    volumeMounts:
      - mountPath: /dev/shm
        name: dshm
```

马上，我就从生产环境将 K8s Pod spec 拷贝下来，并在开发环境部署了参考上面方案修改后的 spec 对其进行验证。

果然可以了！

算法代码不再报错，能够正常运行 Pod 并进行模型训练。

看来问题定位对了。

但这还没有完，显然这个方案里没有设置 `emptyDir` 的大小，我还是不够放心。

我就再次使用 `kubectl explain` 查看了 `emptyDir` 的文档：

```bash
$ kubectl explain pod.spec.volumes.emptyDir
KIND:       Pod
VERSION:    v1

FIELD: emptyDir <EmptyDirVolumeSource>

DESCRIPTION:
    emptyDir represents a temporary directory that shares a pod's lifetime. More
    info: https://kubernetes.io/docs/concepts/storage/volumes#emptydir
    Represents an empty directory for a pod. Empty directory volumes support
    ownership management and SELinux relabeling.

FIELDS:
  medium	<string>
    medium represents what type of storage medium should back this directory.
    The default is "" which means to use the node's default medium. Must be an
    empty string (default) or Memory. More info:
    https://kubernetes.io/docs/concepts/storage/volumes#emptydir

  sizeLimit	<Quantity>
    sizeLimit is the total amount of local storage required for this EmptyDir
    volume. The size limit is also applicable for memory medium. The maximum
    usage on memory medium EmptyDir would be the minimum value between the
    SizeLimit specified here and the sum of memory limits of all containers in a
    pod. The default is nil which means that the limit is undefined. More info:
    https://kubernetes.io/docs/concepts/storage/volumes#emptydir
```

这一次很幸运，文档显示可以通过 `sizeLimit` 参数设置 `emptyDir` 容量大小限制。

并且文档最后有提供链接，直接点击进入浏览器查看，跳转到 K8s 官网找到 [emptyDir 配置示例](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#emptydir-配置示例)：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi
```

这里展示了 `sizeLimit` 的用法。这样，我们就可以通过此参数来控制共享内存大小了。

至此，问题算是彻底解决了。

### 复盘

虽然问题被顺利解决了，但其实我还没有使用过 Docker 的 `--shm-size` 参数，也对 `/dev/shm` 不太了解。

既然这次遇到了问题，那咱们就再稍微深入研究下 Docker 的 `--shm-size` 参数以及 `/dev/shm` 是什么，要知其然知其所以然。只是通过 Google、百度解决问题，然后就放一边了，这显然不是我的风格。

#### 再看问题解决过程中的资料

在回顾问题解决的过程中，我又发现实际上 [stackoverflow](https://stackoverflow.com/questions/46085748/define-size-for-dev-shm-on-container-engine/46434614) 这个回答中其实已经有人给出了如何设置 `emptyDir` 大小的方案链接：

![stack overflow](stackoverflow.png)

点击进入这个 [K8s PR](https://github.com/kubernetes/kubernetes/pull/63641)，里面也给出了示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-1
  namespace: default
spec:
  containers:
  - command:
    - sleep
    - "360000"
    image: busybox
    imagePullPolicy: IfNotPresent
    name: busybox
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
      - name: foo
        mountPath: /data/mysql
    resources:
      limits:
        memory: 1229Mi
      requests:
        cpu: 500m
        memory: 1Gi
  volumes:
   - name: foo
     emptyDir:
       sizeLimit: "350Mi"
       medium: "Memory"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  terminationGracePeriodSeconds: 30
```

并且我还发现，这个 PR 实际上还解决了 [issues/63126](https://github.com/kubernetes/kubernetes/issues/63126)。

所以这里可以总结出一点经验，就是搜索到的答案一定要看全，也许这里就藏着对我们有用的东西，即使它很不起眼，多看一眼总没坏处。

#### /dev/shm 到底是什么？

简单来说，`/dev/shm` 是 Linux 系统下的一个目录，是一个 `tmpfs`（即临时文件系统），速度很快，因为它不在磁盘上，而在内存中。

这也就是说我们往 `/dev/shm` 下写东西，其实会直接写入内存。

`shm` 其实是共享内存（Shared Memory）的缩写。

正因为 `/dev/shm` 的特性，所以常被用来进行进程间通信（IPC）。这种共享内存的通信方式，显然比 Socket 和 K8s 中常用的 gRPC 方式进行通信效率更高。

所以很多算法框架就是使用 `/dev/shm` 来进行进程间通信的。

而这也与 [K8s emptydir 文档](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#emptydir) 中的描述相呼应：

> `emptyDir.medium` 字段用来控制 `emptyDir` 卷的存储位置。 默认情况下，`emptyDir` 卷存储在该节点所使用的介质上； 此处的介质可以是磁盘、SSD 或网络存储，这取决于你的环境。 你可以将 `emptyDir.medium` 字段设置为 `"Memory"`， 以告诉 Kubernetes 为你挂载 tmpfs（基于 RAM 的文件系统）。 虽然 tmpfs 速度非常快，但是要注意它与磁盘不同， 并且你所写入的所有文件都会计入容器的内存消耗，受容器内存限制约束。

#### Docker 中的 /dev/shm

我在查询资料过程中，发现了这篇文章 [Shared Memory & Docker](https://datawookie.dev/blog/2021/11/shared-memory-docker/) 写的比较不错，在此分享给大家。

前文讲过 Docker 默认为容器设置的 `/dev/shm` 大小是 `64M`，可以这样验证：

```bash
# 创建容器
$ docker run --rm -it --name ubuntu ubuntu
# 查看默认 `/dev/shm` 大小
$ docker inspect ubuntu | grep -i shm
            "ShmSize": 67108864,
```

可以这样设置容器 `/dev/shm` 大小：

```bash
$ docker run --rm -it --name ubuntu --shm-size=2gb ubuntu
```

文章在「Mounting Host /dev/shm in a Container」部分还介绍了如何通过挂载宿主机的 `/dev/shm` 目录，来完成在宿主机和容器之间共享内存：

```bash
$ docker run --rm -it --name ubuntu -v /dev/shm:/dev/shm ubuntu
```

这里，我们就能够想到，其实在 K8s 中同样可以实现，只需要通过 `hostPath` 将节点主机目录挂载进 Pod 容器中即可。

不过我就不测试了，大概率以后不会用到这种方式。

#### 总结几种解决方案

现在，回过头仔细想想，其实 `/dev/shm` 不过是一个目录而已，而 `--shm-size` 也仅是 `docker run` 提供的一个参数。

既然这样，那么在 K8s 中的解决这个问题的方案也就确定了：

1. 既然 Docker 可以通过参数设置 `/dev/shm` 大小，那么修改 `daemon.json` 配置也能支持。但这方案显然不够 "K8s"，况且现在生产集群基本都在使用 containerd，所以此方案基本不会被选择。

2. 既然是目录，那么就可以通过为容器挂载 `volumeMounts` 的方式来解决。而 K8s `volumes` 既可以挂载 `emptyDir`，也可以挂载持久化存储卷 PV，或者 `hostPath` 等方式。尽管方法很多，显然从性能和 `/dev/shm` 本身特性考虑，`emptyDir` 是最终答案，不过别忘记设置 `medium: Memory`。

这么分析下来，看来 K8s 确实没必要在 Pod spec 中增加类似 `--shm-size` 的参数来支持此功能。

### 总结

此文记录了我在生产实践过程中，遇到的如何为 K8s Pod 设置容器 `/dev/shm` 大小的问题，以及完整解决过程。

`/dev/shm` 是 Linux 系统下的一个目录，是一个 `tmpfs`（即临时文件系统），多用于进程间通信。

在 Docker 中 `docker run` 命令提供了 `--shm-size` 来设置容器的 `/dev/shm` 大小。

在 K8s 中可以使用 `medium: Memory` 类型的 `emptyDir` 卷挂载的方式来设置容器的 `/dev/shm` 大小。

希望此文能对你有所启发。

至此，本文完。

**延伸阅读**

- Define size for /dev/shm on container engine：https://stackoverflow.com/questions/46085748/define-size-for-dev-shm-on-container-engine/46434614
- K8s PR 63641: Add size limit to tmpfs：https://github.com/kubernetes/kubernetes/pull/63641
- K8s issue 63126: emptyDir with medium: Memory ignores sizeLimit：https://github.com/kubernetes/kubernetes/issues/63126
- K8s Documentation emptyDir：https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#emptydir
- Shared Memory & Docker：https://datawookie.dev/blog/2021/11/shared-memory-docker/

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn

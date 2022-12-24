---
title: Kubernetes 本地持久化存储方案 OpenEBS LocalPV 落地实践上——使用篇
date: 2022-11-04 21:35:10
tags:
- Kubernetes
- CSI
- 存储
categories:
- Kubernetes
---

K8s 支持多达 20+ 种类型的持久化存储，如常见的 CephFS 、Glusterfs 等，不过这些大都是分布式存储，随着社区的发展，越来越多的用户期望将 K8s 集群中工作节点上挂载的数据盘利用起来，于是就有了 local 类型持久卷的支持。

我将通过上、下两篇文章介绍 K8s 本地持久化存储方案 OpenEBS LocalPV 落地实践完整过程。本篇为使用篇，着重介绍实践过程，下一篇文章为原理篇，将对 OpenEBS LocalPV 原理进行讲解。

<!-- more -->

我们可以把 local 类型持久卷称作：Local Persistent Volume，简称 LocalPV。LocalPV 所代表的是某个被挂载的本地（工作节点）存储设备，例如磁盘、分区或者目录，所以 LocalPV 并不能像分布式存储一样可靠，但速度极快，这也决定了 LocalPV 使用场景：I/O 敏感度高且能够容忍小概率数据丢失现象。

### K8s 本地存储

K8s 官方文档里有一个使用 LocalPV 的[简单示例](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#local)，由于篇幅所限，我这里就不对其进行演示了，感兴趣的读者可以自行尝试。

#### 特点

这里简单总结下 K8s LocalPV 的特点：

- 只能用作静态创建的持久卷，不支持动态供应，也就是说必须通过手动的方式创建 PV。

- 与 `hostPath` 卷相比，LocalPV 能够以持久和可移植的方式使用，而无需手动将 Pod 调度到节点。系统通过查看 PV 的节点亲和性（`nodeAffinity`）配置，就能了解卷的节点约束。

- 如果想使用存储类来自动绑定 PVC 和 PV，则**必须将 StorageClass 配置成延迟绑定**，示例如下：

```yaml
apiVersion: storage.K8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

其中 `volumeBindingMode: WaitForFirstConsumer` 属性即为延迟卷绑定，它使得调度器在为 PVC 选择一个合适的 PV 时能考虑到所属 Pod 的调度限制。

举个例子，假设我们创建好了两个 PV 分别为 `PV1`、`PV2`，然后创建一个 Pod 并申明一个 PVC 叫 `PVC1` ，`PV1` 和 `PV2` 同时满足 `PVC1` 的要求，但此时存储类并不能马上将 `PVC1` 与 `PV1` 或 `PV2` 中任何一个 PV 进行绑定，而是要考虑 Pod 的调度策略。如果 Pod 指定了节点亲和性必须要部署到 `PV1` 所在节点，则 `PVC1` 就需要跟 `PV1` 进行绑定，而不能与 `PV2` 进行绑定。

可以发现，当 Pod 需要使用 LocalPV 时，PVC 与 PV 的绑定就需要考虑 Pod 的调度情况了，所以 LocalPV 的存储类无法支持立即绑定，只能将绑定时机延迟到 Pod 调度时进行（`WaitForFirstConsumer`）。 

### OpenBES 本地存储

由于 K8s LocalPV 的使用限制无法满足生产需求，所以就需要寻找替代方案，好在社区已经有人实现了更强大的 LocalPV 存储方案——OpenEBS LocalPV。

#### 简介

OpenEBS 官网地址为：https://openebs.io/。

OpenEBS 可以将 K8s 工作节点上的任何可用存储转换为本地或分布式（也称为复制）持久卷。

OpenEBS 最初由 [MayaData](https://mayadata.io/) 构建并捐赠给 CNCF，现在是 [CNCF 沙盒项目](https://www.cncf.io/sandbox-projects/)。

#### 卷类型

OpenEBS 支持两种卷类型：本地卷、复制卷，架构如下：

![openebs](openebs-arch.png)

本地卷能够直接将工作节点上插入的数据盘抽象为持久卷，供当前节点上的 Pod 挂载使用。

而复制卷则相对复杂一些，OpenEBS 使用其内部的引擎为每个复制卷创建一个微服务，在使用时，有状态服务将数据写入 OpenEBS 引擎，引擎将数据同步复制到集群中的多个节点，从而实现了高可用。

#### 本地卷

由于本次 OpenEBS 的落地实践针对于本地持久化存储，故本文将主要介绍 OpenEBS 本地卷的使用，不会对复制卷进行过多讲解，感兴趣的读者可以进入官网查看学习。

OpenEBS 本地卷支持多种类型：Hostpath、Device、LVM、ZFS、Rawfile

每种类型各有特点，都有自己的适用场景，比如相较于 K8s 原生 Hostpath，OpenEBS 的 Hostpath 能够支持将外挂的数据盘目录作为 Hostpath 目录，从而避免 Pod 可能将宿主机目录写满的问题。Device 能够将块设备用于 LocalPV 的使用，速度极快。而借助 LVM 的能力，则可以更灵活的使用 LocalPV，它可以支持 PV 的动态扩缩容操作。

OpenEBS 为其支持的每种类型都实现了一个单独的项目，这里以使用块设备为例，介绍 OpenEBS LocalPV 使用实践，项目地址为：[device-localpv](https://github.com/openebs/device-localpv)，下文中我将以 Device-LocalPV 来指代它。

#### 实践

这里我将通过 5 个步骤来演示 OpenEBS LocalPV 的实践过程，你可以跟着我一步步操作。

##### 第一步，环境准备

这里我们使用 minikube 来搭建 K8s 集群环境，OpenEBS 官方要求 K8s 版本为 1.20+，我实测下来 1.19 版本也是没有问题的，不过建议大家还是尽量优先选用推荐版本。

- 使用 VirtualBox 作为驱动启动两个 minikube 节点：`minikube`、`minikube-m02`（可以参考：https://minikube.sigs.k8s.io/docs/drivers/virtualbox/）

- minikube 节点挂载一块 4GB 磁盘，`minikube-m02` 节点分别挂载一块 4GB、一块 8GB 磁盘

##### 第二步，安装 Device-LocalPV

由于 OpenEBS Device-LocalPV 本身即为云原生而开发的应用，所以安装起来非常简单，只需要一条 `kubectl apply` 命令即可。

```bash
➜ kubectl apply -f https://raw.githubusercontent.com/openebs/device-localpv/develop/deploy/device-operator.yaml
```

执行以上命令后，会得到如下几个相关 Pod：

```bash
➜ kubectl get pod -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   openebs-device-controller-0        2/2     Running   0          2m23s
kube-system   openebs-device-node-4wld7          2/2     Running   0          2m23s
kube-system   openebs-device-node-p2r6m          2/2     Running   0          2m23s
```

确保这几个 Pod 全部处于 `Running` 状态，就表示 Device-LocalPV 安装成功了。如果安装失败，则需要根据 `kubectl describe` 命令的描述信息进行排查。

##### 第三步，准备磁盘

Device-LocalPV 能够直接接管节点上的块设备，有时候节点上可能同时插入多块数据盘，而这些数据盘中，也许某些数据盘我们不想当作 LocalPV 来使用，为了能够区分哪些块设备可以供 Device-LocalPV 来使用，我们需要在对应的块设备上创建一个（~10MiB）Meta 分区，用于存储磁盘标识信息，Meta 分区有如下要求：

- 是块设备的第一个分区（`ID_PART_ENTRY_NUMBER=1`）

- 不能被格式化成任何文件系统

- 不能设置 flags 分区标记

操作命令如下：

```bash
## 在 minikube 节点上执行如下命令
$ sudo parted /dev/sdb mklabel gpt
$ sudo parted /dev/sdb mkpart test-device 1MiB 10MiB

## 在 minikube-m02 节点执行如下命令
$ sudo parted /dev/sdb mklabel gpt
$ sudo parted /dev/sdb mkpart test-device 1MiB 10MiB
$ sudo parted /dev/sdc mklabel gpt
$ sudo parted /dev/sdc mkpart test-device 1MiB 10MiB
```

以上分别对 `minikube`、`minikube-m02` 两个节点上的块设备进行了 Meta 分区的操作，其中 `/dev/sdb`、`/dev/sdc` 就是节点上挂载的块设备名，你需要根据自己实际情况来指定。

其中三块盘的 Meta 分区名都叫 `test-device` 是有意而为之，接下来创建存储类时会用到。

##### 第四步，创建存储类

既然 Device-LocalPV 支持动态供应，那么必然少不了创建存储类的步骤，将以下 yaml 文件保存为 `sc.yaml` 然后通过 `kubectl apply -f sc.yaml` 创建存储类。

```yaml
apiVersion: storage.K8s.io/v1
kind: StorageClass
metadata:
  name: openebs-device-sc
allowVolumeExpansion: false
parameters:
  devname: "test-device"
provisioner: device.csi.openebs.io
volumeBindingMode: WaitForFirstConsumer
```

**注意：**存储类 `parameters` 字段需要指定 `devname`，其值为上面在为块设备分区时指定的分区名称 `test-device`，存储类在创建 PV 的时候正是根据这个分区名的匹配，来找到哪些块设备是供 Device-LocalPV 来使用的。

##### 第五步，创建 StatefulSet 来申请使用 LocalPV

StatefulSet 以及相关资源定义如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: hello
spec:
  selector:
    matchLabels:
      app: hello
  serviceName: "nginx"
  replicas: 2
  template:
    metadata:
      labels:
        app: hello
    spec:
      terminationGracePeriodSeconds: 1
      containers:
        - name: html
          image: busybox
          imagePullPolicy: IfNotPresent
          command:
            - sh
            - -c
            - 'while true; do echo "`date` [`hostname`] Hello from OpenEBS Local PV." >> /mnt/store/index.html; sleep $(($RANDOM % 5 + 300)); done'
          volumeMounts:
            - mountPath: /mnt/store
              name: csi-devicepv
        - name: web
          image: K8s.gcr.io/nginx-slim:0.8
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: csi-devicepv
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: csi-devicepv
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "openebs-device-sc"
        resources:
          requests:
            storage: 1Gi
```

这一步我们创建了一个 Service 和一个名叫 `hello` 的 StatefulSet。我们重点关注 StatefulSet，它启动两个副本，并通过 `volumeClaimTemplates` 来申请 1Gi 大小的 PVC。

通过 `kubectl apply` 命令安装以上文件资源后，可以看到，两个 Pod 分别被调度到了不同节点上：

```bash
# 查看 Pod 调度情况
➜ kubectl get pod -o wide
NAME      READY   STATUS    RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
hello-0   2/2     Running   0          4m13s   10.244.1.3   minikube-m02   <none>           <none>
hello-1   2/2     Running   0          2m42s   10.244.0.3   minikube       <none>           <none>

# 查看 PVC 资源
➜ kubectl get pvc -o wide
NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE     VOLUMEMODE
csi-devicepv-hello-0   Bound    pvc-042661c8-c000-4dde-9950-2b6859d5f273   1Gi        RWO            openebs-device-sc   4m46s   Filesystem
csi-devicepv-hello-1   Bound    pvc-26f92829-e0d4-4520-86da-2d7741cd68c2   1Gi        RWO            openebs-device-sc   3m15s   Filesystem

# 查看 PV 资源
➜ kubectl get pv -o wide
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS        REASON   AGE   VOLUMEMODE
pvc-042661c8-c000-4dde-9950-2b6859d5f273   1Gi        RWO            Delete           Bound    default/csi-devicepv-hello-0   openebs-device-sc            18m   Filesystem
pvc-26f92829-e0d4-4520-86da-2d7741cd68c2   1Gi        RWO            Delete           Bound    default/csi-devicepv-hello-1   openebs-device-sc            17m   Filesystem
```

现在分别登录两台 minikube 主机节点，使用 `fdisk` 命令查看两个节点磁盘使用情况：

![fdisk](fdisk.png)

上图中，左侧为 `minikube` 节点，右侧为 `minikube-m02` 节点，可以看到 PV 创建成功后会在对应节点的块设备上创建与 PV 相同大小的分区来提供 LocalPV 的支持，而这个分区的生命周期管理工作都是由 Device-LocalPV 来帮我们完成的，它会随着 PV 的创建而创建，随着 PV 的删除而销毁。
以上几步，我们就完成了 OpenEBS Device-LocalPV 实践演练。

#### 避坑指南

正常来说，通过上面的实践步骤，你应该能顺利的搭建并使用 OpenEBS Device-LocalPV。但如果你不够幸运，也许会遇到一些奇怪的问题，接下来我就介绍几个我在 OpenEBS Device-LocalPV 落地实践过程所踩到的坑，供你参考避坑。

##### parted 命令版本问题

根据我的经验，`parted` 命令不同版本表现可能不一致。如下所示，使用两个不同版本的 `parted` 工具执行相同命令，来对块设备进行 Meta 分区操作，得到的结果却不一样。

![parted](parted.png)

可以发现，截图中 3.3 版本 `parted` 命令分区会产生 `flag（msftdata）`，下面 3.1 版本的 `parted` 命令分区则没有产生 `flag`。

OpenEBS Device-LocalPV 规定 Meta 分区不能有 `flag`，如果产生 `flag` 则 Device-LocalPV 会将这个块设备忽略，不进行使用。

为了保证执行 parted 命令对磁盘进行分区时，得到预期结果，我们可以总是进入 OpenEBS Daemonset 的容器内部来使用 OpenEBS 提供的 `parted` 命令进行分区。这样就保证了与 Device-LocalPV 工作时内部使用的 `parted` 命令版本一致，不会出现一些意料之外的问题。Daemonset 所对应的 Pod 即为上面的 `openebs-device-node-4wld7`、`openebs-device-node-p2r6m` 两个 Pod，OpenEBS 启动时会在每个节点上启动一个 Daemonset 工作负载，用来操作节点块设备提供 LocalPV 支持（下一篇文章原理篇中将会对其有更详细的介绍）。

##### PV 动态扩容问题

OpenEBS Device-LocalPV 目前还不支持扩容操作，所以在创建存储类时需要指定 `allowVolumeExpansion` 属性值为 `false`，以此来标记这个存储类所创建出来的 PV 不支持动态扩容操作。

##### PV 可能无法创建成功问题

你可能会遇到 PVC 所申请的容量刚好等于块设备剩余容量时，PV 无法创建成功，PVC 一直处于 `Pending` 状态的问题。

假设我们现在只有一个节点，节点上也仅仅只有一个块设备供 Device-LocalPV 使用，其容量为 10Gi。如果我们连续申请 4 个 PVC，其容量依次为 1G、3G、3G、1G，分别对应如下截图（`fdisk` 命令显示结果）中 `sdb2`、`sdb3`、`sdb4`、`sdb5` 4 个分区。

![partion](partion.png)

现在如果删除第一个容量为 3G 的 PVC，Device-LocalPV 则会自动删除 `/dev/sdb3` 这个分区。可是，此时如果我们尝试再次创建一个 3G 的 PVC，这个 PVC 将永远无法创建成功，一直处于 `Pending` 状态。

当我们每创建一个 PVC 时，存储类都会通过 Device-LocalPV 在节点的块设备上帮我们新创建一个和 PVC 中申请的容量相等的一个磁盘分区出来，从截图中可以看出，这个分区的 `Start` 、`End` 是从小到大且连续的。新创建的 3G 的 PVC 容量等于 `/dev/sdb3` 分区容量，理论上应该是可以创建成功的。

分析 Device-LocalPV 源码可以发现，在计算节点上块设备剩余可用容量是否满足 PVC 申请的容量大小时，Device-LocalPV 通过 `if tmp.SizeMiB > partSize` 这条语句来进行判断，其中 `tmp.SizeMiB` 为块设备剩余可用容量，`partSize` 为 PVC 申请容量，只有块设备剩余容量大于 PVC 申请容量时，才会进行分区操作，如果没有满足 PVC 申请容量的可用分区，PVC 就会一直处于 `Pending` 状态。

https://github.com/openebs/device-localpv/blob/develop/pkg/device/device-util.go#L243

![part-size](part-size.png)

由此可见，我们将 `if tmp.SizeMiB > partSize` 改成 `if tmp.SizeMiB >= partSize` 即可解决这个问题。

另外，在阅读源码的过程中，我还发现 Device-LocalPV 在计算块设备可用分区时，对于分区大小的计算会涉及从 `Bytes` 到 `Mib` 的单位转换操作：
https://github.com/openebs/device-localpv/blob/develop/pkg/device/device-util.go#L294

![parse-part-free](parse-part-free.png)

这里的 `beginBytes` 、`endBytes` 对应的就是上面 `fdisk` 命令截图中的 `Start` 、`End`，此时如果分区没有对齐，则会出现浮点数计算精度丢失问题，其中 `beginMib` 会被 `math.Ceil` 向上取整，`endMib` 会被 `math.Floor` 向下取整，最终得到的 `sizeMib` 有可能小于实际剩余可用空间，这样就导致可能会出现磁盘剩余容量满足 PVC 申请容量，而 PVC 却无法创建成功现象，所以在创建 PVC 时应尽量申请 `1024` 整数倍大小的容量。

### 总结

本篇文章首先简单介绍 K8s LocalPV 使用场景及其特点，接着又介绍了 OpenEBS Device-LocalPV 的使用方法以及实践过程中可能遇到的一些坑，如果你也刚好需要落地 LocalPV，希望本文能对你有所帮助。

下一篇文章我将带你一起剖析 OpenEBS Device-LocalPV 底层原理，记得不要错过。

**参考**

https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#local
https://openebs.io/
https://github.com/openebs/device-localpv
https://minikube.sigs.k8s.io/docs/drivers/virtualbox/

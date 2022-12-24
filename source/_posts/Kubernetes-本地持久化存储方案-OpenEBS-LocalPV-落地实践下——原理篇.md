---
title: Kubernetes 本地持久化存储方案 OpenEBS LocalPV 落地实践下——原理篇
date: 2022-11-16 08:24:14
tags:
- Kubernetes
- CSI
- 存储
categories:
- Kubernetes
---

本篇文章我将讲解 OpenEBS Device-LocalPV 实现原理，如果还不了解了 OpenEBS Device-LocalPV 如何使用，可以移步至本系列上篇文章 [Kubernetes 本地持久化存储方案 OpenEBS LocalPV 落地实践上——使用篇](https://jianghushinian.cn/2022/11/04/Kubernetes-%E6%9C%AC%E5%9C%B0%E6%8C%81%E4%B9%85%E5%8C%96%E5%AD%98%E5%82%A8%E6%96%B9%E6%A1%88-OpenEBS-LocalPV-%E8%90%BD%E5%9C%B0%E5%AE%9E%E8%B7%B5%E4%B8%8A%E2%80%94%E2%80%94%E4%BD%BF%E7%94%A8%E7%AF%87/) 进行学习。

<!-- more -->

### CSI

在深入理解 OpenEBS Device-LocalPV 原理之前，我们不得不了解一个新的概念 CSI，它规定了 K8s 存储插件实现方式。

在 K8s 中，如果 K8s 内置的存储功能不满足我们的生产需求，则可以通过一种叫作 CSI 的插件机制来对其进行扩展，而 Device-LocalPV 正是采用这种机制来实现的。

CSI 是 Container Storage Interface 的简称，是 K8s 官方定义的容器存储接口规范，它试图定义一个统一的业界标准，专门用来扩展容器编排系统的存储能力。

#### 基础架构

CSI 基本架构如下：

![CSI](CSI.png)

一个 CSI 插件包含两个主体部分 `External Components` 和 `Custom Components`，其中 `External Components` 由 K8s 官方提供，而 `Custom Components` 则由编写插件的作者来提供。这两个 `Components` 又各自包含 3 个组件，一起协同工作。

#### 工作流程

我先在这里大概描述一下 CSI 插件工作流程，方便稍后分析 Device-LocalPV 的实现原理。

首先，在 CSI 插件启动时，`External Components` 中的 `Driver Registrar` 组件最先开始工作，它通过与 `Custom Components` 中的 `Identity` 组件进行通信，获取到 CSI 插件的基本信息，并将其注册到 kubelet 中。

而 `External Components` 中的 `Provisioner` 组件则通过 Watch 机制，监听了 APIServer 中 PVC 对象的创建，一旦有新的 PVC `被创建，Provisioner` 就会与 `Custom Components` 中的 `Controller` 组件进行通信，让其创建 PV 相关资源。

创建完 PV，下一步就到了 Attach 阶段，而 Attach 操作正对应了 `External Components` 中的 `Attacher` 组件，它同样会与 `Custom Components` 中的 `Controller` 组件进行通信，协同完成 Attach 操作。

而最后一步 Mount 操作，则由 Node 节点上的 kubelet 直接调用 `External Components` 中的 `Node` 组件来完成。至此，Pod 内部应用就可以使用主机节点上挂载的 LocalPV 了。

### Device-LocalPV

以上我们简单的过了一遍 CSI 插件基本流程，接下来我们将通过对 OpenEBS Device-LocalPV 运行流程的分析，更深入的理解使用 CSI 插件机制实现 LocalPV 的原理。

#### 部署

要分析一个程序的执行流程，当然要从程序启动入口开始，而 OpenEBS 是一个面向云原生的应用，那么我们首先要看的，就是项目的部署方式，OpenEBS Device-LocalPV 项目部署 yaml 文件也在其项目的 [git 仓库](https://github.com/openebs/device-localpv/blob/develop/deploy/device-operator.yaml)中。

可以看到，部署文件中最重要的两个资源分别是一个名为 `openebs-device-controller` 的 StatefulSet，和一个名为 `openebs-device-node` 的 DaemonSet。

在 StatefulSet 中启动了两个容器，分别是 K8s 官方提供的 `External Provisioner` 组件，和由 OpenEBS 开发的 `Custom Controller` 组件，这两个组件被放在一个 Pod 中协同工作。

而在 DaemonSet 中同样也启动了两个容器，分别是 K8s 官方提供的 `External Driver Registrar` 组件，和由 OpenEBS 开发的 `Custom Node` 组件。

那么，CSI 机制中的 `External Attacher` 组件去哪里了？实际上 LocalPV 并没有 Attach 操作，创建出来后只需要一步 Mount 操作即可使用，所以也就没有必要部署这个组件了。

你是否好奇 `External` 组件和 `Custom` 对应组件之间如何来进行通信？如果你详细看过我上面提供的 Device-LocalPV 项目部署 yaml 文件中的内容，就不难发现，里面有 `unix:///xxx/csi.sock` 字样，实际上它们之间的通信都是依靠基于 Unix socket 的 `gRPC` 来进行的，这样即实现了组件间的解耦，又能高效进行通信。

#### 组件

知道了 OpenEBS Device-LocalPV 项目都部署了哪些组件，接下来我们通过阅读源码来分析程序执行流程。

无论是 StatefulSet 还是 DaemonSet，Device-LocalPV 程序启动[入口文件](https://github.com/openebs/device-localpv/blob/develop/cmd/main.go)都是同一个，程序启动后，在 `main` 函数里会调用 `run` 函数，`run` 函数定义如下：

```go
func run(config *config.Config) {
        if config.Version == "" {
                config.Version = version.Current()
        }

        klog.Infof("Device Driver Version :- %s - commit :- %s", version.Current(), version.GetGitCommit())
        klog.Infof(
                "DriverName: %s Plugin: %s EndPoint: %s NodeID: %s",
                config.DriverName,
                config.PluginType,
                config.Endpoint,
                config.NodeID,
        )

        if len(config.IgnoreBlockDevicesRegex) > 0 {
                device.DeviceConfiguration.IgnoreBlockDevicesRegex = regexp.MustCompile(config.IgnoreBlockDevicesRegex)
        }

        err := driver.New(config).Run()
        if err != nil {
                log.Fatalln(err)
        }
        os.Exit(0)
}
```

值得注意的是 `err := driver.New(config).Run()` 这句代码，通过 `config` 参数启动了一个 `Driver` 并执行 `Run` 方法，我们可以跟踪到 `New` 函数内部来查看其实现：

```go
func New(config *config.Config) *CSIDriver {
   driver := &CSIDriver{
      config: config,
      cap:    GetVolumeCapabilityAccessModes(),
   }

   switch config.PluginType {
   case "controller":
      driver.cs = NewController(driver)
   case "agent":
      driver.ns = NewNode(driver)
   }

   driver.ids = NewIdentity(driver)
   return driver
}
```

可以发现，`New` 函数内部会通过 `config` 创建一个 `CSIDriver` 对象，这个对象会根据配置参数注册 `Controller` 组件或 `Node` 组件（在 Device-LocalPV 项目中 `agent` 和 `Node` 等价），而这两个组件分别对应的就是 StatefulSet 和 DaemonSet，也就是说，`Controller` 组件会以 StatefulSet 的方式启动，`Node` 组件则会以 DaemonSet 方式启动。

##### Identity 组件

根据上面的源码可以发现，不管启动 `Controller` 或 `Node` 中的哪个组件，`Identity` 组件都会被注册进来（`driver.ids = NewIdentity(driver)`），因为我们将 Device-LocalPV 实现的 CSI 插件分开成两个工作负载来部署，而它们都需要注册给 K8s，`Identity` 组件正是干这件事情的。

`Identity` 代码实现在这里：[`identity.go`](https://github.com/openebs/device-localpv/blob/develop/pkg/driver/identity.go)，这个文件中为 Identity 组件定义了 3 个方法，分别是 `GetPluginInfo` 、`Probe` 、`GetPluginCapabilities`。

- 其中 `GetPluginInfo` 方法返回插件的名称和版本号。

- `Probe` 顾名思义是一个探针程序，K8s 可以根据这个探针检查插件是否正常工作。

- 而 `GetPluginCapabilities` 方法返回当前插件的能力，用告诉 K8s 这个 CSI 插件实现了哪些功能，比如 Device-LocalPV 项目没有实现 Attach 功能，当我们在创建 PVC 时指定了这个 CSI 插件作为存储类的 provisioner 时，K8s 就会自动跳过 Attach 阶段，直接进入 Mount 阶段。

细心的读者可能已经发现每个方法上都有一行 `// This implements csi.ControllerServer` 注释，实际上这些方法的名称都是固定的，已经被 CSI 规范所定义，而编写 CSI 插件的作者只需要按照规范实现对应方法即可。CSI spec 详细定义了每个组件需要实现的方法，其地址为：[`spec.md`](https://github.com/container-storage-interface/spec/blob/master/spec.md)

##### Controller 组件

上面分析 CSI 基本架构的时候，讲到 `External Provisioner` 组件通过 Watch 机制监听 APIServer 中 PVC 对象的创建，一旦有新的 PVC 被创建，`Provisioner` 组件就会与 `Custom Controller` 组件进行通信，让其创建 **PV 相关资源**。这里我说的是创建 PV 相关资源，而不是创建 PV。所谓的 PV 相关资源则是一个 CRD，是编写 CSI 组件的开发人员自定义的一种资源类型，一个 CRD 会与一个 PV 对应，其生命周期也基本相同。这么做的原因是 PV 属于 K8s 内部资源，而 CSI 是一个通用规范，不止适用于 K8s，还适用于任何容器编排系统。所以 CSI 插件应该自己定义一种资源类型，来与 PV 进行对应。

而这个与 PV 相对应的 CRD 叫作 `devicevolumes.local.openebs.io`，它定义在 Device-LocalPV 项目部署文件中，相当于 Device-LocalPV 自己管理的 PV 资源。一个典型的 `devicevolumes.local.openebs.io` 资源定义如下：

```yaml
apiVersion: v1
items:
- apiVersion: local.openebs.io/v1alpha1
  kind: DeviceVolume
  metadata:
    creationTimestamp: "2022-08-01T08:45:51Z"
    finalizers:
    - device.openebs.io/finalizer
    generation: 3
    labels:
      kubernetes.io/nodename: minikube-m02
    name: pvc-8e659633-e052-439a-85eb-30a6d385a12b
    namespace: openebs
    resourceVersion: "1715"
    uid: 540aa699-4f6f-487c-b38b-38f8b414bd7c
  spec:
    capacity: "1073741824"
    devname: device-localpv
    ownerNodeID: minikube
  status:
    state: Ready
kind: List
metadata:
  resourceVersion: ""
```

为了方便对节点块设备进行管理，Device-LocalPV 还定义了一个叫 `devicenodes.local.openebs.io` 的 CRD，这个 CRD 对应的是 K8s 工作节点，我们有几个供 Device-LocalPV 使用的节点，就有几个 `devicenodes.local.openebs.io` 资源，这个 CRD 中记录了当前节点上所有可用块设备。
那么 `devicevolumes.local.openebs.io` 是在什么时候创建的呢？在 `Custom Controller` 组件中有一个关键方法叫 `Createvolume`，正是这个方法负责 PV 相关 CRD 的创建。

`External Provisioner` 组件监听到有新的 PVC 创建，就会执行组件内部的 `Provision` 方法，而在 `Provision` 方法内部，会通过 `gRPC` 的方式调用 `Custom Controller` 组件的 `Createvolume` 方法来创建 CRD，CRD 创建完成后，会由 `Provisioner` 来创建 PV 对象。相关代码实现如下：
> https://github.com/kubernetes-csi/external-provisioner/blob/master/pkg/controller/controller.go#L741

![Controller](controller.png)

##### Agent 组件

最后一个要介绍的 Device-LocalPV 组件是 `Agent`，它实际上就是 CSI 插件中的 `Node` 组件，从这个名称不难猜测，所有在节点宿主机上的操作，都会通过这个组件来完成。

当 `Provisioner` 组件和 `Controller` 组件分别创建好 PV 和 CRD 以后，那么还剩下 2 个步骤，分别是在块设备上创建分区和将分区 Mount 到容器内部，这两个操作正是 `Agent` 组件的职责。

`Agent` 组件其构造函数如下：
> https://github.com/openebs/device-localpv/blob/develop/pkg/driver/agent.go

```go
// NewNode returns a new instance
// of CSI NodeServer
func NewNode(d *CSIDriver) csi.NodeServer {
        var ControllerMutex = sync.RWMutex{}

        // set up signals so we handle the first shutdown signal gracefully
        stopCh := signals.SetupSignalHandler()

        // start the device node resource watcher
        go func() {
                err := devicenode.Start(&ControllerMutex, stopCh)
                if err != nil {
                        klog.Fatalf("Failed to start Device node controller: %s", err.Error())
                }
        }()

        // start the device volume  watcher
        go func() {
                err := volume.Start(&ControllerMutex, stopCh)
                if err != nil {
                        klog.Fatalf("Failed to start Device volume management controller: %s", err.Error())
                }
        }()

        if d.config.ListenAddress != "" {
                exposeMetrics(d.config, stopCh)
        }

        return &node{
                driver: d,
        }
}
```

可以发现在 `Agent` 组件内部，启动了两个 Goroutine，它们分别用来监听 `devicenodes.local.openebs.io` 和 `devicevolumes.local.openebs.io` 这两个 CRD，其内部都实现了 `syncHandler` 方法，根据 CRD 状态进行相应操作。

volume 的 `syncHandler` 主要逻辑如下：
> https://github.com/openebs/device-localpv/blob/develop/pkg/mgmt/volume/volume.go#L39

```go
func (c *VolController) syncHandler(key string) error {
   ...
   // Get the Vol resource with this namespace/name
   Vol, err := c.VolLister.DeviceVolumes(namespace).Get(name)
   if K8serror.IsNotFound(err) {
      runtime.HandleError(fmt.Errorf("devicevolume '%s' has been deleted", key))
      return nil
   }
   if err != nil {
      return err
   }
   VolCopy := Vol.DeepCopy()
   err = c.syncVol(VolCopy)
   return err
}

func (c *VolController) syncVol(vol *apis.DeviceVolume) error {
   ...
   // if the status Pending means we will try to create the volume
   if vol.Status.State == device.DeviceStatusPending {
      err = device.CreateVolume(vol)
      if err == nil {
         err = device.UpdateVolInfo(vol, device.DeviceStatusReady)
      } else if custError, ok := err.(*apis.VolumeError); ok && custError.Code == apis.InsufficientCapacity {
         vol.Status.Error = custError
         return device.UpdateVolInfo(vol, device.DeviceStatusFailed)
      }
   }
   return err
}
```

可以看到，`syncHandler` 方法会将查询到的 `devicevolumes.local.openebs.io` 信息传递给 `syncVol` 方法，而这个方法中有一行非常关键的代码 `device.CreateVolume(vol)`，调用此方法的作用正是根据 CRD 的信息，在块设备上创建出真正的分区。

分区一旦被成功创建，那么就只剩下最后一个步骤 Mount 操作了，Mount 操作由 PV 所在节点的 kubelet 直接调用 `Agent` 组件的 `NodePublishVolume` 方法来完成。值得注意的是，在部署 Device-LocalPV 项目的 yaml 文件中，Agent 组件所在容器的 `volumeMounts` 属性中有一个 `mountPropagation: "Bidirectional"` 配置，其作用是为了使在容器内部执行的 `Mount` `命令能够向上传播到宿主机上。所以尽管Agent` 组件运行在容器中，但在其内部执行的 `Mount` 命令依然能够在节点上生效。

至此，整个 OpenEBS Device-LocalPV 项目的基本原理，我们就一起分析完成了。细心的你应该能够发现，CSI 插件其实就是一个  CRD + Custom Controller 的实践。

#### 调度策略

接下来我还想跟你分享下 Device-LocalPV 的调度原理。

Device-LocalPV 提供了两种调度策略：`CapacityWeighted`、`VolumeWeighted`，其相关源码实现在：[`schd_helper.go`](https://github.com/openebs/device-localpv/blob/develop/pkg/driver/schd_helper.go)，我们可以在存储类中通过参数指定调度策略：

```yaml
parameters:
 scheduler: "VolumeWeighted"
 devname: "test-device"
```

##### CapacityWeighted

`CapacityWeighted` 为默认调度策略，即根据使用容量调度，它会查找已部署了 OpenEBS 的节点，按节点上块设备已使用容量进行打分，优先调度到已使用容量较小的节点。

下图中，Node1 节点上已经存在 3 个 PV，Node2 节点上存在 2 个 PV，Node3 节点上没有部署 OpenEBS，如果此时新创建一个 PVC，那么 PV 会如何调度呢？

![CapacityWeighted](CapacityWeighted.png)

根据 `CapacityWeighted` 调度策略来看，Node3 节点第一个被排除，尽管 Node1 节点上已经存在的 PV 数量比 Node2 节点上的多，但已使用容量较少，故根据已使用容量来排序，显然这个新创建的 PV 将会被调度到 Node1 节点。

##### VolumeWeighted

`VolumeWeighted` 调度策略则根据使用卷数量来进行调度，查找已部署了 OpenEBS 的节点，按节点上块设备已分配卷数量进行打分，优先调度到已使用卷数量较小的节点。

那么分析下来，在跟上图中同样的情况下，使用 `VolumeWeighted` 调度策略后，新创建的 PV 将会被调度到 Node2 节点。

![VolumeWeighted](VolumeWeighted.png)

##### 自定义调度策略

如果事情按照理想化方向发展，那么上面两种 Device-LocalPV 提供的调度策略没有任何问题，但真实场景中，也许你会遇到如下情况：

![FreeCapacityWeighted](FreeCapacityWeighted.png)

现在部署了 OpenEBS 的节点从 2 个扩展成 3 个，但是 Node3 上仅仅部署了 OpenEBS，还没有插入块设备供 OpenEBS  使用。
在这种情况下，如果新创建 PVC，那么这个 PVC 将一直处于 `Pending` 状态，PV 无法完成创建。

因为无论是哪种调度策略，Device-LocalPV 在为节点打分阶段，总是会将使块设备用量为 0 的节点排序在最前面，所以两种策略最终都会将 PV 调度到 Node3 节点，而又因为 Node3 节点上没有供 OpenEBS  使用的块设备，无法进行磁盘分区，PV 也就无法成功创建。

这也是 Device-LocalPV 支持的调度策略存在问题的地方，如果想解决这个问题，我们可以参考官方提供的调度策略，实现自己的调度策略。我实现了一个 `FreeCapacityWeighted` 策略，根据节点上块设备剩余容量大小来调度 PV，这样就能够保证 Node3 这种情况在节点打分阶段使其排序在最后面。如果你感兴趣可以进入到这个 [`issue`](https://github.com/openebs/device-localpv/issues/61) 查看。

#### 故障处理

如果你想在生产环境中使用 OpenEBS Device-LocalPV 来提供 LocalPV 的支持，那么就要考虑 LocalPV 的故障处理，根据 LocalPV 特点，我这里主要分析下如下两种情况的故障处理思路。

##### 节点故障

如果某个正在被 LocalPV 使用的节点出现故障，可以通过迁移块设备来让 LocalPV 恢复使用，具体步骤如下：

1. 将数据盘移动到新节点上

2.  如果新节点没有部署 OpenEBS 则需要先部署 Device-LocalPV 的 DaemonSet 到新节点上

3. 修改 CRD 资源 `devicevolumes.local.openebs.io` 所属的节点，即修改 `spec.ownerNodeID` 属性到新的节点

4. 修改 PV 的节点信息，即修改 `spec.nodeAffinity` 属性到新的节点

5. 删除使用 PV 的 Pod 让其自动重启，并调度到 PV 所指定的新节点上

##### 磁盘故障

如果某个正在被 LocalPV 使用的块设备出现故障，则会造成数据丢失。

需要注意的是，当某个 PV 所使用的磁盘分区出现故障时，PV 无法感知，只有部署在 Pod 内的程序可以感知到，可以尝试在出现故障的 Pod 容器内部执行读写操作，会得到如下错误：

```bash
/mnt/store # echo abc > a.txt
sh: can't create a.txt: Input/output error
```

所以使用 LocalPV 的程序要有比较完善的异常处理机制以应对可能出现的故障问题。

### 总结

本篇文章我们一起学习了 CSI 的概念并对 OpenEBS Device-LocalPV 原理进行了剖析。其中 CSI 不仅可以用来实现 LocalPV，其他任何存储扩展都可以用它的规范来实现。通过分析 Device-LocalPV  源码，我们知道它其实是一个  CRD + Custom Controller 的实践。

由于篇幅所限，文中不能将所有细节都涵盖，如果有问题，欢迎一起交流学习。

参考：

https://openebs.io/
https://github.com/container-storage-interface/spec
https://github.com/openebs/device-localpv

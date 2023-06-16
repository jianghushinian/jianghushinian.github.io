---
title: 一文读懂 Kubernetes 存储设计
date: 2023-06-16 10:36:33
tags:
- Kubernetes
- 存储
categories:
- Kubernetes
---

在 docker 的设计中，容器内的文件是临时存放的，当容器被删除后，容器内部的数据将会一同被清空。不过，我们可以通过在 `docker run` 启动容器时，使用 `--volume/-v` 参数来指定挂载卷，这样就能够将容器内部的路径挂载到主机，当我们在容器内部存放数据时会被同步到被挂载的主机路径中，容器删除后，保存到主机路径中的数据仍然存在。

docker 通过挂载卷的方式解决了持久化存储的问题，但 K8s 存储要面临的问题比这要复杂的多，K8s 通常会在多个主机部署节点，如果由 K8s 编排的 docker 容器崩溃，K8s 可能会在其他节点上重新拉起容器，原来节点主机上挂载的容器目录就无法使用了。K8s 为了解决容器存储的诸多限制，对存储资源做了一层抽象，叫作卷（Volume）。

### Kubernetes 支持的卷类型

K8s 支持的卷基本上可以分为三类：配置信息、临时存储、持久存储。接下来我将一一介绍。

### 配置信息

首先我们来关注下配置信息，我们常见的应用不管是哪种类型，一般都会用到配置文件或启动参数，K8s 将配置信息进行了抽象，定义成了几种资源，常用的配置信息主要包含以下三种：

- ConfigMap
- Secret
- DownwardAPI

#### ConfigMap

ConfigMap 卷主要用来保存应用的配置数据，以一个或多个 `key: value` 形式存在，其中 `value` 可以是字面量或配置文件。

ConfigMap 在设计上不是用来保存大量数据的，在 ConfigMap 中保存的数据不可超过 `1MiB`（兆字节）。

##### 创建示例

- 通过命令行创建

在创建 ConfigMap 的时候可以通过 `--from-literal` 参数来指定 `key: value`，以下示例中 `foo=bar` 即为字面量形式，`bar=bar.txt` 为配置文件形式。

```bash
$ kubectl create configmap c1 --from-literal=foo=bar --from-literal=bar=bar.txt
```

`bar.txt` 内容如下：

```
baz
```

通过 `kubectl describe` 命令查看新创建的名称为 `c1` 的这个 `ConfigMap` 资源内容。

```bash
$ kubectl describe configmap c1
Name:         c1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
bar:
----
baz
foo:
----
bar
Events:  <none>
```

根据输出结果可以看到数据被保存到 `Data` 属性中，每个 `key: value` 之间通过 `----` 进行分隔。

- 通过 Yaml 文件创建

创建 `configmap-demo.yaml` 内容如下：

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: c2
  namespace: default
data:
  foo: bar
  bar: baz
```

通过 `kubectl apply` 命令应用这个文件。

```bash
$ kubectl apply -f configmap-demo.yaml

$ kubectl get configmap c2
NAME   DATA   AGE
c2     2      11s

$ kubectl describe configmap c2
Name:         c2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
foo:
----
bar
bar:
----
baz
Events:  <none>
```

得到的结果跟通过命令行方式创建的 ConfigMap 没什么两样。

##### 使用示例

知道了如何创建 ConfigMap ，我们再来看下如何使用。

有两种方式可以使用创建好的 ConfigMap ，一种是通过环境变量将 ConfigMap 注入到容器内部，另一种是通过卷挂载的方式直接将 ConfigMap 以文件形式挂载到容器。

- 通过环境变量方式引用

创建 `use-configmap-env-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "use-configmap-env"
  namespace: default
spec:
  containers:
    - name: use-configmap-env
      image: "alpine"
      # 一次引用单个值
      env:
        - name: FOO
          valueFrom:
            configMapKeyRef:
              name: c2
              key: foo
      # 一次引用所有值
      envFrom:
        - prefix: CONFIG_  # 配置引用前缀
          configMapRef:
            name: c2
      command: ["echo", "$(FOO)", "$(CONFIG_bar)"]
```

这里创建一个名为 `use-configmap-env` 的 Pod，Pod 的容器分别用两种方式引用 ConfigMap 的内容。

指定 `spec.containers.env` 可以为容器引用一个 ConfigMap 的 `key: value` 对，`valueFrom.configMapKeyRef` 表明我们要引用 ConfigMap ，ConfigMap 的名称为 `c2`，引用的 `key` 为 `foo`。

指定 `spec.containers.envFrom` 则可以一次将 ConfigMap 中的所有 `key: value` 传递给容器，只需要通过 `configMapRef.name` 指定 ConfigMap 的名称即可。`prefix` 可以给引用的 `key` 前面增加统一前缀。

容器启动命令为 `echo $(FOO) $(CONFIG_bar)`，可以分别打印通过 `env` 和 `envFrom` 两种方式引用的 ConfigMap 的内容。

```bash
# 创建 Pod

$ kubectl apply -f use-configmap-env-demo.yaml
# 通过查看 Pod 日志来观察容器内部引用 ConfigMap 结果
$ kubectl logs use-configmap-env
bar baz
```

实验结果表明，容器内部可以通过环境变量的方式引用到 ConfigMap 的内容。

- 通过卷挂载方式引用

创建 `use-configmap-volume-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "use-configmap-volume"
  namespace: default
spec:
  containers:
    - name: use-configmap-volume
      image: "alpine"
      command: ["sleep", "3600"]
      volumeMounts:
        - name: configmap-volume
          mountPath: /usr/share/tmp  # 容器挂载目录
  volumes:
    - name: configmap-volume
      configMap:
        name: c2
```

这里创建一个名为 `use-configmap-volume` 的 Pod，通过 `spec.containers.volumeMounts` 指定容器挂载，`name` 指定挂载的卷名，`mountPath` 指定容器内部挂载路径（也就是将 ConfigMap 挂载到容器内部的指定目录下），`spec.volumes` 声明一个卷，`configMap.name` 表明了这个卷要引用的 ConfigMap 名称。

可以通过如下命令创建 Pod 并验证容器内部能否引用到 ConfigMap。

```bash
# 创建 Pod
$ kubectl apply -f use-configmap-volume-demo.yaml
# 进入 Pod 容器内部
$ kubectl exec -it use-configmap-volume -- sh
# 进入容器挂载目录
/ # cd /usr/share/tmp/
# 查看挂载目录下的文件
/usr/share/tmp # ls
bar  foo
# 查看文件内容
/usr/share/tmp # cat foo
bar
/usr/share/tmp # cat bar
baz
```

创建 Pod 后，可以通过 `kubectl exec` 命令进入容器内部，查看容器 `/usr/share/tmp/` 目录，会有两个以 ConfigMap 中 `key` 为名称的文本文件（`foo`、`bar`），文件内容则为 `key` 所对应的 `value` 内容。

以上我们演示了两种方式将 ConfigMap 的内容注入到容器内部，容器内部的应用可以分别通过读取环境变量、文件内容的方式使用配置信息。

#### Secret

熟悉了 ConfigMap 的用法，接下来看下 Secret 的使用。

其实 Secret 同 ConfigMap 非常类似，只不过它会对存储的数据进行 base64 编码，Secret 卷用来给 Pod 传递敏感信息，例如密码、SSH 密钥等。

##### 创建示例

- 通过命令行创建

Secret 同样可以通过 `--from-literal` 参数来指定 `key: value`，不过这里演示另一种方式，通过 `--form-file` 参数直接从文件加载配置，文件名即为 `key`，文件内容作为 `value`。

```bash
# generic 参数对应 Opaque 类型，既用户定义的任意数据
$ kubectl create secret generic s1 --from-file=foo.txt
```

`foo.txt` 内容如下：

```
foo=bar
bar=baz
```

查看创建出来的 Secret，与 ConfigMap 不同的是，创建 Secret 需要指明类型，以上命令通过指定 `generic` 参数来创建类型为 `Opaque` 的 Secret ，这也是 Secret 默认类型（Secret 支持的类型可以在官方文档查看，不过初期学习阶段你完全可以不必关心，只使用默认类型即可，因为只通过默认类型也能够实现其他几种类型的功能）。

```bash
$ kubectl describe secret s1
Name:         s1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
foo.txt:  16 bytes
```

可以发现，与 ConfigMap 不同，Secret 中的数据的 `value` 是不会直接展示出来的，仅展示了 `value` 的字节大小，这也是 Secret 名称的含义，为了保存密文。

- 通过 Yaml 文件创建

创建 `secret-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: s2
  namespace: default
type: Opaque  # 默认类型
data:
  user: cm9vdAo=
  password: MTIzNDU2Cg==
```

通过 `kubectl apply` 命令应用这个文件。

```bash
$ kubectl apply -f secret-demo.yaml

$ kubectl get secret s2
NAME   TYPE     DATA   AGE
s2     Opaque   2      59s

$ kubectl describe secret s2
Name:         s2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  7 bytes
user:      5 bytes
```

同样能够正确创建出 Secret 资源，通过 Yaml 文件创建 Secret 时，指定的 `data` 内容必须经过 `base64` 编码，如我们指定的 `user` 和 `password` 都是编码后的结果。

```
data:
  user: cm9vdAo=
  password: MTIzNDU2Cg==
```

如果觉得麻烦，也可以使用原始字符串方式，如下：

```
data:
  stringData:
   user: root
   password: "123456"
```

以上两种方式等价，更推荐使用 `base64` 编码的方式。

##### 使用示例

同 ConfigMap 使用方式一样，我们也可以通过环境变量或卷挂载的方式来使用 Secret。

这了演示通过卷挂载方式引用 Secret ，首先创建 `use-secret-volume-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "use-secret-volume-demo"
  namespace: default
spec:
  containers:
    - name: use-secret-volume-demo
      image: "alpine"
      command: ["sleep", "3600"]
      volumeMounts:
        - name: secret-volume
          mountPath: /usr/share/tmp # 容器挂载目录
  volumes:
    - name: secret-volume
      secret:
        secretName: s2
```

这里创建一个名为 `use-secret-volume-demo` 的 Pod，Pod 的容器通过卷挂载方式引用 Secret 的内容。

```bash
# 创建 Pod
$ kubectl apply -f use-secret-volume-demo.yaml

# 进入 Pod 容器内部
$ kubectl exec -it use-secret-volume-demo -- sh
# 进入容器挂载目录
/ # cd /usr/share/tmp/
# 查看挂载目录下的文件
/usr/share/tmp # ls
password  user
# 查看文件内容
/usr/share/tmp # cat password 
123456
/usr/share/tmp # cat user 
root
```

可以发现被挂载到容器内部以后，Secret 的内容将变成明文存储。容器内部应用可以同使用 ConfigMap 一样来使用 Secret。

由此可见，无论是 ConfigMap 还是 Secret，都可以用来存储配置信息，可以通过 ConfigMap 来存放普通配置，通过 Secret 来存放敏感配置。

值得一提的是，使用环境变量方式引用 ConfigMap 或 Secret，当 ConfigMap 或 Secret 内容变更时，容器内部引用的内容不会自动更新；使用卷挂载方式引用 ConfigMap 或 Secret，当 ConfigMap 或 Secret 内容变更时，容器内部引用的内容会自动更新。如果容器内部应用支持配置文件热加载，那么通过卷挂载对的方式引用 ConfigMap 或 Secret 内容将是推荐方式。

#### DownwardAPI

DownwardAPI 可以将 Pod 对象自身的信息注入到 Pod 所管理的容器内部。

##### 使用示例

创建 `downwardapi-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downwardapi-volume-demo
  labels:
    app: downwardapi-volume-demo
  annotations:
    foo: bar
spec:
  containers:
    - name: downwardapi-volume-demo
      image: alpine
      command: ["sleep", "3600"]
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          # 指定引用的 labels
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          # 指定引用的 annotations
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```

使用如下：

```bash
# 创建 Pod
$ kubectl apply -f downwardapi-demo.yaml
pod/downwardapi-volume-demo created

# 进入 Pod 容器内部
$ kubectl exec -it downwardapi-volume-demo -- sh
# 进入容器挂载目录
/ # cd /etc/podinfo/
# 查看挂载目录下的文件
/etc/podinfo # ls
annotations  labels
# 查看文件内容
/etc/podinfo # cat annotations 
foo="bar"
kubectl.kubernetes.io/last-applied-configuration="{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{\"foo\":\"bar\"},\"labels\":{\"app\":\"downwardapi-volume-demo\"},\"name\":\"downwardapi-volume-demo\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sleep\",\"3600\"],\"image\":\"alpine\",\"name\":\"downwardapi-volume-demo\",\"volumeMounts\":[{\"mountPath\":\"/etc/podinfo\",\"name\":\"podinfo\"}]}],\"volumes\":[{\"downwardAPI\":{\"items\":[{\"fieldRef\":{\"fieldPath\":\"metadata.labels\"},\"path\":\"labels\"},{\"fieldRef\":{\"fieldPath\":\"metadata.annotations\"},\"path\":\"annotations\"}]},\"name\":\"podinfo\"}]}}\n"
kubernetes.io/config.seen="2022-03-12T13:06:50.766902000Z"
/etc/podinfo # cat labels
app="downwardapi-volume-demo"
```

不难发现，DownwardAPI 的使用方式同 ConfigMap 和 Secret 如出一辙，通过卷挂载方式挂载到容器内部以后，都是在容器挂载的目录下生成对应文件，用来存储 key: value。不同的是 DownwardAPI 不需要预先定义，可以直接使用，因为它能引用的内容已经都在当前 yaml 文件中定义好了。

#### 小结

ConfigMap 、Secret 、DownwardAPI 这三种 Volume 存在的意义不是为了保存容器中的数据，而是为了给容器传递预先定义好的数据。

### 临时卷

接下来我们要关注的是临时卷，即临时存储。K8s 支持的临时存储中最常用的就是如下两种：

- EmptyDir
- HostPath

临时存储卷会遵从 Pod 的生命周期，与 Pod 一起创建和删除。

#### EmptyDir

先来看 EmptyDir 如何使用，EmptyDir 相当于使用 docker 时通过 `--volume/-v` 挂载时的隐式 Volume 形式，即不显式的声明在宿主机上的目录。K8s 会在宿主机上创建一个临时目录，被挂载到容器所声明的 `mountPath` 目录上。

##### 使用示例

创建 `emptydir-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "emptydir-nginx-pod"
  namespace: default
  labels:
    app: "emptydir-nginx-pod"
spec:
  containers:
    - name: html-generator
      image: "alpine:latest"
      command: ["sh", "-c"]
      args:
       - while true; do
          date > /usr/share/index.html;
          sleep 1;
         done
      volumeMounts:
        - name: html
          mountPath: /usr/share
    - name: nginx
      image: "nginx:latest"
      ports:
        - containerPort: 80
          name: http
      volumeMounts:
        - name: html
          # nginx 容器 index.html 文件所在目录
          mountPath: /usr/share/nginx/html
          readOnly: true
  volumes:
    - name: html
      emptyDir: {}
```

这里创建一个名为 `emptydir-nginx-pod` 的 Pod，它包含两个容器 `html-generator` 和 `nginx`，顾名思义，`html-generator` 用来不停的生成 `html` 文件，`nginx` 是一个 Web 服务，用来展示 `html-generator` 生成的 `index.html` 文件。

通过两个容器对的描述信息可以发现，`html-generator` 不停的将当前时间写入到 `/usr/share/index.html` 下，并将 `/usr/share` 目录挂载到名为 `html` 的卷中，而 `nginx` 容器则将 `/usr/share/nginx/html` 目录挂载到名为 `html` 的卷中，这样两个容器通过同一个卷 `html` 挂载到了一起。现在通过 `kubectl apply` 命令应用这个文件。

```bash
# 创建 Pod
$ kubectl apply -f emptydir-demo.yaml
pod/emptydir-nginx-pod created

# 进入 Pod 容器内部
$ kubectl exec -it pod/emptydir-nginx-pod -- sh
# 查看系统时区
/ # curl 127.0.0.1
Sun Mar 13 08:40:01 UTC 2022
/ # curl 127.0.0.1
Sun Mar 13 08:40:04 UTC 2022
```

根据 `nginx` 容器内部 `curl 127.0.0.1` 命令输出结果可以发现，`nginx` 容器 `/usr/share/nginx/html/indedx.html` 文件内容即为 `html-generator` 容器 `/usr/share/index.html` 文件内容。

能够实现此效果的原理是，当我们声明卷类型为 `emptyDir: {}` 后，K8s 会自动在主机目录上创建一个临时目录，然后将 `html-generator` 容器 `/usr/share/` 目录和 `nginx` 容器 `/usr/share/nginx/html/` 同时挂载到这个临时目录上，这样两个容器的目录就能够同步数据了。

需要注意的是，容器崩溃并不会导致 Pod 被从节点上移除，因此容器崩溃期间 EmptyDir 卷中的数据是安全的。另外，`emptyDir.medium` 除了可以设成 `{}`，还可以设成 `Memory` 表示内存挂载。

#### HostPath

HostPath 卷能将主机节点文件系统上的文件或目录挂载到指定的 Pod 中，与 EmptyDir 不同的是，当 Pod 删除时，与之绑定的 HostPath 并不会随之删除，如果新创建一个 Pod 并且挂载到上一个 Pod 使用过的 HostPath，原 HostPath 中的内容仍然存在。但这仅限于新的 Pod 和已经删除的 Pod 被调度到同一节点上，所以严格来讲 HostPath 仍然属于临时存储。

##### 使用示例

HostPath 卷的一个典型应用是将主机节点上的时区通过卷挂载的方式注入到容器内部，这样能够保证启动的容器和主机节点时间同步。

创建 `hostpath-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "hostpath-volume-pod"
  namespace: default
  labels:
    app: "hostpath-volume-pod"
spec:
  containers:
    - name: hostpath-volume-container
      image: "alpine:latest"
      command: ["sleep", "3600"]
      volumeMounts:
        - name: localtime
          mountPath: /etc/localtime
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
```

要实现时间同步，只需要将主机目录 `/usr/share/zoneinfo/Asia/Shanghai` 通过卷挂载的方式挂载到容器内部的 `/etc/localtime` 目录即可。通过 `kubectl apply` 命令应用这个文件，然后进入 Pod 容器内部使用 `date` 命令查看容器当前时间。

```bash
# 创建 Pod
$ kubectl apply -f hostpath-demo.yaml
pod/hostpath-volume-pod created
# 进入 Pod 容器内部
$ kubectl exec -it hostpath-volume-pod -- sh
# 执行 date 命令输出当前时间
/ # date
Sun Mar 13 17:00:22 CST 2022  # 上海时区
```

输出结果为 `Sun Mar 13 17:00:22 CST 2022`，其中 `CST` 代表了上海时区，也就是主机节点的时区，如果不通过卷挂载的方式将主机时区挂载到容器内部，容器默认时区为 `UTC` 时区。

#### 小结

本节介绍了 K8s 临时存储方案以及应用，EmptyDir 适用范围较少，可以当作临时缓存或者耗时任务检查点等。

绝大多数 Pod 应该忽略主机节点，不应该访问节点上的文件系统，有时候 DaemonSet 可能需要访问主机节点的文件系统，HostPath 可以用来同步主机节点时区到容器，其他情况下使用较少，HostPath 的最佳实践就是尽量不使用 HostPath。

### 持久卷

临时卷的生命周期与 Pod 相同，当 Pod 被删除时，K8s 会自动删除 Pod 挂载的临时卷。而当我们的 Pod 中应用需要将数据保存到磁盘，且即使 Pod 被调度到其他节点数据也应该存在时，我们就需要一个真正的持久化存储了。

K8s 支持的持久卷类型非常多，这里列出了 `v1.24` 版本支持的所有卷：

- `awsElasticBlockStore` - AWS 弹性块存储（EBS）
- `azureDisk` - Azure Disk
- `azureFile` - Azure File
- `cephfs` - CephFS volume
- `csi` - 容器存储接口 (CSI)
- `fc` - Fibre Channel (FC) 存储
- `gcePersistentDisk` - GCE 持久化盘
- `glusterfs` - Glusterfs 卷
- `iscsi` - iSCSI (SCSI over IP) 存储
- `local` - 节点上挂载的本地存储设备
- `nfs` - 网络文件系统 (NFS) 存储
- `portworxVolume` - Portworx 卷
- `rbd` - Rados 块设备 (RBD) 卷
- `vsphereVolume` - vSphere VMDK 卷
- ...

看到这么多持久卷类型不必恐慌，有很多类型我也是第一次见，K8s 针对持久卷有一套独有的思想，旨在让开发者不必关心这背后的持久化存储类型，在 K8s 中，开发者无论使用哪种持久卷，其用法都是一致的。

K8s 持久卷设计架构如下：

![K8s storage](k8s-storage.png)

`Node1` 和 `Node2` 分别代表两个工作节点，当我们在工作节点创建 Pod 时，可以通过 `spec.containers.volumeMounts` 来指定容器挂载目录，通过 `spec.volumes` 来指定挂载卷，之前我们用这个挂载卷挂载了配置信息和临时卷，这里指定挂载持久卷可以采用同样的方式，图中每个 `volumes` 指向的是下方存储集群中不同的存储类型，K8s 完全支持这样做。为了保证高可用，我们通常会搭建一个存储集群，而存储集群内部可以使用任何 K8s 支持的持久化存储，如上图的 `NFS`、`CephFS`、`CephRBD`。

#### 存储集群

关于存储集群，因为我们通过 Pod 来操作存储，而 Pod 都会部署在 Node 中，所以存储集群最好跟 Node 集群搭建在同一内网，这样速度更快。

![K8s storage cluster](storage-cluster.png)

#### 使用 NFS

这里我们使用 NFS 存储来演示 K8s 对持久卷的支持（NFS 测试环境搭建过程可以参考文章结尾的附录部分），创建 `nfs-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "nfs-nginx-pod"
  namespace: default
  labels:
    app: "nfs-nginx-pod"
spec:
  containers:
    - name: nfs-nginx
      image: "nginx:latest"
      ports:
        - containerPort: 80
          name: http
      volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html/
  volumes:
    - name: html-volume
      nfs:
        server: 192.168.99.101  # 指定 nfs server 地址
        path: /nfs/data/nginx  # 目录必须存在
```

持久化挂载方式与临时卷大同小异，我们这里同样使用一个 Nginx 服务来进行测试，将容器 `index.html` 所在目录 `/usr/share/nginx/html/` 挂载到 NFS 服务的 `/nfs/data/nginx` 目录下，在 `spec.volumes` 配置项中指定 NFS 服务，`server` 指明了 NFS 服务器地址，`path` 指明 NFS 服务器中挂载的路径，这个路径必须是已经存在的路径。通过 `kubectl apply` 命令应用这个文件。

```bash
$ kubectl apply -f nfs-demo.yaml
```

接下来我们查看这个 Pod 使用 NFS 存储的结果：

- NFS 节点

在 NFS 节点中我们准备一个 `index.html` 文件，其内容为 `hello nfs`。

![index.html](nfs-index.png)

- Pod

使用 `curl` 命令直接访问 Pod 的 `IP` 地址，即可返回 Nginx 服务的 `index.html` 内容，结果输出为 `hello nfs`，证明 NFS 持久卷挂载成功。

![hello nfs](curl-pod.png)

可以登入 Pod 容器，通过 `df -Th` 命令查看容器目录挂载信息，可以发现，容器的 `/usr/share/nginx/html/` 目录被挂载到 NFS 服务的 `/nfs/data/nginx` 目录。

![mount volume](volume.png)

现在，当我们执行 `kubectl delete -f nfs-demo.yaml` 删除 pod 后，NFS 服务器上数据盘中的数据依然存在，这就是持久卷。

#### 持久卷使用痛点

虽然我们通过使用持久卷，解决了临时卷数据易丢失的问题，但目前持久卷的使用方式，还是有一些不足之处：

- Pod 开发人员可能对存储不够了解，却要对接多种存储
- 安全问题，有些存储可能需要账号密码，这些信息不应该暴露给 Pod

为了解决这些不足，K8s 又针对持久化存储抽象出了三种资源 PV、PVC、StorageClass。

#### PV（PersistentVolume）、PVC（PersistentVolumeClaim）、StorageClass

三种资源定义如下：

- PV 描述的是持久化存储数据卷
- PVC 描述的是 Pod 想要使用的持久化存储属性，既存储卷申明
- StorageClass 作用是根据 PVC 的描述，申请创建对应的 PV

PV 和 PVC 的概念可以对应编程语言中的面向对象思想，PVC 是接口，PV 是具体实现。

有了这三种资源类型后，Pod 有两种方式来使用持久卷，分别是静态供应和动态供应。

#### 静态供应

我们先来学习静态供应，静态供应不涉及 StorageClass，只涉及到 PVC 和 PV。

其使用流程图如下：

![static](static-storage.png)

使用静态供应时，Pod 不再直接绑定持久存储，而是会绑定到 PVC 上，然后再由 PVC 跟 PV 进行绑定。这样就实现了 Pod 中的容器可以使用由 PV 真正去申请的持久化存储。

##### 使用示例

创建 `pv-demo.yaml` 内容如下：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-1g
  labels:
    type: nfs
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  nfs:
    server: 192.168.99.101
    path: /nfs/data/nginx1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-100m
  labels:
    type: nfs
spec:
  capacity:
    storage: 100m
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  nfs:
    server: 192.168.99.101
    path: /nfs/data/nginx2
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-500m
  labels:
    app: pvc-500m
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500m
---
apiVersion: v1
kind: Pod
metadata:
  name: "pv-nginx-pod"
  namespace: default
  labels:
    app: "pv-nginx-pod"
spec:
  containers:
    - name: pv-nginx
      image: "nginx:latest"
      ports:
        - containerPort: 80
          name: http
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html/
  volumes:
    - name: html
      persistentVolumeClaim:
        claimName: pvc-500m
```

以上 `Yaml` 文件定义了两个 PV、一个 PVC 和一个 Pod。

两个 PV 申请容量分别为 `1Gi`、`100m`，通过 `spec.capacity.storage` 指定，并且他们通过 `spec.nfs` 指定了 NFS 存储服务的地址和路径。PVC 则申请 `500m` 大小的存储。最后在 Pod 的定义中，`spec.volumes` 绑定了这个名为 `pvc-500m` 的 PVC，而没有直接去绑定 NFS 存储服务。

通过 `kubectl apply` 命令应用这个文件：

```bash
$ kubectl apply -f pv-demo.yaml
```

创建好这些资源后，来观察下结果如何：

首先通过 `kubectl get pod` 命令查看新创建的 Pod，通过 `curl` 命令访问 Pod 的 `IP` 地址，可以得到 `hello nginx1` 的响应结果。

![hello nginx1](hello-nginx1.png)

通过 `kubectl get pvc` 查看创建的 PVC，`STATUS` 字段标识这个 PVC 已经处于绑定（`Bound`）状态，也就是与 PV 进行了绑定，`CAPACITY` 字段标识这个 PVC 绑定到了 `1Gi` 的 PV 上，尽管我们申请的 PVC 大小是 `500m`，但由于我们创建的两个 PV 大小分别是 `1Gi` 和 `100m`，K8s 会帮我们选择满足条件的最优解，因为没有刚好等于 `500m` 大小的 PV 存在，而 `100m` 又不满足，所以 PVC 会自动与 `1Gi` 大小的 PV 进行绑定。

![pvc](get-pvc.png)

通过 `kubectl get pv` 来查询创建的 PV 资源，可以发现 `1Gi` 大小的 PV `STATUS` 字段为 `Bound` 状态，`CLAIM` 有值，它标识的就是与之绑定的 PVC 的名字。

![pv](get-pv.png)

现在我们来登录 NFS 服务器，确认NFS 存储上不同持久卷（PV）挂载的目录下文件内容。

![nfs volume](nfs-volume.png)

可以看到，`/nfs/data/nginx1` 目录下的 `index.html` 内容为 `hello nginx1`，即为上面通过 `curl` 命令访问 Pod 服务的响应结果。

总结下整个持久卷使用流程：首先创建一个 Pod，这个 Pod 的 `spec.volumes` 中绑定了 PVC，而 PVC 只是一个存储申明，代表我们的 Pod 需要什么样的持久化存储，它并没有标明 NFS 服务地址，也没有明确要和哪个 PV 进行绑定，我们只需要创建出这个 PVC 即可，接着我们创建了两个 PV，在 PV 中也没有明确指出要与哪个 PVC 进行绑定，只需要指出它的大小和 NFS 存储服务地址即可，此时 K8s 会自动帮我们进行 PVC 和 PV 的绑定，这样 Pod 就和 PV 产生了联系，也就可以访问持久化存储了，这就是静态供应持久卷的使用流程。

细心的你可能已经发现，前文中我提到静态供应不涉及 StorageClass，但是我在定义 PVC 和 PV 的 `Yaml` 文件时，还是都为其指定了 `spec.storageClassName` 值为 `nfs-storage`，因为这是一个比较好的实践，便于管理，只有具有相同 StorageClass 的 PVC 和 PV 才可以进行绑定。

无论是 PVC 还是 PV，都有一个 `spec.accessModes` 字段，这个字段标识了持久卷的访问模式，K8s 持久化支持四中访问模式：

- RWO - ReadWriteOnce —— 卷可以被一个节点以读写方式挂载
- ROX - ReadOnlyMany —— 卷可以被多个节点以只读方式挂载
- RWX - ReadWriteMany —— 卷可以被多个节点以读写方式挂载
- RWOP - ReadWriteOncePod —— 卷可以被单个 Pod 以读写方式挂载（ K8s `1.22` 以上版本）

只有具有相同读写模式的 PVC 和 PV 才可以进行绑定。

现在我们来继续实验，通过命令 `kubectl delete pod pv-nginx-pod` 删除 Pod，再次查看 PVC 和 PV 状态。

![pv-pvc](get-pvc-status.png)

说明 Pod 删除后，PVC 和 PV 还在。Pod 删除并不影响 PVC 的存在，而当 PVC 删除时 PV 是否删除，则可以通过设置回收策略来决定。

PV 回收策略（`pv.spec.persistentVolumeReclaimPolicy`）有三种：

- Retain —— 手动回收，也就是说删除 PVC 后，PV 依然存在，需要管理员手动进行删除
- Recycle —— 基本擦除 (相当于 rm -rf /*)（新版已废弃不建议使用，建议使用动态供应）
- Delete —— 删除 PV，即级联删除

现在通过命令 `kubectl delete pvc pvc-500m` 删除 PVC，查看 PV 状态。

![pv](get-pv-status.png)

可以看到 PV 依然存在，其 `STATUS` 已经变成 `Released`，此状态下的 PV 无法再次绑定到 PVC，需要由管理员手动删除，这是由回收策略决定的。

注意：绑定了 Pod 的 PVC，如果 Pod 正在运行中，PVC 无法删除。

##### 静态供应的不足

我们一起体验了静态供应的流程，虽然比直接在 Pod 中绑定 NFS 服务更加清晰，但静态供应依然存在不足。

- 首先会造成资源浪费，如上面示例中，PVC 申请 `500m`，而没有刚好等于 `500m` 的 PV 存在，这 K8s 会将 `1Gi` 的 PV 与之绑定
- 还有一个致命的问题，如果当前没有满足条件的 PV 存在，则这 PVC 一直无法绑定到 PV 处于 `Pending` 状态，Pod 也将无法启动，所以就需要管理员提前创建好大量 PV 来等待新创建的 PVC 与之绑定，或者管理员时刻监控是否有满足 PVC 的 PV 存在，如果不存在则马上进行创建，这显然是无法接受的

#### 动态供应

介于静态供应存在不足，K8s 推出一种更加方便的持久卷使用方式，即动态供应。

动态供应核心组件上面已经见过，就是 StorageClass——存储类。

StorageClass 主要作用有两个：

- 一是资源分组，我们上面使用静态供应时指定 StorageClass 的目前就是对资源进行分组，便于管理
- 二是 StorageClass 能够帮我们根据 PVC 请求的资源，自动创建出新的 PV，这个功能是 StorageClass 中 `provisioner` 存储插件帮我们来做的。

其使用流程图如下：

![dynamic](dynamic-storage.png)

相较于静态供应，动态供应在 PVC 和 PV 之间增加了存储类，这次 PV 并不需要提前创建好，只要我们申请了 PVC 并且绑定了有  `provisioner` 功能的 StorageClass，StorageClass 会帮我们自动创建 PV 并与 PVC 进行绑定。

我们可以根据提供的持久化存储类型，分别创建对应的 StorageClass，比如：

- nfs-storage
- cephfs-storage
- rbd-storage

也可以设置一个默认 StorageClass， 通过在创建 StorageClass 资源时指定对应的 `annotations` 实现：

```yaml
apiVersion: storage.K8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
...
```

当创建 PVC 时不指定 `spec.storageClassName`，这个 PVC 就会使用默认 StorageClass。

##### 使用示例

这里我们仍然使用 NFS 来作为持久化存储，首先我们需要有一个能够支持自动创建 PV 的 `provisioner`，可以在 GitHub 中找到一些开源的实现，我这里使用了 `nfs-subdir-external-provisioner` 这个存储插件，具体安装方法也非常简单，只需要通过 `kubectl apply` 命令应用它提供的几个 `Yaml` 文件即可，你可以点击[项目链接](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)查看文档。
当存储插件安装完成，我们可以创建如下 StorageClass：

```yaml
apiVersion: storage.K8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: K8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "true"
```

这个 StorageClass 指定了 `provisioner` 为我们安装好的 `K8s-sigs.io/nfs-subdir-external-provisioner`，指定了 `provisioner` 的 StorageClass 就有了自动创建 PV 的能力。 `provisioner` 其实本质上也是一个 Pod，可以通过 `kubectl get pod` 来查看，是这个 Pod 能够自动创建 PV。

![StorageClass](get-storageclass.png)

现在 `provisioner` 和 StorageClass 都已经创建好了，接下来就可以进行动态供应的实验了，创建 `nfs-provisioner-demo.yaml` 内容如下：

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: "test-nginx-pod"
  namespace: default
  labels:
    app: "test-nginx-pod"
spec:
  containers:
    - name: test-nginx
      image: "nginx:latest"
      ports:
        - containerPort: 80
          name: http
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html/
  volumes:
    - name: html
      persistentVolumeClaim:
        claimName: test-claim
```

这里我们只定义了一个 PVC 和一个 Pod，并没有定义 PV，PVC 的 `spec.storageClassName` 指定为上面创建好的 StorageClass `nfs-storage`，现在只需要通过 `kubectl apply` 命令来创建出 PVC 和 Pod 即可：

```bash
$ kubectl apply -f nfs-provisioner-demo.yaml
persistentvolumeclaim/test-claim created
pod/test-nginx-pod created
```

现在可以查看 PV、PVC 和 Pod，发现 PV 也已经被自动创建出来了，并且它们之间实现了绑定关系。

![Bound](pv-pvc-bound.png)

现在登录 NFS 服务，给远程挂载的卷写入 `hello nfs` 数据。

![NFS write data](hello-nfs.png)

在 K8s 侧，就可以通过 `curl` 命令验证这次挂载的正确性了。

![NFS data](curl-nfs.png)

此时如果你通过 `kubectl delete -f nfs-provisioner-demo.yaml` 删除 Pod 和 PVC，PV 也会跟着删除，因为 PV 的删除策略是 Delete ，不过你会发现，NFS 卷中的数据还在，只不过被归档成了以 `archived` 开头的目录，这个功能是 `K8s-sigs.io/nfs-subdir-external-provisioner` 这个存储插件帮我们实现的，你可以回到上面查看在定义 StorageClass 是指定的 `parameters.archiveOnDelete` 属性，这就是存储插件的强大。

![NFS archive](ls-nfs.png)

#### 小结

通过定义指定了 `provisioner` 的 StorageClass，我们不仅可以实现 PV 的自动化创建，甚至可以实现数据删除时自动归档的功能，这完全依附于 K8s 动态供应存储设计的精妙。
通过以上的实验，我们可以得出结论：动态供应是持久化存储最佳实践。

### 附录

#### NFS 实验环境搭建

NFS 全称 Network File System，是一种分布式存储，它能够通过局域网实现不同主机间目录共享。

##### 架构图

一个 Server 节点和两个 Client 节点。

![NFS-arch](nfs-arch.png)

##### NFS 在 Centos 系统中的搭建过程：

Server 节点：

```bash
# 安装 nfs 工具
yum install -y nfs-utils

# 创建 NFS 目录
mkdir -p /nfs/data/

# 创建 exports 文件，* 表示所有网络上的 IP 都可以访问
echo "/nfs/data/ *(insecure,rw,sync,no_root_squash)" > /etc/exports

# 启动 rpc 远程绑定功能、NFS 服务功能
systemctl enable rpcbind
systemctl enable nfs-server
systemctl start rpcbind
systemctl start nfs-server

# 重载使配置生效
exportfs -r
# 检查配置是否生效
exportfs
# 输出结果如下所示
# /nfs/data       *
```

Client 节点：

```bash
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 安装 nfs 工具
yum install -y nfs-utils

# 挂载 nfs 服务器上的共享目录到本机路径 /root/nfsmount
mkdir /root/nfsmount
mount -t nfs 192.168.99.101:/nfs/data /root/nfsmount
```

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/

**参考**

- 图书《Kubernetes in Action中文版》
- kubernetes 存储：https://kubernetes.io/zh-cn/docs/concepts/storage/
- kubernetes 配置：https://kubernetes.io/zh-cn/docs/concepts/configuration/
- NFS 存储插件：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

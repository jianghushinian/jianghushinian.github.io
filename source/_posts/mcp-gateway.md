---
title: '使用 MCP Gateway 一键将你的 HTTP 接口转换成 MCP Server'
date: 2025-05-14 16:00:31
tags:
- Go
- MCP
- 网关
categories:
- Go
---

最近 MCP 非常火热，各家厂商也都提供了自己的 MCP Server 实现，截止发文当日在 [https://mcp.so/](https://mcp.so/) 网站中已经收录了超过 1.3w+ 数量的 MCP Server，仿佛一夜之间所有的 HTTP 接口都变成了 MCP 接口。

MCP Server 短期内如此大规模的爆发，市面上这么多的 HTTP 服务，全部要转换成 MCP 服务，显然不是一天两天就能通过开发实现的，这需要大量的人力资源。那么有没有一种方式能够低成本的将 HTTP Server 转换成 MCP Server 呢？本文就为大家介绍一款能够实现一键将你的 HTTP 接口转换成 MCP Server 的 Go 项目 [mcp-gateway](https://github.com/mcp-ecosystem/mcp-gateway)。

<!-- more -->

### MCP Gateway 介绍

MCP Gateway 是用 Go 编写的服务，顾名思义本质上是一个网关服务，实现了 MCP 协议。它能够将现有 API 快速转化为 MCP 服务，无需改动任何一行代码。

使用  mcp-gateway 将 HTTP 接口转换为 MCP 服务，我们唯一要提供的就是一个符合 OpenAPI 规范的 HTTP 接口文档。

> NOTE:
>
> 关于 OpenAPI 规范，你可以参考我写的另一篇文章《[使用 OpenAPI 构建 API 文档](https://jianghushinian.cn/2023/02/12/build-api-documentation-using-openapi/)》。
>

MCP Gateway 提供了控制台页面来管理 MCP Server 配置：

![MCP Gateway 控制台](mcp-gateway-console.png)

上图中两个卡片就表示两个 MCP 服务，服务可以来自于手动编写的 MCP 服务配置，也可以来自于通过 OpenAPI 接口文档进行一键转换得到的 MCP 服务配置。

有了 MCP 服务，我们可以通过与大模型对话的方式来测试 MCP 服务是否可用：

![MCP Gateway 聊天](mcp-gateway-chat.png)

图中展示了通过与大模型进行聊天会话的形式，来调用 MCP Server 创建用户。

接下来，我将带大家一起在本地部署一个 MCP Gateway 并演示其用法。

### 部署 MCP Gateway

MCP Gateway 官方文档中 [https://mcp.ifuryst.com/getting-started/quick-start/](https://mcp.ifuryst.com/getting-started/quick-start/) 提供了 Docker 和二进制两种部署方式，都非常简单，可以实现一键部署。在这里我们换一种玩法，实现在 Kubernetes 中部署 MCP Gateway 服务。

#### 准备 Kubernetes 集群

本次示例，我将在本机 Mac 环境中使用 kind 安装 Kubernetes 集群，这样你也能非常方便的复现，只要跟着我一步一步操作，你就能轻松部署 MCP Gateway 服务。

首先编写如下 kind 配置：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: mcp-cluster
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        listenAddress: "0.0.0.0"
        protocol: TCP
      - containerPort: 30081
        hostPort: 30081
        listenAddress: "0.0.0.0"
        protocol: TCP
      - containerPort: 30082
        hostPort: 30082
        listenAddress: "0.0.0.0"
        protocol: TCP
      - containerPort: 30083
        hostPort: 30083
        listenAddress: "0.0.0.0"
        protocol: TCP

```

这里定义一个单节点的 Kubernetes 集群，并映射了多个容器端口到主机端口，这样，当我们为容器创建 NodePort 类型的 Service 资源时，其 NodePort 端口就能在 Mac 主机上直接访问了。

按照如下步骤来安装 Kubernetes 集群：

> NOTE:
>
> 你需要提前安装好 kind 工具：[https://kind.sigs.k8s.io/docs/user/quick-start/#installation](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
>

```bash
# 根据配置文件创建 Kubernetes 集群
$ kind create cluster --config=config.yaml
# 查看本机现有的 Kubernetes 集群
$ kind get clusters                       
mcp-cluster
# 查看 Kubernetes 集群节点信息
$ k get node -o wide
NAME                        STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION     CONTAINER-RUNTIME
mcp-cluster-control-plane   Ready    control-plane   56s   v1.32.2   172.22.0.3    <none>        Debian GNU/Linux 12 (bookworm)   6.10.14-linuxkit   containerd://2.0.2

```

现在你就拥有了一个 Kubernetes 集群。

#### 在 Kubernetes 集群部署 MCP Gateway

接下来，我们需要编写 Kubernetes 资源 yaml，以此来实现部署 MCP Gateway。

MCP Gateway 提供了如下几个组件：

+ API Server: 管理平台后端，可理解为控制面
+ MCP Gateway: 核心服务，负责实际的网关服务，可理解为数据面
+ Mock User Service: 模拟用户服务，提供测试用的用户服务（你的存量API服务可能就是类似这样的）
+ Web 前端: 管理平台前端，提供可视化的管理界面
+ Nginx: 反向代理其他几个服务

不过官方其实只提供了一个 All-in-One 的镜像，所以部署起来没想象中复杂，无需独立部署以上各个组件。

关于 MCP Gateway 的设计你可以在这里 [https://mcp.ifuryst.com/development/architecture](https://mcp.ifuryst.com/development/architecture) 看到 MCP 网关架构总揽图。

根据官方提供的 Docker 部署文档，我们可以编写出如下 Kubernetes 资源 yaml：

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: mcp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-gateway
  namespace: mcp
  labels:
    app: mcp-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-gateway
  template:
    metadata:
      name: mcp-gateway
      labels:
        app: mcp-gateway
    spec:
      containers:
        - name: mcp-gateway
          image: registry.ap-southeast-1.aliyuncs.com/mcp-ecosystem/mcp-gateway-allinone:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80   # Web 界面端口
            - containerPort: 5234 # API Server 端口
            - containerPort: 5235 # MCP Gateway 端口
            - containerPort: 5335 # MCP Gateway 管理端口（承载诸如 reload 的内部接口，生产环境切勿对外）
            - containerPort: 5236 # Mock User Service 端口
          env:
            - name: ENV
              value: production
            - name: OPENAI_BASE_URL
              value: https://dashscope.aliyuncs.com/compatible-mode/v1
            - name: OPENAI_API_KEY
              value: sk-xxx
            - name: OPENAI_MODEL
              value: qwen-turbo
          volumeMounts:
            - mountPath: /app/configs # 配置文件目录
              name: config-volume
            - mountPath: /app/data # 数据目录
              name: data-volume
            - mountPath: /app/.env
              name: env-volume
      restartPolicy: Always
      volumes:
        - name: config-volume
          configMap:
            name: mcp-config
        - name: env-volume
          configMap:
            name: mcp-env
        - name: data-volume
          persistentVolumeClaim:
            claimName: mcp-storage
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-gateway-web
  namespace: mcp
spec:
  type: NodePort
  selector:
    app: mcp-gateway
  ports:
    - name: web
      port: 80
      targetPort: 80
      nodePort: 30080  # 映射到宿主机 30080 端口
    - name: api-server
      port: 5234
      targetPort: 5234
      nodePort: 30081
    - name: mcp-gateway
      port: 5235
      targetPort: 5235
      nodePort: 30082
    - name: mock-user
      port: 5236
      targetPort: 5236
      nodePort: 30083
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mcp-config
  namespace: mcp
data:
  mcp-gateway.yaml: |
    ...
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mcp-env
  namespace: mcp
data:
  ...
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mcp-storage
  namespace: mcp
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard

```

这个资源清单看似很长，不过不要慌，其实这些都是从官方文档 [https://mcp.ifuryst.com/deployment/docker](https://mcp.ifuryst.com/deployment/docker) 中摘抄下来的必要配置。

我为你解读下这个 yaml 是怎么来的：

+ 首先是 Namespace 资源的创建，这个无需多言，我将部署的资源全部放在 mcp 这个 Namespace 下。
+ 接着是 Deployment 资源，这里定义的 Pod 就是在部署 MCP Gateway，其中的镜像就是官方提供的 All-in-One 镜像。
+ 容器的端口说明如下：
    - `80`: Web 界面端口
    - `5234`: API Server 端口
    - `5235`: MCP Gateway 端口
    - `5335`: MCP Gateway 管理端口（承载诸如 reload 的内部接口，生产环境切勿对外）
    - `5236`: Mock User Service 端口
+ Service 资源会通过 NodePort 的方式，暴露这几个端口。而前文中，我们已经在通过 kind 创建 Kubernetes 集群时，将这些 NodePort 端口映射到宿主机了。
+ 此外，容器还挂载了配置文件和持久化存储 PVC，这些也都是参考官方 Docker 部署文档来改造的。
+ 最后，有一个重中之重的配置项，就是容器环境变量中的 `OPENAI_BASE_URL`、`OPENAI_API_KEY` 和 `OPENAI_MODEL`，这三项是用来调用大模型的，你可以申请自己的 API，我这里配置的是阿里的千问大模型，如果你也用千问，那么只需要修改 `OPENAI_API_KEY` 这一项即可。

> NOTE:
> 注意，文章篇幅所限，这里的配置并不完整，你可以访问 https://github.com/jianghushinian/blog-go-example/blob/main/mcp-gateway/mcp-gateway.yaml 获取完整 yaml。

接下来可以通过如下步骤将 MCP Gateway 部署到 Kubernetes 集群：

```bash
# 可以想在宿主机上将 All-in-One 镜像拉取到本地
$ docker pull registry.ap-southeast-1.aliyuncs.com/mcp-ecosystem/mcp-gateway-allinone:latest
...

# 接着将 All-in-One 镜像导入 kind 安装的 Kubernetes 集群
$ kind load docker-image registry.ap-southeast-1.aliyuncs.com/mcp-ecosystem/mcp-gateway-allinone:latest --name mcp-cluster
Image: "registry.ap-southeast-1.aliyuncs.com/mcp-ecosystem/mcp-gateway-allinone:latest" with ID "sha256:f5f22d5680e39822835c700b5ed9cea19e31910f5aed6b7f0957a2fa45f21f72" not yet present on node "mcp-cluster-control-plane", loading...

# 将 MCP Gateway 部署到 Kubernetes 集群
$ kubectl apply -f mcp-gateway.yaml
namespace/mcp created
deployment.apps/mcp-gateway created
service/mcp-gateway-web created
configmap/mcp-config created
configmap/mcp-env created
persistentvolumeclaim/mcp-storage created

# 查看部署的所有资源
$ kubectl -n mcp get all
NAME                               READY   STATUS    RESTARTS   AGE
pod/mcp-gateway-5d745d4f8b-4rtpv   1/1     Running   0          2m16s

NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                     AGE
service/mcp-gateway-web   NodePort   10.96.248.189   <none>        80:30080/TCP,5234:30081/TCP,5235:30082/TCP,5335:30083/TCP   2m16s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mcp-gateway   1/1     1            1           2m17s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/mcp-gateway-5d745d4f8b   1         1         1       2m17s
```

现在我们就将 MCP Gateway 部署到 Kubernetes 集群了。

接下来就可以上手体验 MCP Gateway 了。

### MCP Gateway 初体验

### 创建 MCP Server

访问 [http://127.0.0.1:30080/](http://127.0.0.1:30080/) 会跳转到登录页面，然后用户名和密码都是 `admin`，输入后就进入到管理控制台页面了：

![MCP Gateway 管理控制台](map-gateway-admin.png)

我们可以使用官方提供的 [https://github.com/mcp-ecosystem/mcp-gateway/blob/main/configs/mock-user-svc.yaml](https://github.com/mcp-ecosystem/mcp-gateway/blob/main/configs/mock-user-svc.yaml) 配置信息来创建一个 MPC Server。

![MCP Gateway 配置](mcp-gateway-config.png)

现在我们就得到了一个可用的 MCP Server：

![MCP 用户服务](mock-user-svc.png)

`mock-user-svc` 是 MCP Gateway 官方提供的用于测试用的 MCP Server，我们可以使用它来测试 MCP Gateway 运行状态是否正常。

在聊天窗口，我们可以选择 MCP Server `mock-user-svc`，并进行会话聊天：

![通过聊天创建用户](chat-create-user.png)

输入 `帮我创建用户 jianghushinian007@outlook.com`，大模型就会自动帮我们调用 MCP 提供的用户注册工具来注册用户。

我们还可以访问 [http://127.0.0.1:30083/users/email/jianghushinian007@outlook.com](http://127.0.0.1:30083/users/email/jianghushinian007@outlook.com) 来验证用户是否真的注册成功了。

![查看已创建用户](user.png)

#### 一键将 OpenAPI 转换为 MCP Server

其实 MCP Gateway 最核心的功能我们已经演示完成了，不过我最近发现它新增了 ` 导入 OpenAPI` 的按钮，这极大的方便了我们实现一键将 OpenAPI 转换为 MCP Server，而无需编写 MCP Server 配置文件。

你可以在 [https://editor.swagger.io/](https://editor.swagger.io/) 下载一个标准的 OpenAPI 3.0 文档，并将其保存为 `petstore.yaml`。

> https://github.com/jianghushinian/blog-go-example/blob/main/mcp-gateway/configs/petstore.yaml

然后，只需要通过拖拽的方式，将 `petstore.yaml` 文档导入 MCP Gateway 即可：

![OpenAPI 3.0 文档](openapi.png)

现在我们就得到了一个新的 MCP Server。

![OpenAPI 转换的 MCP Server](openapi-mcp-server.png)

当然，因为这个示例是使用了 Swagger 官方提供的示例文档，这个转换后的 MCP Server 暂时不可使用，因为后台没有真正的 HTTP 服务供我们调用。

但是，这个过程完整的演示了如何一键将 OpenAPI 转换为 MCP Server，可以说非常丝滑。你可以参照这个方式，尝试将公司现有的 HTTP 服务接口，转换成 MCP Server 供大模型使用。

MCP Gateway 剩下的能力就交给你自行去探索了。

### 总结

本文带大家体验了 MCP Gateway 的部署和使用，以及如何方便快捷的通关 MCP Gateway 一键将 OpenAPI 转换为 MCP Server，这极大的提升了我们的效率。

其实这个方案现在很多网关项目都在做，比如现在做的比较不错的阿里开源网关项目 Higress，这个项目已经非常成熟，如果有需要你可以调研一下。

今天之所以想介绍 MCP Gateway 项目，是因为这是一个纯用 Go 语言编写的轻量级 MCP 网关项目，非常适合我们学习原理。虽然本文没有讲解其实现原理，但是这个项目代码量并不多，你完全可以自己尝试阅读源码。甚至可以在此基础上进行魔改，并去掉无用组件，比如聊天功能，其实可以集成类似 MCP 官方提供的 [inspector](https://github.com/modelcontextprotocol/inspector) 组件来检查 MCP 服务的可用性，让网关项目回归简单，不必集成大模型的聊天功能。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/mcp-gateway) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ MCP Gateway 官方文档：[https://mcp.ifuryst.com/getting-started/quick-start](https://mcp.ifuryst.com/getting-started/quick-start)
+ MCP Gateway GitHub 源码：[https://github.com/mcp-ecosystem/mcp-gateway](https://github.com/mcp-ecosystem/mcp-gateway)
+ MCP 官方文档：[https://modelcontextprotocol.io/introduction](https://modelcontextprotocol.io/introduction)
+ MCP 官方 inspector 工具：[https://github.com/modelcontextprotocol/inspector](https://github.com/modelcontextprotocol/inspector)
+ MCP 市场：[https://mcp.so/](https://mcp.so/)
+ Swagger 官方示例文档网址：[https://editor.swagger.io/#/](https://editor.swagger.io/#/)
+ 使用 OpenAPI 构建 API 文档：[https://jianghushinian.cn/2023/02/12/build-api-documentation-using-openapi/](https://jianghushinian.cn/2023/02/12/build-api-documentation-using-openapi/)
+ kind 官方文档：[https://kind.sigs.k8s.io/docs/user/quick-start/#installation](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/mcp-gateway](https://github.com/jianghushinian/blog-go-example/tree/main/mcp-gateway)
+ 本文永久地址：[https://jianghushinian.cn/2025/05/14/mcp-gateway/](https://jianghushinian.cn/2025/05/14/mcp-gateway/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

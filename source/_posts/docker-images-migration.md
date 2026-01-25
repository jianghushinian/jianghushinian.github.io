---
title: '如何在两个镜像仓库之间迁移 Docker 跨平台镜像'
date: 2026-01-25 13:02:30
tags:
- Docker
categories:
- Docker
---

> 大家好，我是江湖十年。久违了，这是我的 2026 年第一篇技术文章。
>

Docker 镜像迁移的需求想必大家都有，比如因为众所周知的原因需要做镜像加速，将 Docker Hub 中的某个镜像上传到 Mirror 镜像站。今天来介绍下如何在两个镜像仓库之间迁移 Docker 跨平台镜像。

<!-- more -->

> **提示：**如果你对 Docker Manifest 以及 Buildx 不了解，可以参考我之前写过的文章：《[使用 docker manifest 构建跨平台镜像》](https://mp.weixin.qq.com/s/DsqD_iQTPGu1HjiF0czckg)，《[使用 buildx 构建跨平台镜像](https://mp.weixin.qq.com/s/OvOIjj4ser-QWd2BsfsY-Q)》。
>

### 单平台镜像迁移
如果是非跨平台镜像迁移，则非常简单。

1. 拉取源镜像：先通过 `docker pull <IMAGE>` 命令将 Docker Hub 中的镜像拉到本地（或中转）主机。
2. 重命名标签：然后使用 `docker tag <SOURCE_IMAGE> <TARGET_IMAGE>` 将镜像标记为目标仓库的地址。
3. 推送到目标仓库：执行 `docker push <TARGET_IMAGE>` 将镜像上传至私有仓库。
4. 清理本地缓存（可选）：迁移完成后，建议使用 `docker rmi <IMAGE>` 命令删除本地残留的镜像，以释放磁盘空间。

> **提示：** 在执行第 3 步之前，请确保已通过 `docker login` 完成目标仓库的身份验证。对于生产环境，建议在标签（Tag）中明确标注版本号，避免使用不稳定的 `latest` 标签。
>

这个方案在处理单架构镜像时简单高效，但存在一个关键的局限性：`docker pull <IMAGE>` 默认只会拉取与当前主机 CPU 架构相匹配的镜像。

随着 ARM64 架构在生产环境的广泛应用，传统的“本地中转”模式已无法满足现代云原生架构的需求。如果我们在 x86 机器上进行迁移，最终分发的镜像将无法在 ARM 节点上运行。

为了解决这一痛点，我们需要一种能够同时迁移多种架构（amd64、arm64 等）并保持其 Manifest List（清单列表）关联性的优化方案。于是，就有了迁移跨平台镜像的需求。

### 跨平台镜像迁移
本小节演示以 Docker Hub 中的 [golang:1.25-alpine](https://hub.docker.com/_/golang/tags?name=1.25-alpine) 镜像为例，演示将其迁移至阿里云私有镜像仓库。

#### 方案一：使用 Docker Buildx
Docker 早在很久以前就开始推行 Buildx 构建跨平台镜像，非常方便。

1. 初始化构建器（若尚未配置）

```bash
$ docker buildx create --name mybuilder --use
$ docker buildx inspect --bootstrap
```

2. 执行一键迁移命令

```bash
# 变量定义，私有仓库地址
TARGET="crpi-1ql0kmu5z0c9xt5q.cn-hangzhou.personal.cr.aliyuncs.com/jianghushinian/golang:1.25-alpine"

# 使用 buildx 同时构建并推送 amd64 和 arm64 架构
$ echo "FROM golang:1.25-alpine" | docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ${TARGET} \
  --push -
```

只需要一行简单的命令，即可实现同时构建并推送 amd64 和 arm64 架构镜像到私有仓库。此外，这里使用了取巧的办法在不创建创建物理 Dockerfile 情况下，直接通过标准输入传入内容。

当然，还有更简单的方式：

```bash
# 变量定义
SOURCE="golang:1.25-alpine"
TARGET="crpi-1ql0kmu5z0c9xt5q.cn-hangzhou.personal.cr.aliyuncs.com/jianghushinian/golang:1.25-alpine"

# 直接创建并推送新的 manifest
$ docker buildx imagetools create --tag ${TARGET} ${SOURCE}
```

你甚至连 buildx 构建器都无需创建，直接将跨平台镜像从一个仓库迁移至另一个仓库。

使用 `docker buildx build` 和  `docker buildx imagetools` 迁移镜像的区别是什么呢？我总结了如下表格供你参考：

| 特性 | docker buildx build | docker buildx imagetools |
| --- | --- | --- |
| 操作本质 | 重新构建 (Rebuilding) | 清单操作 (Manifest Manipulation) |
| 流量消耗 | 高，需要下载和重新上传所有镜像层 | 极低，仅传输几 KB 的元数据 JSON |
| 性能/速度 | 取决于网络带宽和镜像大小 | 秒级完成 |
| 多架构处理 | 需要通过 `--platform` 指定并逐一处理 | 自动合并源镜像中的所有平台架构 |
| 内容修改 | 可修改，可以更换基础镜像或增删文件 | 不可修改，源镜像是什么，目标就是什么 |
| 依赖环境 | 需要运行中的 BuildKit 实例（Builder） | 仅需能连接两个 Registry 的 API |


`docker buildx imagetools` 命令有个很大的优点，就是镜像不经过本机，也就是说执行完成后，本机是没有目标镜像的。

如果你的 Docker 版本比较老旧，没有 buildx 支持，那么可以选择更加保守的 `pull/push` 命令。

#### 方案二：使用传统 Docker Pull/Push
当然，在某些场景下，最原始的方式反而是最有效的。通过 `docker pull/push` 命令再结合 `docker manifest` 我们同样可以实现跨平台镜像的迁移。

步骤如下：

```bash
# 变量定义
VERSION="1.25-alpine"
SOURCE_IMAGE="golang:${VERSION}"
TARGET_REPO="crpi-1ql0kmu5z0c9xt5q.cn-hangzhou.personal.cr.aliyuncs.com/jianghushinian/golang"

# 1. 处理 amd64 架构
$ docker pull --platform=linux/amd64 ${SOURCE_IMAGE}
$ docker tag ${SOURCE_IMAGE} ${TARGET_REPO}:${VERSION}-amd64
$ docker push ${TARGET_REPO}:${VERSION}-amd64

# 2. 处理 arm64 架构
$ docker pull --platform=linux/arm64 ${SOURCE_IMAGE}
$ docker tag ${SOURCE_IMAGE} ${TARGET_REPO}:${VERSION}-arm64
$ docker push ${TARGET_REPO}:${VERSION}-arm64

# 3. 创建多架构 Manifest
# 这会将 amd64 和 arm64 两个镜像关联到同一个标签 ${VERSION} 下
$ docker manifest create \
  ${TARGET_REPO}:${VERSION} \
  ${TARGET_REPO}:${VERSION}-amd64 \
  ${TARGET_REPO}:${VERSION}-arm64

# 4. 推送 Manifest 到阿里云
$ docker manifest push ${TARGET_REPO}:${VERSION}
```

这没什么好解释的，纯体力活。

#### 方案选型
其实使用那种方式都行，能够达到目的即可。但有些时候，封装更好的工具可能有一些限制。

比如使用 `docker buildx imagetools` 命令迁移镜像，你可能遇到如下错误：

```bash
ERROR: failed commit on ref "layer-sha256:4bd07550e32f74fc2f29bbf38b34a2138634737d53907ad72ea1f4f7923129de": unexpected size 0, expected 126
```

这通常与 Registry 的底层实现有关。所以有些功能并不支持。那么 `docker buildx build` 或者 `docker manifest` 都是保底的解决方案。

甚至，你还可以选择第三方软件，如 Skopeo / Crane 等，他们都支持一行命令迁移，对 CI/CD 也更友好。

**没有最好的工具，只有最适合场景的策略。** 自动化工具是提高效率的利器，但理解 `docker manifest` 这种底层逻辑，才是你解决突发故障、确保镜像在多架构环境下准时上线的核心底气。

### 总结
在本篇文章中，我们探讨了三种不同维度的路径实现跨平台镜像迁移：

+ 追求极致效率：首选 `docker buildx imagetools`，利用“逻辑同步”实现秒级迁移，且不占用本地带宽与磁盘。
+ 确保物理落地：使用 `docker buildx build` 通过 BuildKit 重新封装，是应对复杂网络环境和 Registry 兼容性问题的强力手段。
+ 回归底层原理：通过传统的 `pull/push` 结合 `docker manifest` 手动操作，虽然步骤略显繁琐，但它是理解镜像分发机制的基石，更是故障排查时的终极“保底方案”。

至于第三方软件的方案，你可以自行尝试。

希望此文能对你有所启发。

**P.S.**

很久没写文章了，一个原因是忙，另一个原因是感觉现在 AI 已经如此强大了，有些文章确实没必要花心思写了。比如这篇文章，里面提到的方案，我想你通过 AI 也能很容易搜到答案，甚至让 AI 来写这样一篇文章，可能一分钟就搞定了。那我为什么还要写下这篇文章呢？

我能保证文中出现的每一个代码片段，都是我执行验证过的，这是一个真实人能给出的确定性。里面还会夹杂一些我个人对某个问题的看法。

但不得不承认，一篇文章能给读者带来的价值越来越低了，反而可能会作为模型的语料最终被训练进去。通过另外一种方式，与未来的读者见面。

也许，未来的哪一天，一篇技术文章最大的价值，可能仅剩下作为个人笔记而存在了。

**延伸阅读**

+ Docker Hub Go 镜像：[https://hub.docker.com/_/golang/tags?name=1.25-alpine](https://hub.docker.com/_/golang/tags?name=1.25-alpine)
+ 使用 docker manifest 构建跨平台镜像：[https://mp.weixin.qq.com/s/DsqD_iQTPGu1HjiF0czckg](https://mp.weixin.qq.com/s/DsqD_iQTPGu1HjiF0czckg)
+ 使用 buildx 构建跨平台镜像：[https://mp.weixin.qq.com/s/OvOIjj4ser-QWd2BsfsY-Q](https://mp.weixin.qq.com/s/OvOIjj4ser-QWd2BsfsY-Q)
+ 本文永久地址：[https://jianghushinian.cn/2026/01/25/docker-images-migration/](https://jianghushinian.cn/2026/01/25/docker-images-migration/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

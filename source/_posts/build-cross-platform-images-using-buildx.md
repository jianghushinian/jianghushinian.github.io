---
title: 使用 buildx 构建跨平台镜像
date: 2023-04-15 22:03:40
tags:
- Docker
categories:
- Docker
---

构建跨平台镜像是 Docker 生态系统中的一个重要话题，因为跨平台镜像可以在多种平台上运行，极具灵活性。为了实现这个目标，Docker 社区提供了多种方式来构建跨平台镜像，其中之一是使用 `docker manifest`，我在[《使用 docker manifest 构建跨平台镜像》](https://jianghushinian.cn/2023/04/07/build-cross-platform-images-using-docker-manifest/)一文中详细介绍了这种方法。然而，目前最流行的方式是使用 Docker 的 `buildx` 工具，这种方式不仅可以轻松构建跨平台镜像，还可以自动化整个构建过程，大大提高了效率。在本文中，我们将重点介绍使用 `buildx` 构建跨平台镜像的方法和技巧。

<!-- more -->

### 简介

`buildx` 是 Docker 官方提供的一个构建工具，它可以帮助用户快速、高效地构建 Docker 镜像，并支持多种平台的构建。使用 `buildx`，用户可以在单个命令中构建多种架构的镜像，例如 x86 和 ARM 架构，而无需手动操作多个构建命令。此外，`buildx` 还支持 Dockerfile 的多阶段构建和缓存，这可以大大提高镜像构建的效率和速度。

### 安装

`buildx` 是一个管理 Docker 构建的 CLI 插件，底层使用 [BuildKit](https://docs.docker.com/build/buildkit/) 扩展了 Docker 构建功能。

> 笔记：`BuildKit` 是 Docker 官方提供的一个高性能构建引擎，可以用来替代 Docker 原有的构建引擎。相比于原有引擎，`BuildKit` 具有更快的构建速度、更高的并行性、更少的资源占用和更好的安全性。

要安装并使用 `buildx`，需要 Docker Engine 版本号大于等于 19.03。

如果你使用的是 Docker Desktop，则默认安装了 `buildx`。可以使用 `docker buildx version` 命令查看安装版本，得到以下类似输出，证明已经安装过了。

```bash
$ docker buildx version
github.com/docker/buildx v0.9.1 ed00243a0ce2a0aee75311b06e32d33b44729689
```

如果需要手动安装，可以从 GitHub 发布页面[下载](https://github.com/docker/buildx/releases/tag/v0.10.4)对应平台的最新二进制文件，重命名为 `docker-buildx`，然后将其放到 Docker 插件目录下（Linux/Mac 系统为 `$HOME/.docker/cli-plugins`，Windows 系统为 `%USERPROFILE%\.docker\cli-plugins`）。

Linux/Mac 系统还需要给插件增加可执行权限 `chmod +x ~/.docker/cli-plugins/docker-buildx`，之后就可以使用 `buildx` 了。

更详细的安装过程可以参考[官方文档](https://docs.docker.com/build/install-buildx/)。

### 构建跨平台镜像

首先，需要澄清的是，本文中所提到的「跨平台镜像」这一说法并不十分准确。实际上，Docker 官方术语叫 `Multi-platform images` 即「多平台镜像」，意思是支持多种不同 CPU 架构的镜像。之所以使用「跨平台镜像」这一术语，是因为从使用者的角度来看，在使用如 `docker pull`、`docker run` 等命令来拉取和启动容器时，并不会感知到这个镜像是一个虚拟的 `manifest list` 镜像还是针对当前平台的镜像。 

> 笔记：`manifest list` 是通过指定多个镜像名称创建的镜像列表，是一个虚拟镜像，它包含了多个不同平台的镜像信息。可以像普通镜像一样使用 `docker pull` 和 `docker run` 等命令来操作它。如果你想了解关于 `manifest list` 的更多信息，可参考[《使用 docker manifest 构建跨平台镜像》](https://jianghushinian.cn/2023/04/07/build-cross-platform-images-using-docker-manifest/)一文。

#### 跨平台镜像构建策略

`builder` 支持三种不同策略构建跨平台镜像：

1. 在内核中使用 [QEMU](https://zh.wikipedia.org/wiki/QEMU) 仿真支持。

如果你正在使用 Docker Desktop，则已经支持了 QEMU，QEMU 是最简单的构建跨平台镜像策略。它不需要对原有的 `Dockerfile` 进行任何更改，`BuildKit` 会通过 [binfmt_misc](https://zh.wikipedia.org/wiki/Binfmt_misc) 这一 Linux 内核功能实现跨平台程序的执行。

工作原理：

QEMU 是一个处理器模拟器，可以模拟不同的 CPU 架构，我们可以把它理解为是另一种形式的虚拟机。在 `buildx` 中，QEMU 用于在构建过程中执行非本地架构的二进制文件。例如，在 x86 主机上构建一个 ARM 镜像时，QEMU 可以模拟 ARM 环境并运行 ARM 二进制文件。

binfmt_misc 是 Linux 内核的一个模块，它允许用户注册可执行文件格式和相应的解释器。当内核遇到未知格式的可执行文件时，会使用 binfmt_misc 查找与该文件格式关联的解释器（在这种情况下是 QEMU）并运行文件。

QEMU 和 binfmt_misc 的结合使得通过 `buildx` 跨平台构建成为可能。这样我们就可以在一个架构的主机上构建针对其他架构的 Docker 镜像，而无需拥有实际的目标硬件。

虽然 Docker Desktop 预配置了 binfmt_misc 对其他平台的支持，但对于其他版本 Docker，你可能需要使用 `tonistiigi/binfmt` 镜像启动一个特权容器来进行支持：

```bash
$ docker run --privileged --rm tonistiigi/binfmt --install all
```

2. 使用相同的构建器实例在多个本机节点上构建。

此方法直接在对应平台的硬件上构建镜像，所以需要准备各个平台的主机。因为此方法门槛比较高，所以并不常使用。

3. 使用 Dockerfile 中的多阶段构建，交叉编译到不同的平台架构中。

交叉编译的复杂度不在于 Docker，而是取决于程序本身。比如 Go 程序就很容易实现交叉编译，只需要在使用 `go build` 构建程序时指定 `GOOS`、`GOARCH` 两个环境变量即可实现。

#### 创建 `builder`

要使用 `buildx` 构建跨平台镜像，我们需要先创建一个 `builder`，可以翻译为「构建器」。

使用 `docker buildx ls` 命令可以查看 `builder` 列表：

```bash
$ docker buildx ls
NAME/NODE       DRIVER/ENDPOINT STATUS  BUILDKIT PLATFORMS
default *       docker
  default       default         running 20.10.21 linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
desktop-linux   docker
  desktop-linux desktop-linux   running 20.10.21 linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
```

这两个是默认 `builder`，`default *` 中的 `*` 表示当前正在使用的 `builder`，当我们运行 `docker build` 命令时就是在使用此 `builder` 构建镜像。

可以发现，这两个默认的 `builder` 第二列 `DRIVER/ENDPOINT` 项的值都是 `docker`，表示它们都使用 `docker` 驱动程序。

`buildx` 支持以下几种驱动程序：

|  驱动   | 说明  |
|  ----  | ----  |
| docker | 使用捆绑到 Docker 守护进程中的 BuildKit 库，就是安装 Docker 后默认的 BuildKit。 |
| docker-container | 使用 Docker 新创建一个专用的 BuildKit 容器。 |
| kubernetes | 在 Kubernetes 集群中创建一个 BuildKit Pod。 |
| remote | 直接连接到手动管理的 BuildKit 守护进程。 |

默认的 `docker` 驱动程序优先考虑简单性和易用性，所以它对缓存和输出格式等高级功能的支持有限，并且不可配置。其他驱动程序则提供了更大的灵活性，并且更擅长处理高级场景。

具体差异你可以到[官方文档](https://docs.docker.com/build/drivers/)中查看。

因为使用 `docker` 驱动程序的默认 `builder` 不支持使用单条命令（默认 `builder` 的 `--platform` 参数只接受单个值）构建跨平台镜像，所以我们需要使用 `docker-container` 驱动创建一个新的 `builder`。

命令语法如下：

```bash
$ docker buildx create --name=<builder-name> --driver=<driver> --driver-opt=<driver-options>
```

参数含义如下：

- `--name`：构建器名称，必填。

- `--driver`：构建器驱动程序，默认为 `docker-container`。

- `--driver-opt`：驱动程序选项，如选项 `--driver-opt=image=moby/buildkit:v0.11.3` 可以安装指定版本的 `BuildKit`，默认值是 [moby/buildkit](https://hub.docker.com/r/moby/buildkit)。

更多可选参数可以参考[官方文档](https://docs.docker.com/engine/reference/commandline/buildx_create/#options)。

我们可以使用如下命令创建一个新的 `builder`：

```bash
$ docker buildx create --name mybuilder
mybuilder
```

再次查看 `builder` 列表：

```bash
$ docker buildx ls
NAME/NODE       DRIVER/ENDPOINT             STATUS   BUILDKIT PLATFORMS
mybuilder *     docker-container
  mybuilder0    unix:///var/run/docker.sock inactive
default         docker
  default       default                     running  20.10.21 linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
desktop-linux   docker
  desktop-linux desktop-linux               running  20.10.21 linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
```

可以发现选中的构建器已经切换到了 `mybuilder`，如果没有选中，你需要手动使用 `docker buildx use mybuilder` 命令切换构建器。

#### 启动 `builder`

我们新创建的 `mybuilder` 当前状态为 `inactive`，需要启动才能使用。

```bash
$ docker buildx inspect --bootstrap mybuilder
[+] Building 16.8s (1/1) FINISHED
 => [internal] booting buildkit                                                                                                                                  16.8s
 => => pulling image moby/buildkit:buildx-stable-1                                                                                                               16.1s
 => => creating container buildx_buildkit_mybuilder0                                                                                                              0.7s
Name:   mybuilder
Driver: docker-container

Nodes:
Name:      mybuilder0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Buildkit:  v0.9.3
Platforms: linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6
```

`inspect` 子命令用来检查构建器状态，使用 `--bootstrap` 参数则可以启动 `mybuilder` 构建器。

再次查看 `builder` 列表，`mybuilder` 状态已经变成了 `running`。

```bash
$ docker buildx ls
NAME/NODE       DRIVER/ENDPOINT             STATUS  BUILDKIT PLATFORMS
mybuilder *     docker-container
  mybuilder0    unix:///var/run/docker.sock running v0.9.3   linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6
default         docker
  default       default                     running 20.10.21 linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
desktop-linux   docker
  desktop-linux desktop-linux               running 20.10.21 linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
```

其中 `PLATFORMS` 一列所展示的值 `linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6` 就是当前构建器所支持的所有平台了。

现在使用 `docker ps` 命令可以看到 `mybuilder` 构建器所对应的 `BuildKit` 容器已经启动。

```bash
$ docker ps
CONTAINER ID   IMAGE                           COMMAND                  CREATED         STATUS         PORTS                                NAMES
b8887f253d41   moby/buildkit:buildx-stable-1   "buildkitd"              4 minutes ago   Up 4 minutes                                        buildx_buildkit_mybuilder0
```

这个容器就是辅助我们构建跨平台镜像用的，不要手动删除它。

#### 使用 `builder` 构建跨平台镜像

现在一些准备工作已经就绪，我们终于可以使用 `builder` 构建跨平台镜像了。

这里以一个 Go 程序为例，来演示如何构建跨平台镜像。

`hello.go` 程序如下：

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Printf("Hello, %s/%s!\n", runtime.GOOS, runtime.GOARCH)
}
```

这个程序非常简单，执行后打印 `Hello, 操作系统/CPU 架构`。

Go 程序还需要一个 `go.mod` 文件：

```
module hello

go 1.20
```

编写 `Dockerfile` 内容如下：

```dockerfile
FROM golang:1.20-alpine AS builder
WORKDIR /app
ADD . .
RUN go build -o hello .

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/hello .
CMD ["./hello"]
```

这是一个普通的 `Dockerfile` 文件，为了减小镜像大小，使用了多阶段构建。它跟构建仅支持当前平台的镜像所使用的 `Dockerfile` 没什么两样。

```bash
$ ls
Dockerfile go.mod     hello.go
```

以上三个文件需要放在同一个目录下，然后就可以在这个目录下使用 `docker buildx` 来构建跨平台镜像了。

```bash
$ docker buildx build --platform linux/arm64,linux/amd64 -t jianghushinian/hello-go .
```

`docker buildx build` 语法跟 `docker build` 一样，`--platform` 参数表示构建镜像的目标平台，`-t` 表示镜像的 Tag，`.` 表示上下文为当前目录。

唯一不同的是对 `--platform` 参数的支持，`docker build` 的 `--platform` 参数只支持传递一个平台信息，如 `--platform linux/arm64`，也就是一次只能构建单个平台的镜像。

而使用 `docker buildx build` 构建镜像则支持同时传递多个平台信息，中间使用英文逗号分隔，这样就实现了只用一条命令便可以构建跨平台镜像的功能。

执行以上命令后，我们将会得到一条警告：

```log
WARNING: No output specified with docker-container driver. Build result will only remain in the build cache. To push result image into registry use --push or to load image into docker use --load
```

这条警告提示我们没有为 `docker-container` 驱动程序指定输出，生成结果将只会保留在构建缓存中，使用 `--push` 可以将镜像推送到 Docker Hub 远程仓库，使用 `--load` 可以将镜像保存在本地。

这是因为我们新创建的 `mybuilder` 是启动了一个容器来运行 `BuildKit`，它并不能直接将构建好的跨平台镜像输出到本机或推送到远程，必须要用户来手动指定输出位置。

我们可以尝试指定 `--load` 将镜像保存的本地主机。

```bash
$ docker buildx build --platform linux/arm64,linux/amd64 -t jianghushinian/hello-go . --load
[+] Building 0.0s (0/0)
ERROR: docker exporter does not currently support exporting manifest lists
```

结果会得到一条错误日志。看来它并不支持直接将跨平台镜像输出到本机，这其实是因为传递了多个 `--platform` 的关系，如果 `--platform` 只传递了一个平台，则可以使用 `--load` 将构建好的镜像输出到本机。

那么我们就只能通过 `--push` 参数将跨平台镜像推送到远程仓库了。不过在此之前需要确保使用 `docker login` 完成登录。

```bash
$ docker buildx build --platform linux/arm64,linux/amd64 -t jianghushinian/hello-go . --push
```

现在登录 Docker Hub 就可以看见推送上来的跨平台镜像了。

![jianghushinian/hello-go](hello-go.png)

我们也可以使用 `imagetools` 来检查跨平台镜像的 `manifest` 信息。

```bash
$ docker buildx imagetools inspect jianghushinian/hello-go
Name:      docker.io/jianghushinian/hello-go:latest
MediaType: application/vnd.docker.distribution.manifest.list.v2+json
Digest:    sha256:51199dadfc55b23d6ab5cfd2d67e38edd513a707273b1b8b554985ff562104db

Manifests:
  Name:      docker.io/jianghushinian/hello-go:latest@sha256:8032a6f23f3bd3050852e77b6e4a4d0a705dfd710fb63bc4c3dc9d5e01c8e9a6
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/arm64

  Name:      docker.io/jianghushinian/hello-go:latest@sha256:fd46fd7e93c7deef5ad8496c2cf08c266bac42ac77f1e444e83d4f79d58441ba
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/amd64
```

可以看到，这个跨平台镜像包含了两个目标平台的镜像，分别是 `linux/arm64` 和 `linux/amd64`。

我们分别在 Apple M2 芯片平台和 Linux x86 平台来启动这个 Docker 镜像看下输出结果。

```bash
$ docker run --rm jianghushinian/hello-go
Hello, linux/arm64!
```

```bash
$ docker run --rm jianghushinian/hello-go
Hello, linux/amd64!
```

至此，我们使用 `builder` 完成了跨平台镜像的构建。

#### 使用交叉编译

以上演示的构建跨平台镜像过程就是利用 QEMU 的能力，因为 Go 语言的交叉编译非常简单，所以我们再来演示一下如何使用交叉编译来构建跨平台镜像。

我们只需要对 `Dockerfile` 文件进行修改：

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.20-alpine AS builder
ARG TARGETOS
ARG TARGETARCH
WORKDIR /app
ADD . .
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o hello .

FROM --platform=$TARGETPLATFORM alpine:latest
WORKDIR /app
COPY --from=builder /app/hello .
CMD ["./hello"]
```

其中 `BUILDPLATFORM`、`TARGETOS`、`TARGETARCH`、`TARGETPLATFORM` 四个变量是 `BuildKit` 提供的全局变量，分别表示构建镜像所在平台、操作系统、架构、构建镜像的目标平台。

在构建镜像时，`BuildKit` 会将当前所在平台信息传递给 `Dockerfile` 中的 `BUILDPLATFORM` 参数（如 `linux/arm64`）。

通过 `--platform` 参数传递的 `linux/arm64,linux/amd64` 镜像目标平台列表会依次传递给 `TARGETPLATFORM` 变量。

而 `TARGETOS`、`TARGETARCH` 两个变量在使用时则需要先通过 `ARG` 进行声明，`BuildKit` 会自动为其赋值。

在 Go 程序进行编译时，可以通过 `GOOS` 环境变量指定目标操作系统，通过 `GOARCH` 环境变量指定目标架构。

所以这个 `Dockerfile` 所表示的含义是：首先拉取当前构建镜像所在平台的 `golang` 镜像，然后使用交叉编译构建目标平台的 Go 程序，最后将构建好的 Go 程序复制到目标平台的 `alpine` 镜像。

最终我们会通过交叉编译得到一个跨平台镜像。

> 笔记：通过 `FROM --platform=$BUILDPLATFORM image` 可以拉取指定平台的镜像，由此我们可以知道，其实 [golang](https://hub.docker.com/_/golang/tags) 和 [alpine](https://hub.docker.com/_/alpine/tags) 镜像都是支持跨平台的。

构建镜像命令不变：

```bash
$ docker buildx build --platform linux/arm64,linux/amd64 -t jianghushinian/hello-cross-go . --push
```

启动镜像后输出结果不变：

```bash
$ docker run --rm jianghushinian/hello-cross-go
Hello, linux/arm64!
```

```bash
$ docker run --rm jianghushinian/hello-cross-go
Hello, linux/amd64!
```

至此，我们利用 Go 语言的交叉编译完成了跨平台镜像的构建。

#### 平台相关的全局变量

关于上面提到的几个全局变量，`BuildKit` 后端预定义了一组 ARG 全局变量（共 8 个）可供使用，其定义和说明如下：

|  变量   | 说明  |
|  ----  | ----  |
| TARGETPLATFORM  | 构建镜像的目标平台，如：`linux/amd64`，`linux/arm/v7`，`windows/amd64`。 |
| TARGETOS  | TARGETPLATFORM 的操作系统，如：`linux`、`windows`。 |
| TARGETARCH  | TARGETPLATFORM 的架构类型，如：`amd64`、`arm`。 |
| TARGETVARIANT  | TARGETPLATFORM 的变体，如：`v7`。 |
| BUILDPLATFORM  | 执行构建所在的节点平台。 |
| BUILDOS  | BUILDPLATFORM 的操作系统。 |
| BUILDARCH  | BUILDPLATFORM 的架构类型。 |
| BUILDVARIANT  | BUILDPLATFORM 的变体。 |

使用示例如下：

```dockerfile
# 这里可以直接使用 TARGETPLATFORM 变量
FROM --platform=$TARGETPLATFORM alpine

# 稍后的 RUN 命令想要使用变量必须提前用 ARG 进行声明
ARG TARGETPLATFORM

RUN echo "I'm building for $TARGETPLATFORM"
```

#### 删除 `builder`

我们已经实现了使用 `builder` 构建跨平台镜像。如果现在你想要恢复环境，删除新建的 `builder`。则可以使用 `docker buildx rm mybuilder` 命令来完成。

```bash
$ docker buildx rm mybuilder
mybuilder removed
```

跟随 `mybuilder` 启动的 `buildx_buildkit_mybuilder0` 容器也会随之被删除。

现在再使用 `docker buildx ls` 命令查看构建器列表，已经恢复成原来的样子了。

```bash
$ docker buildx ls
NAME/NODE       DRIVER/ENDPOINT STATUS  BUILDKIT PLATFORMS
default *       docker
  default       default         running 20.10.21 linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
desktop-linux   docker
  desktop-linux desktop-linux   running 20.10.21 linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
```

### 功能清单

除了前文介绍的几个 `buildx` 常用命令，更多功能可以通过 `--help` 参数进行查看。

```bash
$ docker buildx --help

Usage:  docker buildx [OPTIONS] COMMAND

Extended build capabilities with BuildKit

Options:
      --builder string   Override the configured builder instance

Management Commands:
  imagetools  Commands to work on images in registry

Commands:
  bake        Build from a file
  build       Start a build
  create      Create a new builder instance
  du          Disk usage
  inspect     Inspect current builder instance
  ls          List builder instances
  prune       Remove build cache
  rm          Remove a builder instance
  stop        Stop builder instance
  use         Set the current builder instance
  version     Show buildx version information

Run 'docker buildx COMMAND --help' for more information on a command.
```

如 `stop`、`rm` 可以管理 `builder` 的生命周期。每条子命令又可以使用 `docker buildx COMMAND --help` 方式查看使用帮助，感兴趣的同学可以自行学习。

### 总结

本文讲解了如何使用 `buildx` 构建跨平台镜像，这也是在 Docker 生态中目前最优的构建方式。

首先介绍了 `buildx` 是什么，以及如何安装。接下来就进入了构建跨平台镜像的讲解，我们分析了三种跨平台镜像构建策略，并且分别对 QEMU 和 交叉编译两种策略进行了演示。QEMU 策略无需对 `Dockerfile` 做任何更改，而使用交叉编译方式则需要根据程序的支持来编写 `Dockerfile` 构建跨平台应用。

最后我们还讲解了如何管理 `buildx` 的生命周期，以及罗列了 `buildx` 的功能清单帮助你进一步深入学习。

**参考**

- buildx 仓库地址：https://github.com/docker/buildx
- buildx 安装文档：https://docs.docker.com/build/install-buildx/
- buildx 文档：https://docs.docker.com/engine/reference/commandline/buildx/
- buildx 驱动程序：https://docs.docker.com/build/drivers/
- 多平台镜像：https://docs.docker.com/build/building/multi-platform/
- 多阶段构建：https://docs.docker.com/build/building/multi-stage/
- buildkit 文档：https://docs.docker.com/build/buildkit/
- buildkit 支持的全局可用变量：https://docs.docker.com/engine/reference/builder/#automatic-platform-args-in-the-global-scope

---
title: 如何设计一个优秀的 Go Web 项目目录结构
date: 2023-02-25 08:48:22
tags:
- Golang
- Web
categories:
- Web
---

Go 语言作为一门高效、简洁、并发安全的语言，越来越受到开发者们的青睐，特别是在 Web 开发及云原生领域。而对于一个大型的 Go Web 项目而言，一个优秀的目录结构设计是必不可少的。它可以帮助我们更好地组织代码、减少冗余、提高可维护性和可扩展性。在本文中，我们将讨论如何设计一个优秀的 Go Web 项目目录结构。

<!-- more -->

### 基本原则

在开始设计项目目录结构之前，我们需要先了解一些设计目录的基本原则：

- 可读性和可维护性：设计目录结构应该易于阅读和维护，目录名称要简洁、清晰，最好能达到顾名思义的效果。

- 可扩展性和模块化：设计目录结构应该易于扩展和模块化，随着时间的推移，项目会不断变大，项目的目录结构应该能够很容易支撑这种变化。

- 规范性和一致性：设计目录结构应该遵循规范和一致性，如无特殊情况，目录名称最好统一使用单数形式（特殊情况可以打破，如 `/docs`、`/examples`）。

### Go 项目标准布局

早期的 Go 项目目录结构设计五花八门，因为大多数 Go 开发者都有其他编程语言基础，比如 Java、Python 等，这些开发者在设计 Go 项目时，目录结构往往会携带一些其他编程语言、框架所惯用的风格。

但很多情况下，直接套用其他编程语言风格的目录所设计出来的 Go 项目都会有很多不合理之处，为了改变现状，慢慢的 Go 语言社区诞生了很多更符合 Go 哲学的项目目录风格。而这其中，最著名的项目当属 [golang-standards/project-layout](https://github.com/golang-standards/project-layout)。

 `golang-standards/project-layout` 是一个 Go 社区维护的 Go 项目目录结构标准，它的目标是提供一种一致性的、易于理解和使用的目录结构，从而帮助开发者更好地组织和管理自己的代码。

该项目的目录结构基于功能和约定进行组织，通过提供一种标准化的目录结构，可以使得不同的项目之间具有一致的代码组织方式，便于开发者理解和使用。

该项目的目录结构包括以下几个部分：

#### Go 应用程序相关目录

##### `/cmd`

这个目录主要负责程序的启动、初始化、停止等功能，故主要包含项目的入口文件 `main.go`，如果一个项目有多个组件，则可以存放多个组件的 `main.go`，例如：

```bash
cmd
├── ctl
│   └── main.go
├── server
│   └── main.go
└── task
    └── main.go
```

不要在这个目录中放太多的代码，更不要放业务逻辑代码，保持整洁。

##### `/internal`

存放项目的内部私有代码和库，不允许在项目外部使用。同时这也是 Go 在编译时强制执行的校验规则，如果在其他项目中导入 `internal` 目录下的内容，Go(1.19) 在编译时会得到如下错误：

```bash
use of internal package xxx not allowed
```

> 注：`xxx` 为包含 `internal` 的包路径

在项目的目录树中的任意位置都可以有 `internal` 目录，而不仅仅是在顶级目录中。

在 `/internal` 内部可以增加额外的包结构来区分组件间共享和私有的内部代码：

```bash
internal
├── app
│   ├── ctl
│   ├── server
│   └── task
└── pkg
```

其中 `/internal/app` 下存放各个组件的逻辑代码，`/internal/pkg` 下存放各组件间的共享代码。

##### `/pkg`

包含可导出的公共库，可以被其他项目引用。这意味着此目录下的代码可以被导入任何其他项目，被当作库程序来使用，所以将代码放到此目录前要慎重考虑，不要将私有代码放到此目录下。

Travis Jeffery 撰写了[一篇文章](https://travisjeffery.com/b/2019/11/i-ll-take-pkg-over-internal/)讲解了 `pkg` 和 `internal` 目录使用建议，你可以进一步了解学习。

##### `/configs`

此目录存放配置文件模板或默认配置。

前文讲设计项目目录的基本原则时提到目录名最好使用单数形式，不过由于使用 `configs` 来存放配置已经是约定俗成的事实标准，故此目录名称可以打破这项设计原则。

##### `/test`

可以用来存放 e2e 测试和测试数据等。

对于较大的项目，有一个数据子目录更好一些。例如，如果需要 Go 在编译时忽略目录中的内容，则可以使用 `/test/data` 或 `/test/testdata` 这样的目录名称。

另外 Go 还会忽略以 `.` 或 `_` 开头的目录或文件，因此可以更具灵活性的来命名测试数据目录。

##### `deployments`

用来存放 IaaS、PaaS 系统和容器编排部署所需要的配置及模板（如：Docker-Compose，Kubernetes/Helm，Mesos，Terraform，Bosh）。

如果你的项目作为 Kubernetes 生态中的一员或使用 Kubernetes 部署，则建议命名为 `/deploy`，更加符合 Kubernetes 社区风格。

##### `/third_party`

外部辅助工具目录，fork 的代码和其他第三方工具（例如 Swagger UI）。比如我们修改了某个开源的第三方项目的代码，使其满足当前项目的使用需求，就可以将修改后的代码放到 `/third_party/fork` 目录下进行维护。

##### `/web`

如果你打算在项目目录下包含配套的前端程序代码，则可以存放到此目录。主要包括静态资源、前端代码、路由等。

如果你的项目仅提供 RESTful API，且前后端程序需要分开独立维护，则可以不需要此目录，建议将前端代码作为一个独立的项目存在。

#### 项目管理相关目录

##### `/init`

包含系统初始化（systemd、upstart、sysv）和进程管理（runit、supervisord）等配置。这在非容器化部署的项目中非常有用。

另外还可以包含初始化代码，如数据库迁移、缓存初始化等。

##### `/scripts`

存放用于执行各种构建、安装、分析等操作的脚本。

根文件 `/Makefile` 可以引用这些脚本，使其变得更小、更易于维护。

##### `/build`

存放程序构建和持续集成相关的文件。例如：

使用 `/build/package` 目录来存放云（AMI），容器（Docker），操作系统（deb，rpm，pkg）软件包配置和脚本。

使用 `/build/ci` 目录来存放 CI（travis、circle、drone）配置文件和脚本。

##### `/tools`

此项目的支持工具。这些工具可以从 `/pkg` 和 `/internal` 目录导入代码。

##### `/assets`

项目使用的其他资源 (Image、CSS、JavaScript、SQL 文件等)。

##### `/githooks`

Git 相关的钩子存放目录。

#### 项目文档相关目录

##### `/api`

当前项目对外暴露的 API 文档，如 OpenAPI/Swagger 规范文档、JSON Schema 文件、ProtoBuf 定义文件等。

```bash
api
└── openapi
    └── openapi.yaml
```

##### `/docs`

设计、开发和用户文档等（除 godoc 生成的文档）。

##### `/examples`

应用程序或公共库的示例。降低使用者的上手难度。

#### 不建议使用的目录

##### `/src`

一些有 Java 或 Python 开发经验的开发者习惯在项目中设计一个 `/src` 目录，但在 Go 语言中这是不推荐的做法。

早期的 Go 语言的项目都会被放置到 `$GOPATH/src` 目录下，如果项目中再有一个 `/src` 目录，那么项目最终的存放的路径就显得比较奇怪：

```bash
$GOPATH/src/your_project/src/your_code.go
```

因此，请不要在一个 Go 项目中设计 `/src` 目录。

#### 一些放在项目根目录下的文件

##### `/README.md`

`README.md` 是学习并使用项目的入口，是让用户了解项目的第一手资料。一个项目的 `README.md` 通常包含项目名称和简介、安装说明、使用说明、贡献方式、版权和许可等。

GitHub 也会默认解析 `README.md` 文件并渲染成 HTML 文档。

##### `/Makefile`

`Makefile` 是一个老牌的项目管理工具，建议在 Go 项目中都集成它。`Makefile` 语法可以参考 [跟我一起写 Makefile
](https://seisman.github.io/how-to-write-makefile/index.html)。

##### `/CHANGELOG`

用于存放项目的更新记录，如版本号、作者、更新内容等。如果嫌麻烦，还可以使用 [git-chglog](https://github.com/git-chglog/git-chglog) 或类似工具自动生成。

##### `/CONTRIBUTING.md`

如果你的项目是开源项目，则建议包含 `/CONTRIBUTING.md` 文件，用来说明如何贡献代码、项目规范等，让第三方开发者更容易参与进来。

##### `/LICENSE`

开源项目一定要包含 `/LICENSE`，即开源许可证。没有开源许可证的项目，严格来讲不叫开源项目，如何选择开源许可证可以参考我的另一篇文章 [开源协议简介](https://jianghushinian.cn/2023/01/15/open-source-license-introduction/)。

### 总结

经过上面的讲解，最终我们得到的项目目录结构如下：

```bash
project
├── CHANGELOG
├── CONTRIBUTING.md
├── LICENSE
├── Makefile
├── README.md
├── api
│   └── openapi
│       └── openapi.yaml
├── assets
├── build
├── cmd
│   ├── ctl
│   │   └── main.go
│   ├── server
│   │   └── main.go
│   └── task
│       └── main.go
├── configs
├── deployments
├── docs
├── examples
├── githooks
├── init
├── internal
│   ├── app
│   │   ├── ctl
│   │   ├── server
│   │   └── task
│   └── pkg
├── pkg
├── scripts
├── test
├── third_party
├── tools
└── web
```

此目录结构主要参考 `golang-standards/project-layout` 项目，和一些我自己的思考，最终总结出来。

通过使用以上提供的项目目录结构，可以帮助开发者更好地管理和组织自己的代码，提高代码的可维护性和可扩展性。

以上介绍的目录结构比较适合中大型项目，如果你的项目比较小，则可以只使用 `/cmd`、`/internal`、`/configs` 等几个少量目录，其他目录根据需要再来创建。如果你的项目足够小，甚至不需要什么目录，直接采用平铺式代码结构（将所有文件都放在项目根目录下）即可。

另外，随着技术的不断迭代发展，如 DDD 正在流行起来，Go 社区对项目目录结构的探索也仍在继续，在可预见的未来，一个优秀的 Go Web 项目目录结构的定义一定会被更新，期待下次能够分享更优秀的目录结构设计。

P.S.

本文部分内容由 ChatGPT 生成，体验科技乐趣 :)。

**参考**

> Standard Go Project Layout：https://github.com/golang-standards/project-layout
> Standard Go Project Layout 中文文档：https://github.com/golang-standards/project-layout/blob/master/README_zh-CN.md
> Go 面向包的设计和架构分层：https://github.com/danceyoung/paper-code/blob/master/package-oriented-design/packageorienteddesign.md
> pkg 和 internal 目录使用建议：https://travisjeffery.com/b/2019/11/i-ll-take-pkg-over-internal/

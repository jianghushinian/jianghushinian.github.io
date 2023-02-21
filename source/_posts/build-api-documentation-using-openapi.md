---
title: 使用 OpenAPI 构建 API 文档
date: 2023-02-12 16:18:26
tags:
- 文档
- Golang
categories:
- 文档
---

作为一名开发者，往往需要编写程序的 API 文档，尤其是 Web 后端开发者，在跟前端对接 HTTP 接口的时候，一个好的 API 文档能够大大提高协作效率，降低沟通成本，本文就来聊聊如何使用 OpenAPI 构建 HTTP 接口文档。

<!-- more -->

### OpenAPI

#### 什么是 OpenAPI

OpenAPI 是规范化描述 API 领域应用最广泛的行业标准，由 [OpenAPI Initiative(OAI)](https://www.openapis.org/) 定义并维护，同时也是 Linux 基金会下的一个开源项目。通常我们所说的 OpenAPI 全称应该是 OpenAPI Specification(OpenAPI 规范，简称 OSA)，它使用规定的格式来描述 HTTP RESTful API 的定义，以此来规范 RESTful 服务开发过程。使用 JSON 或 YAML 来描述一个标准的、与编程语言无关的 HTTP API 接口。OpenAPI 规范最初基于 SmartBear Software 在 2015 年捐赠的 [Swagger 规范](https://swagger.io/resources/open-api/)演变而来，目前最新的版本是 [v3.1.0](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.1.0.md)。

简单来说，OpenAPI 就是用来定义 HTTP 接口文档的一种规范，大家都按照同一套规范来编写接口文档，能够极大的减少沟通成本。

#### OpenAPI 规范基本信息

OpenAPI 规范内容包含非常多的细节，本文无法一一讲解，这里仅介绍常见的基本信息，以 YAML 为例进行说明。YAML 是 JSON 的超集，在 OpenAPI 规范中定义的所有语法，两者之间是可以互相转换的，如果手动编写，建议编写 YAML 格式，更为易读。

OpenAPI 文档编写在一个 `.json` 或 `.yaml` 中，推荐将其命名为 `openapi.json` 或 `openapi.yaml`，OpenAPI 文档其实就是一个单一的 JSON 对象，其中包含符合 [OpenAPI 规范](https://spec.openapis.org/oas/latest.html)中定义的结构字段。

OpenAPI 规范基本信息如下：

| 字段名 | 类型 | 描述 |
| :---- | :---- | :---- |
| openapi     | string | 必选，必须是 OpenAPI 已发布的合法版本，如 `3.0.1`。
| info        | object | 必选，此字段提供 API 相关的元数据（如描述、作者和联系信息）。
| servers     | array[object] | 这是一个 Server 对象的数组，提供到服务器的连接信息。
| paths       | object | 必选，API 提供的可用的路径和操作。
| components  | object | 一个包含多种结构的元素，可复用组件。
| security    | array  | 声明 API 使用的安全认证机制，目前支持 `HTTP Basic Auth`、`HTTP Bearer Auth`、`ApiKey Auth` 以及 `OAuth2`。
| tags        | array  | 提供标签可以为 API 归类，每个标签名都应该是唯一的。
| externalDocs| object | 附加的文档，可以通过扩展属性来扩展文档。

一个 YAML 格式的 OpenAPI 文档示例如下：

```yaml
openapi: 3.1.0
info:
  title: Tic Tac Toe
  description: |
    This API allows writing down marks on a Tic Tac Toe board
    and requesting the state of the board or of individual squares.
  version: 1.0.0 # 此为 API 接口文档版本，与 openapi 版本无关
tags:
  - name: Gameplay
paths:
  # Whole board operations
  /board:
    get:
      summary: Get the whole board
      description: Retrieves the current state of the board and the winner.
      tags:
        - Gameplay
      operationId: get-board
      responses:
        "200":
          description: "OK"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/status"

  # Single square operations
  /board/{row}/{column}:
    parameters:
      - $ref: "#/components/parameters/rowParam"
      - $ref: "#/components/parameters/columnParam"
    get:
      summary: Get a single board square
      description: Retrieves the requested square.
      tags:
        - Gameplay
      operationId: get-square
      responses:
        "200":
          description: "OK"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/mark"
        "400":
          description: The provided parameters are incorrect
          content:
            text/html:
              schema:
                $ref: "#/components/schemas/errorMessage"
              example: "Illegal coordinates"
...
```

以上示例完整文档[在此](https://oai.github.io/Documentation/examples/tictactoe.yaml)，具体语法我就不在这里介绍了。如果你开发过 API 接口，相信能看懂文档大部分内容所代表的含义。**不必完全掌握其语法，这并不会对阅读本文接下来的内容造成困扰**，因为稍后我会介绍如何通过代码注释的方式自动生成此文档。

如果你想手动编写 OpenAPI 文档，那么我还是推荐你阅读下 [OpenAPI 规范](https://spec.openapis.org/oas/latest.html)，这里有一份[中文版的规范](https://openapi.apifox.cn/)。阅读规范是一个比较枯燥的过程，如果你没有耐心读完，强烈建议阅读 [OpenAPI 规范入门](https://oai.github.io/Documentation/)，相较于完整版的规范要精简得多，并且讲解更加易于理解。

另外还推荐访问 [OpenAPI Map](https://openapi-map.apihandyman.io/) 网站来掌握 OpenAPI 规范，该网站以思维导图的形式展现规范的格式以及说明。

![OpenAPI Map](openapi-map.png)

### OpenAPI.Tools

现在我们知道了 OpenAPI 规范，接下来要做什么？当然是了解 OpenAPI 开放了哪些能力。

有一个叫 [OpenAPI.Tools](https://openapi.tools/) 的网站，分类整理并记录了社区围绕 OpenAPI 规范开发的流行工具。

![Tool Types](openapi-tool-types.png)

可以看到列表中有很多分类，在我们日常开发中，最经常使用的有三类：

#### 文档编辑器

文档编辑器方便我们用来编写符合 OpenAPI 规范的文档，有助于提高编写文档的效率，就像 [VS Code](https://code.visualstudio.com/) 能够方便我们编写代码一样。

文档编辑器有两种：[文本编辑器](https://openapi.tools/#text-editors) 以及 [图形编辑器](https://openapi.tools/#gui-editors)。

文本编辑器推荐使用在线的 [Swagger Editor](https://editor.swagger.io/)，能够实现格式校验和实时预览 Swagger 交互式 API 文档功能，效果如下图所示：

![Swagger Editor](swagger-editor.png)

如果你习惯使用 `VS Code`，也有[相应插件](https://marketplace.visualstudio.com/items?itemName=42Crunch.vscode-openapi)可供使用。

图形编辑器的好处是能够以可视化的形式编辑内容，不了解 OpenAPI 规范语法也能编辑。可以根据自己喜好来进行选择，如 [Stoplight Studio](https://stoplight.io/studio)、[APIGit](https://apigit.com/zh-CN) 等。

#### Mock 服务器

当我们使用 OpenAPI 规范来进行接口开发时，往往采用文档先行的策略，也就是前后端在开发代码前，先定义好接口文档，再进行代码的编写。此时前端如果想测试接口可用性，而后端代码还没有编写完成，[Mock 服务器](https://openapi.tools/#mock)就派上用场了。Mock 服务器能够根据所提供的 OpenAPI 接口文档，自动生成一个模拟的 Web Server。使用 Mock 服务器能够轻松模拟真实的后端接口，方便前端同学进行接口调试。

上面提到的 [APIGit](https://apigit.com/zh-CN/why-apigit/mock-server) 也同时具备此功能。

#### 代码生成器

还有一种很实用的工具是代码生成器，代码生成器有两种类型：一种是从代码/注释生成 OpenAPI 文档，另一种是从 OpenAPI 文档生成代码。

这类工具同样非常多，且更为实用。比如我们有一份写好了的 Go Web Server 代码，想要自动生成一份 OpenAPI 文档，就可以使用 [go-swagger](https://github.com/go-swagger/go-swagger) 这个工具来生成一份 `openapi.yaml` 文档。

而如果我们有一份 `openapi.yaml` 文档，就可以利用 `go-swagger` 生成一份 Go SDK 代码，甚至它还能根据这份 OpenAPI 文档生成 Go Web Server 的框架代码，我们只需要在对应的接口里面实现具体的业务逻辑即可。

不仅 Go 语言有这样的工具，像 [Swagger Codegen](https://swagger.io/tools/swagger-codegen/) 和 [OpenAPI Generator](https://github.com/openapitools/openapi-generator) 这类工具更是支持几乎所有主流编程语言。

代码生成器是开发者应该着重关注的工具，使用这些工具可以减少大量手动且重复的工作，你可以[在此](https://openapi.tools/#sdk)看下有没有感兴趣的项目供你使用。

### Swagger

#### 什么是 Swagger

Swagger 是一套围绕 OpenAPI 规范所构建的开源工具集，提供了强大和易于使用的工具来充分利用 OpenAPI 规范，Swagger 工具集由最初的 Swagger 规范背后的团队所开发。

Swagger 工具集提供了 API 设计、开发、测试、治理和监控等能力，其中最主要的工具包含如下三个：

- [Swagger Codegen](https://swagger.io/tools/swagger-codegen/)：根据 OpenAPI 规范定义生成服务器存根和客户端 SDK。

- [Swagger Editor](https://swagger.io/tools/swagger-editor/)：基于浏览器的在线 OpenAPI 规范编辑器。

- [Swagger UI](https://swagger.io/tools/swagger-ui/)：以 UI 界面的方式可视化展示 OpenAPI 规范定义，并且能够在浏览器中进行交互。

当然 Swagger 也有为企业用户提供的收费版本工具，如 [SwaggerHub Enterprise](https://swagger.io/tools/swaggerhub/enterprise/)，感兴趣的同学可以自行了解。

#### Swagger 和 OpenAPI 的关系

讲到了 Swagger，就不得不提及 Swagger 和 OpenAPI 的联系与区别，因为这二者经常在一起出现。

前文也说过 OpenAPI 规范是基于 Swagger 规范演变而来的，但其实二者并不相等。

在 OpenAPI 尚未出现之前，Swagger 代表了 Swagger 规范以及一系列围绕 Swagger 规范的开源工具集。Swagger 规范最后一个版本是 [2.0](https://swagger.io/specification/v2/)，之后就捐赠给了 OAI 并被重新命名为 OpenAPI 规范，所以 OpenAPI 规范第一个版本是 2.0，也就是 Swagger 规范 2.0，而由 OAI 这个组织发布的第一个 OpenAPI 规范正式版本是 [3.0.0](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.0.md)。

现在，Swagger 规范已被 OpenAPI 规范完全接管并取代。OpenAPI 代表了 OpenAPI 规范以及一系列生态，而 Swagger 则是这个生态中的一部分，是 Swagger 团队围绕 OpenAPI 规范所开发的一系列工具集。

Swagger 是 OpenAPI 生态中非常重要的组成部分，因为它给出了一整套方案，且非常流行。

Swagger 和 OpenAPI 二者 LOGO 对比如下：

![Swagger-OpenAPI-LOGO](swagger-openapi.png)

希望你下次再见到这两个 LOGO 时能清晰分辨出二者，而不被混淆。

### 以 Go 语言为例讲解 OpenAPI 在实际开发中的应用

前文介绍了编写 OpenAPI 文档的两种编辑器：文本编辑器以及图形编辑器。在日常开发中，后端可以先使用这类编辑器如 Swagger Editor 编写出 OpenAPI 文档，然后将这份文档交给前端，前端拿到 OpenAPI 文档后将其导入到 Swagger Editor，就可以在线阅读接口文档并与之进行交互，之后前后端就可以并行开发了。

这样的工作流看起来似乎没什么问题，不过编写 OpenAPI 文档毕竟是个苦力活，不仅有大量的重复工作，还要求开发者熟练掌握 OpenAPI 规范语法。这对于“爱偷懒”的开发者显然是无法接受的，就像段子里说的，程序员最讨厌两件事：1. 写文档，2. 别人不写文档。而这个问题的解法，当然就是前文提到的代码生成器。

#### 使用 Swag 生成 Swagger 文档

在 Go 语言生态里，目前有两个比较流行的开源工具可以生成 Swagger 文档，分别是 [go-swagger](https://github.com/go-swagger/go-swagger) 和 [swag](https://github.com/swaggo/swag)。它们都能根据代码中的注释生成 Swagger 文档，go-swagger 作为一款 OpenAPI.Tools 推荐的工具，其功能比 swag 更加强大且 Github Star 数量也更高。

不过本文将选择 swag 来进行介绍，一是因为 swag 比较轻量，更适合微服务开发；二是如果使用 swag，那么注释代码会离接口代码更近，升级时方便维护。如果你有更高级的需求，如根据 Swagger 文档生成客户端 SDK，服务端存根等，则推荐使用 go-swagger。

> 注意：在这里我一直提到的都是生成 Swagger 文档，而没有说是 OpenAPI 文档。因为无论是 swag 还是功能更强大的 go-swagger，它们目前都仅支持生成 OpenAPI 2.0 文档，并不支持生成 OpenAPI 3.0+ 文档，而 OpenAPI 2.0 版本我们更习惯称其为 Swagger 文档。

##### 安装 Swag

使用 `go install` 命令即可安装。

```bash
$ go install github.com/swaggo/swag/cmd/swag@latest # 安装
$ swag --version # 查看版本
swag version v1.8.10
```

##### Swag 命令行工具

swag 非常简洁，仅提供了两个主要命令 `init` 和 `fmt`：

```bash
$ swag -h # 查看帮助
NAME:
   swag - Automatically generate RESTful API documentation with Swagger 2.0 for Go.

USAGE:
   swag [global options] command [command options] [arguments...]

VERSION:
   v1.8.10

COMMANDS:
   init, i  Create docs.go
   fmt, f   format swag comments
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h     show help (default: false)
   --version, -v  print the version (default: false)
```

- 在包含 `main.go` 文件（默认情况下）的项目根目录运行 `swag init` 命令，将会解析 swag 注释并生成 `docs/` 目录以及 `/docs/docs.go`、`docs/swagger.json`、`docs/swagger.yaml` 三个文件。

```bash
$ swag init -h # 查看 init 子命令使用方法
NAME:
   swag init - Create docs.go

USAGE:
   swag init [command options] [arguments...]

OPTIONS:
   --quiet, -q                            不在控制台输出日志 (default: false)
   --generalInfo value, -g value          API 通用信息所在的 Go 源文件路径，如果是相对路径则基于 API 解析目录 (default: "main.go")
   --dir value, -d value                  API 解析目录，多个目录可用逗号分隔 (default: "./")
   --exclude value                        解析扫描时排除的目录，多个目录可用逗号分隔
   --propertyStrategy value, -p value     结构体字段命名规则，三种：snake_case，camelCase，PascalCase (default: "camelCase")
   --output value, -o value               所有生成文件的输出目录（swagger.json, swagger.yaml and docs.go）(default:"./docs")
   --outputTypes value, --ot value        生成文件的输出类型（docs.go, swagger.json, swagger.yaml）三种：go,json,yaml (default: "go,json,yaml")
   --parseDependency, --pd                解析依赖目录中的 Go 文件 (default: false)
   --markdownFiles value, --md value      指定 API 的描述信息所使用的 Markdown 文件所在的目录，默认禁用
   --parseInternal                        解析 internal 包中的 Go 文件 (default: false)
   --generatedTime                        输出时间戳到输出文件 `docs.go` 顶部 (default: false)
   --parseDepth value                     依赖项解析深度 (default: 100)
   --requiredByDefault                    默认情况下，为所有字段设置 `required` 验证 (default: false)
   --instanceName value                   设置文档实例名 (default: "swagger")
   --parseGoList                          通过 'go list' 解析依赖关系 (default: true)
   --tags value, -t value                 逗号分隔的标签列表，用于过滤指定标签生成 API 文档。特殊情况下，如果标签前缀是 '!' 字符，那么带有该标记的 API 将被排除
   --help, -h                             显示帮助信息 (default: false)
```

> 注意：以上 `swag init` 命令可选参数介绍略有删减，只列出了常用选项，更完整的文档请参考[官方仓库](https://github.com/swaggo/swag/blob/master/README.md#swag-cli)。

- `swag fmt` 命令可以格式化 swag 注释。

```bash
$ swag fmt -h # 查看 fmt 子命令使用方法
NAME:
   swag fmt - format swag comments

USAGE:
   swag fmt [command options] [arguments...]

OPTIONS:
   --dir value, -d value          API 解析目录，多个目录可用逗号分隔 (default: "./")
   --exclude value                解析扫描时排除的目录，多个目录可用逗号分隔
   --generalInfo value, -g value  API 通用信息所在的 Go 源文件路径，如果是相对路径则基于 API 解析目录 (default: "main.go")
   --help, -h                     显示帮助信息 (default: false)
```

##### 在 Gin 中使用 Swag

在 [gin](https://github.com/gin-gonic/gin) 框架能够很方便的使用 swag，步骤如下：

1. 准备项目目录结构如下：

```bash
.
├── go.mod
├── go.sum
└── main.go
```

2. 初始化项目

```bash
$ go mod init gin-swag
```

3. 编写 `main.go` 代码如下：

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// @title           Swagger Example API
// @version         1.0

// @schemes         http
// @host            localhost:8080
// @BasePath        /api/v1

// @tag.name        example
// @tag.description 示例接口

// Helloworld godoc
//
// @Summary     该操作的简短摘要
// @Description 操作行为的详细说明
// @Tags        example
// @Accept      json
// @Produce     json
// @Success     200 {string} string "Hello World!"
// @Router      /example/helloworld [get]
func Helloworld(g *gin.Context) {
	g.JSON(http.StatusOK, "Hello World!")
}

func main() {
	r := gin.Default()
	v1 := r.Group("/api/v1")
	{
		eg := v1.Group("/example")
		{
			eg.GET("/helloworld", Helloworld)
		}
	}

	if err := r.Run(":8080"); err != nil {
		panic(err)
	}
}
```

代码中的注释部分即为 swag 的注释语法，稍后通过这些注释生成 Swagger 文档。

其中通用 API 信息部分注释含义如下：

| 注释 | 说明 |
| :---- | :---- |
| @title | 必填，应用程序的名称。|
| @version | 必填，提供应用程序 API 的版本。|
| @schemes | 用空格分隔的请求传输协议。|
| @host | 运行 API 的主机（主机名或 IP 地址）。|
| @BasePath | 运行 API 的基本路径。|
| @tag.name | 标签的名称。|
| @tag.description | 标签的描述。|

还有一部分注释代表了 API 操作，其含义如下：

| 注释 | 说明 |
| :---- | :---- |
| @Summary | 该操作的简短摘要。|
| @Description | 操作行为的详细说明。|
| @Tags | 该 API 操作的标签列表，多个标签以逗号分隔。|
| @Accept | API 可以接收的参数 MIME 类型列表。|
| @Produce | API 可以生成的参数 MIME 类型列表。|
| @Success | 成功响应。|
| @Router | 路由路径定义。|

以上这些注释最终都会对应到 OpenAPI 2.0 规范的某个字段上。更多说明请参考[官方文档](https://github.com/swaggo/swag#declarative-comments-format)，并且官方也提供了[中文文档](https://github.com/swaggo/swag/blob/master/README_zh-CN.md#%E5%A3%B0%E6%98%8E%E5%BC%8F%E6%B3%A8%E9%87%8A%E6%A0%BC%E5%BC%8F)。

4. 使用 swag 根据注释生成 Swagger 文档，在项目根目录下（`.`）执行 `swag init`，将得到新的目录结构：

```bash
.
├── docs
│   ├── docs.go
│   ├── swagger.json
│   └── swagger.yaml
├── go.mod
├── go.sum
└── main.go
```

可以发现 `swag init` 生成的三个文件 `docs.go`、`swagger.json`、`swagger.yaml` 默认都在 `docs/` 目录下。

其中 `swagger.json`、`swagger.yaml` 正是符合 OpenAPI 2.0 规范的 JSON 和 YAML 接口文档，例如 `swagger.yaml` 内容如下：

```yaml
basePath: /api/v1
host: localhost:8080
info:
  contact: {}
  title: Swagger Example API
  version: "1.0"
paths:
  /example/helloworld:
    get:
      consumes:
      - application/json
      description: 操作行为的详细说明
      produces:
      - application/json
      responses:
        "200":
          description: Hello World!
          schema:
            type: string
      summary: 该操作的简短摘要
      tags:
      - example
schemes:
- http
swagger: "2.0"
tags:
- description: 示例接口
  name: example
```

对比上面代码中的注释，很容易将其对应起来，相比于直接编写 YAML 格式文档，显然在代码中编写注释更为简单。

将其复制到 Swagger Editor 编辑器中即可查看 Swagger UI 预览。

![Swagger UI](gin-swag-example-swagger-editor.png)

##### 将 Gin 作为 Swagger UI 服务器

上面我们通过 swag 生成了 Swagger 文档，并手动将生成的 `swagger.yaml` 复制到 Swagger Editor 编辑器进行 Swagger UI 预览。不过这么做显然有点麻烦，好在 swag 作者也考虑到了这一点，所以他又提供了另外两个项目 [gin-swagger](https://github.com/swaggo/gin-swagger) 和 [files](https://github.com/swaggo/files)，能够直接将 gin 作为 Swagger UI 服务器，这样就不用每次都将 `swagger.yaml` 手动复制到 Swagger Editor 编辑器才能实现 Swagger UI 预览。

使用步骤如下：

1. 下载 gin-swagger、files

```bash
$ go get -u github.com/swaggo/gin-swagger
$ go get -u github.com/swaggo/files
```

2. 在代码中导入 gin-swagger、files

```go
import "github.com/swaggo/gin-swagger" // gin-swagger middleware
import "github.com/swaggo/files" // swagger embed files
```

3. 注册 Swagger 文档路由地址

```go
r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
```

完整代码如下：

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
	swaggerFiles "github.com/swaggo/files"
	ginSwagger "github.com/swaggo/gin-swagger"

	"gin-swag/docs" // 当前包名为 gin-swag
)

// @title           Swagger Example API
// @version         1.0

// @schemes         http
// @host            localhost:8080
// @BasePath        /api/v1

// @tag.name        example
// @tag.description 示例接口

// Helloworld godoc
//
// @Summary     该操作的简短摘要
// @Description 操作行为的详细说明
// @Tags        example
// @Accept      json
// @Produce     json
// @Success     200 {string} string "Hello World!"
// @Router      /example/helloworld [get]
func Helloworld(g *gin.Context) {
	g.JSON(http.StatusOK, "Hello World!")
}

func main() {
	// 会覆盖上面注释部分 title 属性的设置
	docs.SwaggerInfo.Title = "Swag Example API"

	r := gin.Default()
	v1 := r.Group("/api/v1")
	{
		eg := v1.Group("/example")
		{
			eg.GET("/helloworld", Helloworld)
		}
	}

	// Swagger 文档接口地址
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	if err := r.Run(":8080"); err != nil {
		panic(err)
	}
}
```

4. 执行 `go run main.go` 启动服务，访问 `http://localhost:8080/swagger/index.html` 即可查看 Swagger UI 交互式文档界面。

![Swagger UI](gin-swagger.png)

这个本地的 Swagger UI 服务器同样支持交互式操作。

展开 `/example/helloworld` 这个接口，点击 `Try it out`。

![Try it out](gin-swagger-try-it-out.png)

接着，点击 `Execute`。

![Execute](gin-swagger-execute.png)

Swagger UI 将会根据文档的 `Base URL` 去请求真正的接口（同时还会给出 cURL 发送请求的命令，方便复制使用），并将响应结果展示出来。

![Swagger UI](gin-swagger-action.jpg)

同时后端服务器能够打印出请求记录：

![Server](gin-swagger-helloworld.png)

与前端对接时，我们只需要将接口文档地址给到前端，前端就可以根据这个 Swagger UI 界面进行接口查阅和调试了，非常方便。

##### 让 Swag 支持多版本 API 文档

实际工作中，我们的项目会比这个只有一个接口的 demo 复杂得多，同时 API 也可能会支持多版本，比如 `/api/v1`、`/api/v2`，针对这些场景，我实现了一个使用示例 [swag-example](https://github.com/jianghushinian/swag-example)，供你参考。

为了更加方便的使用 swag 命令，在 swag-example 项目中，我把 `swag fmt` 和 `swag init` 命令都写在 `Makefile` 文件中，最重要的就是以下两条命令：

```bash
swag init -g internal/api/controller/v1/docs.go --exclude internal/api/controller/v2 --instanceName v1
swag init -g internal/api/controller/v2/docs.go --exclude internal/api/controller/v1 --instanceName v2
```

这两条命令分别生成 `v1`、`v2` 两个版本的 API 文档，这样可以将不同版本的接口分开展示，更加清晰。

其中 `-g` 参数指明 API 通用注释信息所在的 Go 源文件路径，大型项目中为了保持代码架构整洁，这些注释应该独立于一个文件，而不是直接写在 `main.go` 中。

`--exclude` 参数指明生成 Swagger 文档时，需要排除的目录。可以发现，在生成 `v1` 版本接口文档时，我排除了 `v2` 接口目录，在生成 `v2` 版本接口文档时，排除了 `v1` 接口目录，这样就实现了多版本接口分离。

关于这个项目的更多细节就留给你自己去探索了，相信你阅读完代码后会有所收获。

##### Swag 使用建议

在前文介绍的 swag 使用流程中，不知道你有没有注意到，我们是先编写的代码，然后再生成的 Swagger 文档，最后将这份文档交给前端使用。

这显然违背了「文档先行」的思想，实际工作中，我们更多的时候是先跟前端约定好接口，然后后端提供 Swagger 文档供前端使用，最后才是前后端编码阶段。

要想解决这个问题，最直接的解决方案是不使用 swag 工具，而是直接使用 Swagger Editor 这种编辑器手写 Swagger 文档，这样就能实现文档先行了。

但这又违背了 OpenAPI 给出的「最佳实践」，推荐自动生成 Swagger 文档，而非手动编写。

我自己的解决方案是，依旧选择使用 swag 工具，不过在编写代码时，先写接口的框架代码，而不写具体的业务逻辑，这样就能够先通过接口注释生成 Swagger 文档，供前端使用，然后再编写业务代码。

另外，较为遗憾的是，目前 swag 生成的文档是 OpenAPI 2.0 版本，并不能直接生成 OpenAPI 3.0 版本，如果你想使用 OpenAPI 3.0 版本的文档，一个变通的方法是使用工具将 OpenAPI 2.0 文档转换成 OpenAPI 3.0，如前文提到的 Swagger Editor 就支持此操作。

#### 使用 ReDoc 风格的 API 文档

也许相较于 Swagger UI 多年不变的界面风格，你更喜欢 ReDoc 风格的 UI，那么 [go-redoc](https://github.com/mvrilo/go-redoc) 是一个比较不错的选择。

在 gin 中使用 go-redoc 非常简单，只需要将如下套路代码加入到我们的 `main.go` 文件中即可。

```go
import (
	"github.com/gin-gonic/gin"
	"github.com/mvrilo/go-redoc"
	ginRedoc "github.com/mvrilo/go-redoc/gin"
)

...

doc := redoc.Redoc{
    Title:       "Example API",
    Description: "Example API Description",
    SpecFile:    "./openapi.json", // "./openapi.yaml"
    SpecPath:    "/openapi.json",  // "/openapi.yaml"
    DocsPath:    "/docs",
}

r := gin.New()
r.Use(ginRedoc.New(doc))
```

现在完整代码如下：

```go
package main

import (
	"net/http"
	"path"
	"runtime"

	"github.com/gin-gonic/gin"
	"github.com/mvrilo/go-redoc"
	ginRedoc "github.com/mvrilo/go-redoc/gin"
	swaggerFiles "github.com/swaggo/files"
	ginSwagger "github.com/swaggo/gin-swagger"

	"gin-swag/docs" // 当前包名为 gin-swag
)

// @title           Swagger Example API
// @version         1.0

// @schemes         http
// @host            localhost:8080
// @BasePath        /api/v1

// @tag.name        example
// @tag.description 示例接口

// Helloworld godoc
//
// @Summary     该操作的简短摘要
// @Description 操作行为的详细说明
// @Tags        example
// @Accept      json
// @Produce     json
// @Success     200 {string} string "Hello World!"
// @Router      /example/helloworld [get]
func Helloworld(g *gin.Context) {
	g.JSON(http.StatusOK, "Hello World!")
}

func main() {
	// 会覆盖上面注释部分 title 属性的设置
	docs.SwaggerInfo.Title = "Swag Example API"

	_, filename, _, _ := runtime.Caller(0)
	doc := redoc.Redoc{
		Title:       "ReDoc Example API",
		Description: "ReDoc Example API Description",
		SpecFile:    path.Join(path.Dir(filename), "docs/swagger.json"),
		SpecPath:    "/swagger.json",
		DocsPath:    "/redoc",
	}

	r := gin.Default()
	v1 := r.Group("/api/v1")
	{
		eg := v1.Group("/example")
		{
			eg.GET("/helloworld", Helloworld)
		}
	}

	// Swagger 文档接口地址
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	// 注册 Redoc 文档接口地址
	r.Use(ginRedoc.New(doc))

	if err := r.Run(":8080"); err != nil {
		panic(err)
	}
}
```

执行 `go run main.go` 启动服务，访问 http://localhost:8080/redoc 即可查看 Redoc UI。

![ReDoc](redoc.png)

不过，相较于 Swagger UI，Redoc UI 有个弊端是不能实现交互式操作，如果仅以此作为文档查阅工具，没有交互式操作的需求，那么还是比较推荐使用的。

### 更先进的 API 工具

除了 OpenAPI.Tools 推荐的开源工具，社区中其实还有很多其他优秀工具值得尝试使用，比如我这里要推荐的一款国产工具 [Apifox](https://www.apifox.com/)，官方将其定义为 Apifox = Postman + Swagger + Mock + JMeter，集 API 设计/开发/测试 于一身。

Apifox 可谓一站式图形化工具，其功能非常强大，就像前文提到的 APIGit 同时具备了编辑器和 Mock 服务器的功能，Apifox 有过之而无不及。

![Apifox](apifox.png)

图形化工具上手难度不大，加上 Apifox 本身由国内开发，非常容易上手，所以本文也就不深入介绍了，你可以观看官方教程 [21 分钟学会 Apifox](https://www.apifox.cn/help/#_21-%E5%88%86%E9%92%9F%E5%AD%A6%E4%BC%9A-apifox-%F0%9F%91%8D) 来学习使用。

**参考**

> OpenAPI 官网：https://www.openapis.org/
> OpenAPI 入门：https://oai.github.io/Documentation/
> OpenAPI 规范：https://spec.openapis.org/oas/latest.html
> OpenAPI 规范中文版：https://openapi.apifox.cn/
> OpenAPI 规范思维导图版：https://openapi-map.apihandyman.io/
> OpenAPI.Tools：https://openapi.tools/
> Swagger 官网：https://swagger.io/
> swag：https://github.com/swaggo/swag
> swag-example：https://github.com/jianghushinian/swag-example
> go-redoc：https://github.com/mvrilo/go-redoc
> Apifox 官网：https://www.apifox.com/

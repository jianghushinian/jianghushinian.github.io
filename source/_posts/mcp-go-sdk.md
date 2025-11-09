---
title: 'MCP 官方 Go SDK v1.0.0 正式发布：Go 生态的模型上下文协议步入稳定时代'
date: 2025-10-08 23:24:58
tags:
- Go
- MCP
categories:
- Go
---

在人工智能快速发展的今天，大型语言模型（LLM）需要更丰富、更动态的上下文信息来完成任务。Model Context Protocol (MCP) 应运而生，它定义了一套标准协议，允许模型与外部工具、数据源和服务进行安全、高效的交互，极大地扩展了大模型的能力边界。

<!-- more -->

但是，MCP 官方 Go SDK 却姗姗来迟，在 Python SDK 已经满世界运行时，Go SDK 项目才刚刚启动，这也导致官方 Go SDK 稍显落后。不过，经过数月的社区测试、迭代和打磨，`modelcontextprotocol/go-sdk` 项目终于迎来了具有里程碑意义的 [**v1.0.0** 正式版本](https://github.com/modelcontextprotocol/go-sdk/releases/tag/v1.0.0)。这标志着 Go 语言版本的 MCP 官方开发套件结束了“不稳定”的预览阶段，进入了生产可用的稳定时代。

### v1.0.0 意味着什么？—— 承诺与信任

版本号从 `v0.x.x` 跃升至 `v1.0.0`，绝不仅仅是数字的变化，它背后代表着项目维护者向所有开发者作出的一项庄严承诺：**API 稳定性**。

根据官方发布说明，核心团队明确表示：“Going forward we won’t make breaking API changes.”（未来我们将不会进行破坏性的 API 更改）。这意味着：

1. **可被信赖的依赖**：现在可以放心大胆地将此 SDK 用于真实的、生产级的项目中，无需担心未来因 SDK 本身的不兼容升级而带来的额外成本和风险。
2. **向后兼容性**：任何未来的功能增强和优化都将以向后兼容的方式进行。如果需要对现有 API 进行改进，将会通过“弃用（deprecating）”并添加新方式来实现，给予开发者充足的迁移时间。
3. **生态发展的基石**：稳定的 API 是构建强大生态的基础。库作者、工具开发者可以基于此稳定版本构建更上层的应用和集成，促进整个 Go MCP 生态的繁荣。

### 快速上手

`modelcontextprotocol/go-sdk` 提供了构建 MCP 客户端（Client）和服务器（Server）所需的所有核心功能，设计清晰直观，非常易于使用。

#### 安装

```bash
$ go get github.com/modelcontextprotocol/go-sdk/mcp@v1.0.0
```

#### 创建一个 MCP 服务器

下面的代码示例展示如何创建一个简单的“问候”服务器，它提供了一个名为 `greet` 的工具。

```go
package main

import (
	"context"
	"log"

	"github.com/modelcontextprotocol/go-sdk/mcp" // 导入 MCP Go SDK
)

// Input 结构体定义了工具调用时的输入参数格式
type Input struct {
	// json 标签用于序列化/反序列化，jsonschema 标签提供元数据描述（用于文档或验证）
	Name string `json:"name" jsonschema:"the name of the person to greet"`
}

// Output 结构体定义了工具调用后的输出结果格式
type Output struct {
	Greeting string `json:"greeting" jsonschema:"the greeting to tell to the user"`
}

// SayHi 是工具的实际处理函数
// 它接收 context、MCP 工具调用请求对象和解析后的 Input 参数。
// 返回三个值：*mcp.CallToolResult（可选的额外结果元数据）、Output（业务输出）和 error（错误）。
func SayHi(ctx context.Context, req *mcp.CallToolRequest, input Input) (
	*mcp.CallToolResult,
	Output,
	error,
) {
	return nil, Output{Greeting: "Hi " + input.Name}, nil
}

func main() {
	// 创建一个新的 MCP 服务器实例
	server := mcp.NewServer(&mcp.Implementation{Name: "greeter", Version: "v1.0.0"}, nil)

	// 向服务器注册一个工具
	mcp.AddTool(server, &mcp.Tool{Name: "greet", Description: "say hi"}, SayHi)

	// 启动服务器，使用标准输入输出（stdio）作为传输层
	// Run 方法会阻塞，直到客户端断开连接或发生错误
	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatal(err)
	}
}
```

#### 创建一个 MCP 客户端

客户端可以连接到上述服务器并调用其工具。

```go
package main

import (
	"context"
	"log"
	"os/exec" // 用于执行外部命令（这里用于启动服务器进程）

	"github.com/modelcontextprotocol/go-sdk/mcp" // 导入 MCP Go SDK
)

func main() {
	ctx := context.Background()

	// 创建一个新的 MCP 客户端实例
	client := mcp.NewClient(&mcp.Implementation{Name: "mcp-client", Version: "v1.0.0"}, nil)

	// 设置传输层：通过执行外部命令 "myserver" 来启动服务器进程，并使用其 stdin/stdout 进行通信
	transport := &mcp.CommandTransport{Command: exec.Command("./greeting")}

	// 连接到服务器，建立会话（session）
	session, err := client.Connect(ctx, transport, nil)
	if err != nil {
		log.Fatal(err)
	}
	defer session.Close() // 确保在函数退出前关闭会话，释放资源

	// 准备工具调用参数
	params := &mcp.CallToolParams{
		Name:      "greet",                        // 工具名称
		Arguments: map[string]any{"name": "江湖十年"}, // 工具参数
	}

	// 调用服务器的工具
	res, err := session.CallTool(ctx, params)
	if err != nil {
		log.Fatalf("CallTool failed: %v", err)
	}
	if res.IsError {
		log.Fatal("tool failed")
	}

	// 处理响应内容
	for _, c := range res.Content {
		log.Print(c.(*mcp.TextContent).Text) // 类型断言为 TextContent 并获取文本
	}
}
```

#### 执行示例

根据以下命令执行 MCP 程序。

```bash
# 编译 MCP 服务器程序
$ go build -o client/greeting server/greeting.go
# 当前目录结构
$ tree
.
├── client
│   ├── client.go
│   └── greeting
└── server
    └── greeting.go

3 directories, 3 files
# 进入到 MCP 客户端目录
$ cd client
# 执行客户端程序
$ go run client.go
2025/10/08 22:59:01 {"greeting":"Hi 江湖十年"}
```

至此，一个基于 stdio 的 MCP 使用示例就完成了。

你可以在 [https://github.com/modelcontextprotocol/go-sdk/tree/main/examples](https://github.com/modelcontextprotocol/go-sdk/tree/main/examples) 探索更多使用示例。

### MCP Go SDK 设计

MCP Go SDK 的设计遵循了 Model Context Protocol (MCP) 规范，旨在提供一种标准化、类型安全的方式，让客户端（如 AI Agent）与服务器（提供工具、数据源或服务）进行通信。以下是其核心设计理念：

1. 模块化与分层设计：
    - 传输层抽象：SDK 定义了 `Transport` 接口（如 `StdioTransport`、`CommandTransport`、`SSEServerTransport`），允许通信通过不同渠道（stdio、HTTP）进行。这使得 SDK 能适应各种部署场景（本地、远程、云端）。
    - 会话管理：客户端和服务器通过 `Session` 对象管理连接状态，支持多次交互。会话生命周期包括连接、工具调用、资源访问和断开。
2. 类型安全与代码生成友好：
    - 工具输入和输出使用 Go 结构体定义，并借助 `json` 和 `jsonschema` 标签进行序列化和验证。这确保了数据契约的明确性，减少了运行时错误。
3. 客户端-服务器模式：
    - 服务器：注册工具（`mcp.AddTool`）和资源，处理请求。服务器可以同时支持多个工具，每个工具独立实现。
    - 客户端：发起工具调用（`session.CallTool`）和资源请求。客户端可以连接到多个服务器，实现聚合功能。
4. 协议兼容性与扩展性：
    - SDK 完整实现了 MCP 规范（如工具调用、资源读取、提示管理），并支持未来扩展。
5. 错误处理：
    - 错误通过 Go 的 `error` 机制返回，调用方需显式处理。工具响应中的 `IsError`字段表示业务逻辑错误。

### 总结

Go 语言的高效与 MCP 协议的强大相结合，必将为 AI 应用开发开启新的可能。

MCP Go SDK v1.0.0 的发布是 Go 开发者与 AI 应用交叉领域的一个重大事件。它提供了一个：

+ **稳定的** API 基础
+ **功能完整的** MCP 实现（支持工具、资源、提示等）
+ **经过实践检验的** 生产就绪组件

对于开发者而言，现在是开始使用 Go 构建 MCP 服务器（如连接数据库、内部 API、专属工具链）和客户端的绝佳时机。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/mcp/sdk) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ MCP Go SDK GitHub 源码：[https://github.com/modelcontextprotocol/go-sdk](https://github.com/modelcontextprotocol/go-sdk)
+ MCP Go SDK v1.0.0 Release：[https://github.com/modelcontextprotocol/go-sdk/releases/tag/v1.0.0](https://github.com/modelcontextprotocol/go-sdk/releases/tag/v1.0.0)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/mcp/sdk](https://github.com/jianghushinian/blog-go-example/tree/main/mcp/sdk)
+ 本文永久地址：[https://jianghushinian.cn/2025/10/08/mcp-go-sdk/](https://jianghushinian.cn/2025/10/08/mcp-go-sdk/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

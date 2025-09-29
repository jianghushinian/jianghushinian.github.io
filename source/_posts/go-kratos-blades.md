---
title: 'AI Agent 生态再添一员，Kratos 带着他的武器 Blades 走来了！'
date: 2025-09-29 10:12:07
tags:
- Go
- Agent
categories:
- Go
---

想必广大 Gopher 对 b 站开源的 Go 微服务框架 Kratos 并不陌生，如今 Kratos 生态中又新增了一款开源多模态 AI Agent 框架 Blades，它支持自定义模型、工具、记忆体、中间件等，适用于多轮对话、链式推理和结构化输出等。

<!-- more -->

### 快速开始

与大模型进行聊天（Chat）：

```go
package main

import (
	"context"
	"log"

	"github.com/go-kratos/blades"
	"github.com/go-kratos/blades/contrib/openai"
	"github.com/openai/openai-go/v2/option"
)

func main() {
	agent := blades.NewAgent(
		"Template Agent",
		blades.WithModel("deepseek-chat"),
		blades.WithProvider(openai.NewChatProvider(
			option.WithBaseURL("https://api.deepseek.com"),
			option.WithAPIKey("sk-xxx"),
		)),
	)

	// Define templates and params
	params := map[string]any{
		"topic":    "人工智能的未来",
		"audience": "普通读者",
	}

	// Build prompt using the template builder
	// Note: Use exported methods when calling from another package.
	prompt, err := blades.NewPromptTemplate().
		System("请将 {{.topic}} 总结为三个关键点。", params).
		User("对 {{.audience}} 受众做出简洁准确的回复。", params).
		Build()
	if err != nil {
		log.Fatal(err)
	}

	log.Println("Generated Prompt:", prompt.String())

	// Run the agent with the templated prompt
	result, err := agent.Run(context.Background(), prompt)
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("Generated result:\n%s", result.Text())
}

```

`blades.NewAgent()` 用于构造一个 Agent 对象，这里使用 DeepSeek 模型作为演示。

`blades.NewPromptTemplate()` 用于构造 Prompt 模板，这里使用了 Builder 设计模式，你可以参考我的另一篇文章《[Builder 模式在 Go 语言中的应用](https://mp.weixin.qq.com/s/LRyb7tOTZmq33j9AwOd18g)》进行更深入的学习。

`agent.Run()` 方法执行 Agent，等待大模型输出结果并返回。

示例程序运行结果如下：

```bash
$ go run chat.go                           
2025/09/28 23:16:03 Generated Prompt: [Text: 请将 人工智能的未来 总结为三个关键点。)][Text: 对 普通读者 受众做出简洁准确的回复。)]
2025/09/28 23:16:03 Processing message: system [{请将 人工智能的未来 总结为三个关键点。}]
2025/09/28 23:16:03 Processing message: user [{对 普通读者 受众做出简洁准确的回复。}]
2025/09/28 23:16:07 Generated result:
1. **智能的扩展**：人工智能将更深入地融入日常生活，提升效率并解决复杂问题。  
2. **伦理与责任**：技术发展需关注数据隐私、算法公平性及社会影响。  
3. **人机协作**：未来重点在于人类与AI互补合作，而非取代。
```

第 1 行日志记录了生成的最终 Prompt，较为遗憾的是 Prompt 输出未能打印类型（role：system/user 等），不够清晰。

好在第 2、3 两行日志输出结果中打印了 Prompt 类型。但这是 Blades 框架内部输出的，我们暂时还无法控制。

最终输出大模型生成结果。

### 流式输出

作为一个 AI Agent 框架，Blades 必然支持流式输出：

```go
stream, err := agent.RunStream(context.Background(), prompt)
if err != nil {
    log.Fatal(err)
}
for stream.Next() {
    chunk, err := stream.Current()
    if err != nil {
        log.Fatalf("stream recv error: %v", err)
    }
    log.Print(chunk.Text())
}
```

只需要 `agent.Run()` 方法替换为 `agent.RunStream()` 即可获取大模型的流式输出内容。

使用 `for` 循环遍历流式输出结果，这是典型的 Go 迭代器用法。你可以参考我的另一篇文章《[万字长文：彻底掌握 Go 1.23 中的迭代器——使用篇](https://mp.weixin.qq.com/s/t47eJ9rYK2CZ-hIbjx7kSg)》和《[万字长文：彻底掌握 Go 1.23 中的迭代器——原理篇](https://mp.weixin.qq.com/s/Bgr4GSA2AivTMlnxd4UQiA)》进行更深入的学习。

示例程序运行结果如下：

```bash
$ go run streaming.go
2025/09/28 23:26:41 Generated Prompt: [Text: 请将 人工智能的未来 总结为三个关键点。)][Text: 对 普通读者 受众做出简洁准确的回复。)]
2025/09/28 23:26:41 Processing message: system [{请将 人工智能的未来 总结为三个关键点。}]
2025/09/28 23:26:41 Processing message: user [{对 普通读者 受众做出简洁准确的回复。}]
2025/09/28 23:26:42 
2025/09/28 23:26:42 1
2025/09/28 23:26:43 .
2025/09/28 23:26:43  
2025/09/28 23:26:43 人工智能
2025/09/28 23:26:43 将
2025/09/28 23:26:43 更
...
2025/09/28 23:26:45 。
2025/09/28 23:26:45 
2025/09/28 23:26:45 1. 人工智能将更普及，融入日常生活，提升效率和便利性。
2. 伦理与安全成为重点，确保技术发展符合人类价值观。
3. 人机协作增强，AI辅助人类解决复杂问题，而非取代人类。
```

> NOTE:
>
> 由于流式输出内容过多，日志中省略了大部分流式输出行。
>

### 多轮对话

与大模型进行多轮对话（Conversation）：

```go
package main

import (
	"context"
	"log"

	"github.com/go-kratos/blades"
	"github.com/go-kratos/blades/contrib/openai"
	"github.com/go-kratos/blades/memory"
	"github.com/openai/openai-go/v2/option"
)

func main() {
	agent := blades.NewAgent(
		"History Tutor",
			blades.WithModel("deepseek-chat"),
			blades.WithProvider(openai.NewChatProvider(
				option.WithBaseURL("https://api.deepseek.com"),
				option.WithAPIKey("sk-xxx"),
			)),
		blades.WithInstructions("你是一位知识渊博的历史导师。提供详细、准确的历史事件信息。"),
		blades.WithMemory(memory.NewInMemory(10)),
	)
	// Example conversation in memory
	prompt := blades.NewConversation(
		"conversation_123",
		blades.UserMessage("你能告诉我第二次世界大战的起因吗？"),
	)
	result, err := agent.Run(context.Background(), prompt)
	if err != nil {
		log.Fatal(err)
	}
	log.Println(result.Text())

	prompt = blades.NewConversation(
		"conversation_123",
		blades.UserMessage("我刚刚问你的问题是什么？"),
	)
	result, err = agent.Run(context.Background(), prompt)
	if err != nil {
		log.Fatal(err)
	}
	log.Println(result.Text())
}
```

多轮对话需要记录历史聊天上下文，所以在使用 `blades.NewAgent()` 构造 Agent 时，使用 `blades.WithMemory(memory.NewInMemory(10))` 来实现。

`blades.NewConversation()` 构造一个带有会话 ID 的 Prompt。多次聊天，Prompt 中携带同一个会话 ID，即可实现多轮对话功能。

### 总结

Blades 为我们使用 Go 语言开发 AI Agent 带来了新的选择，本文介绍了使用 Blades 实现 AI Agent 的 3 个小示例，更多使用示例可以在官方 GitHub 仓库 [examples](https://github.com/go-kratos/blades/tree/main/examples) 目录中查看。

Blades 目前处于刚起步阶段，代码量并不多，如果你感兴趣，可以继续阅读源码进行深度学习。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/go-kratos/blades) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Blades GitHub 源码：[https://github.com/go-kratos/blades](https://github.com/go-kratos/blades)
+ Builder 模式在 Go 语言中的应用：[https://mp.weixin.qq.com/s/LRyb7tOTZmq33j9AwOd18g](https://mp.weixin.qq.com/s/LRyb7tOTZmq33j9AwOd18g)
+ 万字长文：彻底掌握 Go 1.23 中的迭代器——使用篇：[https://mp.weixin.qq.com/s/t47eJ9rYK2CZ-hIbjx7kSg](https://mp.weixin.qq.com/s/t47eJ9rYK2CZ-hIbjx7kSg)
+ 万字长文：彻底掌握 Go 1.23 中的迭代器——原理篇：[https://mp.weixin.qq.com/s/Bgr4GSA2AivTMlnxd4UQiA](https://mp.weixin.qq.com/s/Bgr4GSA2AivTMlnxd4UQiA)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/go-kratos/blades](https://github.com/jianghushinian/blog-go-example/tree/main/go-kratos/blades)
+ 本文永久地址：[https://jianghushinian.cn/2025/09/29/go-kratos-blades/](https://jianghushinian.cn/2025/09/29/go-kratos-blades/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

---
title: '记一次在 K8s 环境中排查 Agent 选择多个 MCP 后无结果返回问题'
date: 2025-07-05 15:27:18
tags:
- Kubernetes
- Agent
- AI
- MCP
categories:
- Agent
---

本文完整的记录了我在遇到 Agent 选择多个 MCP 后进行聊天，无法获得返回结果的一次排查案例，以此来提醒自己不要再犯愚蠢的错误。

<!-- more -->

### 起因

首先简单介绍一下项目背景，我最近在负责一个 AI Agent 项目的开发，这个 Agent 在聊天的时候可以调用 MCP 工具。而 MCP 服务有多种来源，其中有一种是通过开源项目 Higress 一键将 HTTP 接口转换成的 MCP 服务（使用 OpenAPI 文档，将文档中所有 HTTP 接口全部转换成 MCP 工具）。Agent 在调用这种 MCP 工具时，会将 MCP 请求发送给 Higress 网关，Higress 网关收到 MCP 请求以后，就通过 HTTP 的方式再去请求 HTTP 接口，在拿到 HTTP 响应后，Higress 通过 MCP 协议将数据返给 MCP 客户端。

> NOTE：
>
> 如果你想了解 HTTP 接口如何转换成 MCP 服务，也可以看我写的另一篇文章《[使用 MCP Gateway 一键将你的 HTTP 接口转换成 MCP Server](https://mp.weixin.qq.com/s/JU10VIvZnCvca1kTkjZufQ)》。
>

基于这个项目背景，我还在工位上搬砖，负责 Agent 测试的同学找到我说 Agent/MCP 服务出问题了，Agent 选择一个 MCP 进行聊天的时候，可以正常调用工具并返回结果。但是，如果 Agent 选择了多个 MCP 进行聊天，Agent 就不会有任何结果返回（前提是多个 MCP 服务都是通过 Higress 一键转换完成的）。

于是，我赶紧放下手中的砖头开始着手解决这个问题。

### 现象

上面对这个问题的描述可能比较抽象，对于没接触过 MCP 的读者不太友好，我以 [Cherry Studio](https://www.cherry-ai.com/) 作为 “AI Agent” 为例对这个问题进一步说明。

如下是 Cherry Studio 客户端聊天窗口的截图：

![Cherry Studio](cherry-studio.png)

这里我为模型聊天会话配置了两个 MCP 工具，这样在聊天时，大模型就能使用外部工具来拓展自身能力。

在与大模型进行聊天时，如果单独勾选「高德地图」或者单独勾选「weather」MCP 服务，Agent 程序是可以正常聊天并调用 MCP 工具的。但是如果同时勾选了这两个 MCP，就无法正常聊天，Agent 无任何结果返回，直到超时断开 WebSocket 连接。

> NOTE:
>
> Agent 程序内部会实例化一个 MCP 客户端，当大模型需要调用 MCP 工具时，就会通过这个 MCP 客户端与 MCP 服务器进行通信。
>

这就是测试同学跟我反馈的问题。

### 想法

遇到问题，当然是要解决问题，以下是对问题的分析。

遇到程序问题，第一个想到的排查方法当然是看日志。不过，以我对 Agent 代码的熟悉程度，其实这个问题我心里是有预期的，大概率通过查日志是排查不出来什么问题的（因为 Agent 代码是基于 [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) 开发的，而 OpenAI Agents SDK 封装了与 MCP 交互的逻辑，所以日志并不能覆盖整个链路）。

所以，我的想法是，先去查看下 Agent 调用 MCP 服务相关的日志。如果 Agent 日志看不出什么问题，就要查看下 Higress 日志，当然，大概率也查不出什么问题。如果日志查不出问题，再思考其他解决办法，比如兜底的方案就是看 Agent 源码是如何调用 MCP 的，毕竟源码之下了无秘密。

### 实操

那么，接下来就开始动手排查问题了。

#### Agent 日志

不出所料，Agent 日志里并不能直观的看出问题所在。不过，还是有所收获的，就是日志中会疯狂打印类似如下输出：

```plain
INFO:   2025-07-01 10:25:22,628 - _client.py:1740 - HTTP Request: POST http://higress-gateway.higress-system.svc.cluster.local/mcp-server/mcp-server-54?sessionId=722661db-c271-4ce4-82bd-cf8c07e4e736 "HTTP/1.1 202 Accepted"
```

这就说明 Agent 在不停的重试，反复调用 MCP 服务。这里我记下了一个疑问，为什么 Agent 会重复调用 MCP？

但这也仅能表明程序存在问题，不能找到出问题的原因。

#### Higress 日志

所以，接下来就要继续排查下 Higress 的日志。很遗憾，这里什么线索也没找到。

至此，日志排查完了，但收获不大。

#### 验证 MCP 服务是否正常

那么这个时候，我打算手动调用一下（不通过 Agent 调用） MCP 工具，验证一下 MCP 服务是否正常。

手动调用 MCP 工具的方式有很多，我这里使用 MCP 官方提供的 MCP 客户端工具 [Inspector](https://modelcontextprotocol.io/docs/tools/inspector) 来验证。

Inspector 界面长这样：

![MCP Inspector](inspector.png)

经测试，MCP 服务全部正常。

但是，因为 Inspector 一次只能调用一个 MCP 工具，所以，也不能验证 Agent 一次调用多个 MCP 工具的场景。所以这个步骤对排查问题帮助不大。

#### MCP 协议工具调用流程

既然 MCP 服务都正常，那么就只能排查 Agent 的问题了。

在验证 MCP 服务之前，我再给你补充一点 MCP 协议的知识，以免有些读者不明白 MCP 工具调用流程。

以下是 MCP 官方提供的 [HTTP with SSE](https://modelcontextprotocol.io/specification/2024-11-05/basic/transports#http-with-sse) 模式下 MCP 客户端和服务端交互流程：

![MCP HTTP with SSE](http-with-sse.png)

步骤如下：

+ MCP 客户端首先向服务端 SSE endpoint（比如 `http://127.0.0.1:8000/sse`）发起 HTTP 请求来建立 SSE 连接。
+ MCP 服务端收到请求以后，会给 MCP 客户端发送第一个 event，来推送客户端调用工具时应该使用的 URL 路由（比如 `/message?sessionId=35e1b1c7-e050-4107-bf3c-efad5709776b`）。需要注意，SSE 连接建立后，就没有被断开，稍后还会被使用。
+ 如果接下来 MCP 客户端需要调用工具，那么 MCP 客户端需要向 `/message?sessionId=35e1b1c7-e050-4107-bf3c-efad5709776b` 发起一个 POST 请求，并通过 HTTP Body 传递调用的工具名称和参数。
+ 此时，重点来了，MCP 服务的接收到来自客户端的请求，会直接返回 202 状态码。但是，并不会返回工具调用结果。工具调用结果是 MCP 服务端通过之前建立的 SSE 连接推送到客户端的。（你是否还记得 Agent 日志中疯狂输出 202 响应状态码的 HTTP 请求，其实就是这个请求）

我们可以在使用 Inspector 调用工具时抓一下 HTTP 包，来验证这个过程。

如下是工具调用时的 HTTP 请求：

![工具调用](tool-call.png)

可以发现，这个 POST 请求状态码的确是 202。

再回到 MCP 客户端和服务端建立连接的那条 SSE 的 HTTP 请求：

![工具响应](tool-call-result.png)

可以发现，工具调用结果果然是通过 SSE 推送给 MCP 客户端的。

#### 陷入僵局

了解了 MCP 协议调用流程，我们继续回过来排查问题。

其实这个时候，我更倾向于是 Higress 因为某种原因，没有通过 `/sse` 接口返回消息，从而导致 Agent 不停的重试，向 `/message` 接口发起请求。但是 Higress 日志现在又看不出什么问题。

现在已经陷入僵局，无法通过表象来快速排查出问题所在了。所以，我打算深入 Agent 来探索问题根因。

这时，我有两条路径可以选择：

1. 看源码，Agent 是基于 [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) 开发的，所以要看一下 OpenAI Agents SDK 源码中调用 MCP 的逻辑，并尝试输出更多的日志，来辅助排查问题。
2. 直接抓包，看一下 Agent 聊天过程中的整个请求链路，排查一下存在什么问题。

如果是 Go 程序，按照我的习惯，我更倾向于通过查看源码来排查问题。不过 Agent 是用 Python 开发的，而以我的了解，很多 AI 相关的包代码写的都比较绕，阅读代码要花费大量时间。所以我决定采用更省时的路径：抓包。

#### 抓包

不过，因为 Agent 程序是部署在 kubernetes 环境中的，以 Pod 形式部署。所以抓包要稍微麻烦一些。

由于 Agent 程序镜像中没有抓包程序，所以就要通过 kubernetes 提供的 debug 模式，来进行抓包了。

假如 agent 程序在 kubernetes 集群中部署为如下 Pod：

```bash
# 查看 agent pod
$ kubectl get pod
NAME     READY   STATUS    RESTARTS   AGE
agent    1/1     Running   0          16s
```

那么可以按照如下步骤来进行抓包：

```bash
# 首先通过 kubectl 的 debug 命令基于 netshoot 镜像拉起一个新的容器，并连接到 agent pod
# 这一步会在 agent pod 中新注入一个 debug 容器，与 agent 的容器共用同一个 network namespace
# 这样，就能在 netshoot 容器中通过 tcpdump 等抓包工具，对 agent 容器下的网络进行抓包了
# 命令格式：kubectl -n <namespace> debug <目标 pod> -it --image=<debug 容器镜像> --target=<目标容器>
$ kubectl -n default debug agent -it --image=nicolaka/netshoot:latest --target=agent

# 现在我们已经顺利进入 debug 容器，就可以通过 tcpdump 命令来进行抓包了
# 抓包内容会被保存到 capture.pcap 文件中
# tcpdump -i eth0 -w capture.pcap

# 在抓包过程中，我们可以在 AI Agent 聊天窗口复现问题，以记录所有网络包
# 我们主要应该抓两个包：
# 1. 第一个是 Agent 成功调用 MCP 工具的完整链路
# 2. 第二个是 Agent 调用 MCP 无返回结果的链路
# 这样在稍后分析问题的时候，可以进行对比分析
# ...

# 抓包完成后，就可以回到宿主机，通过 kubectl cp 命令从 debug 容器复制抓包文件 capture.pcap 到容器外部
# 注意这里的 debug 容器名称 debugger-gj29f 是可以在 pod 的 spec.ephemeralContainers.name 中找到的
# 在执行 kubectl debug 命令进入容器的过程中其实也会展示出来
$ kubectl cp -n default agent:/root/capture.pcap ./capture.pcap -c debugger-gj29f

# 接下来把这个 capture.pcap 下载到本地电脑，就可以对网络包进行回放了
# ...
```

#### 定位问题根因

在本地电脑，我们可以使用像 Wireshark 这样的工具来打开 `capture.pcap` 文件：

![tcpdump caputre](caputre.png)

这里我通过 `ip.addr == 10.97.134.31 && http` 来对网络包进行了过滤，专门针对 Agent 和 Higress 之间的请求进行分析。

截图中 `GET /mcp-server/mcp-server-54/sse HTTP/1.1` 请求就是一次 MCP 客户端到服务端的 SSE 连接的建立。

然后我们接着来分析一次成功的 MCP 工具调用：

![tool call](tool-call-success.png)

这里有一条 `POST /mcp-server/mcp-server-54?sessionId=d79fa662-b3cf-4106-9af5-671c5816453f HTTP/1.1` 请求（这个 URL 是 Higress 生成，等价前文中提到的 `/message` 路由），就是用来进行工具调用的。

请求参数如下：

```json
{
    "method": "tools/call",
    "params": {
        "name": "GetLeadId",
        "arguments": {
            "q": "SELECT Id FROM Lead WHERE Name = '江湖十年'"
        }
    },
    "jsonrpc": "2.0",
    "id": 2
}
```

这是一条成功的工具调用请求。

接下来，就需要找一下 Agent 调用 MCP 服务无返回结果的请求了。

以下是 Agent 同时选择了三个 MCP 服务时，MCP 客户端建立的三条 SSE 连接请求：

![SSE](sse.png)

现在，我们在请求包中找到一条 Agent 调用 MCP 服务无返回结果的请求：

![tool call](tool-call-failed.png)

可以看到，这里有非常多连续的 POST 请求，并且都返回了 202 状态码，这正是我们需要找的请求。

随意选择一个请求，其请求参数如下：

```json
{
    "jsonrpc": "2.0",
    "id": "2025-07-01T12:57:49Z",
    "result": {

    }
}
```

可以发现，这个请求并没有调用工具！而截图中所有的 POST 请求，全部都是类似的请求。

那么到这个时候，你可能就更加疑惑了，为什么会产生这样一个请求，并且 Agent 一直不停的重试？是不是要继续深入 OpenAI Agents SDK 源码来一探究竟了呢？

其实问题分析到这里，已经不用继续排查下去了，我已经猜到了一种可能的问题所在——MCP 工具重名了！

我们再来回顾一下问题：Agent 选择一个 MCP 进行聊天的时候，可以正常调用工具并返回结果。但是，如果 Agent 选择了多个 MCP 进行聊天，Agent 就不会有任何结果返回。

基于抓包中 MCP 客户端的表现，Agent 一直在发送非工具调用的请求，那么大概率是因为它不知道应该调用哪个工具了。而 MCP 官方协议中有明确规定，MCP 工具名称必须唯一。所以，如果熟悉 MCP 协议，那么这个问题不难猜到。

然后我回到 Inspector 中进行了验证，发现确实有两个 MCP 服务提供的工具列表中有一个工具重名了。

### 问题解决

现在，问题已经排查出来了，那么就需要解决问题了。

而解决这个问题的方案有很多，涉及临时性解决，和彻底解决。

#### 临时解决

如果要临时性的解决这个问题，那么只需要删除工具名称重复的 MCP，然后修改工具名称，最后通过 Higress 一键转换工具，重新转换成 MCP 服务即可。

Higress 工具是通过 OpenAPI 文档来进行转换的，一个 OpenAPI 文档会被转换成一个 MCP 服务。而文档中的每一个接口，就会作为 MCP 服务中的一个工具。工具名称则是通过一个叫 `operationId` 的字段来标识的。

如下面这段 OpenAPI 文档片段：

```yaml
paths:
  /users/{userid}/address:
    parameters:
      - name: userid
        in: path
        required: true
        description: the user identifier, as userId
        schema:
          type: string
    # linked operation
    get:
      operationId: getUserAddress
      responses:
        '200':
          description: the user's address
```

这个接口转换成 MCP 工具以后，工具名称就是 `getUserAddress`。如果未提供此字段，则 Higress 工具会根据 URL 中的 PATH 来自动生成工具名。

所以，修改这个 `operationId` 字段的值，就能解决 MCP 工具重名的问题。

> NOTE:
>
> 在标准的 OpenAPI 规范中，`operationId` 本身就是一个唯一字符串，用于标识接口唯一性。`operationId` 值区分大小写 ，其他工具和库可以使用 `operationId` 来作为唯一标识。因此，Higress 会用此字段作为工具名称。
>
> 你可以在 [https://swagger.io/specification/](https://swagger.io/specification/) 这里找到关于 `operationId` 字段的说明。
>

#### 彻底解决

要想彻底解决这个问题，就需要保证一键转换的 MCP 服务工具名称必须唯一，那么唯一性校验就是必须的。

这时候通常有两种方案：

+ 工具名称由转换工具自动生成（比如每个 MCP 服务加一个统一的业务前缀），并且生成后自动校验唯一性，如果不唯一，就重新生成。
+ 工具名称可以交给用户自定义，但是转换过程中需要校验唯一性。

不管采用哪种方案，我们要校验的都不是某一个 MCP 服务中工具名称的唯一性，而是所有 MCP 服务中的所有工具名称全局唯一。

这是一个开放性问题，你有什么思路，可以一起探讨交流。

### 总结

本文记录了我排查并解决 Agent 同时选择多个 MCP 服务后聊天无内容返回的问题。通过一步步深入排查，最终发现这个问题引发的原因是 MCP 工具名称重复所致。要想解决这个问题也非常简单，只需要将 MCP 工具名称修改为唯一标识即可。

问题虽然解决了，但想想这还真是一个愚蠢的问题！因为我之前学习 MCP 协议的时候，其实是有注意到工具名称不能重复的问题。并且我也亲口告诉过负责一键转换 MCP 服务的同学在操作时要给 OpenAPI 文档中每个 URL 路由设置一个唯一的 `operationId`，来作为 MCP 工具名称。但你知道的，只要是程序中不进行严格限制，人为操作就一定会出错，这是无法避免的问题。

而在整个问题排查过程中，我有很多次机会应该能够想到这是 MCP 工具重名所导致的问题，但我一次都没有意识到，我早该想到是这个问题的 :(。

其实在使用 Inspector 工具时，最应该发现问题的，因为 Inspector 会清晰的罗列出 MCP 服务所有工具的列表，只是我没有细心观察。这就导致了后续一系列的徒劳操作，一顿操作猛如虎，定睛一看原地杵，兜兜转转问题竟然又回到了 MCP 基础知识上。

为了显得我花费时间排查这个问题还是有点价值的，就写篇文章记录下吧，不然觉得这波操作又浪费了好多时间。

因为排查过程中选择了抓包，最终我也没去分析 OpenAI Agents SDK 的源码，以后有机会再深入研究吧 -_-!。

最后，其实 MCP [官方文档](https://modelcontextprotocol.io/docs/concepts/tools)中有明确说明工具名称必须唯一。

> Like [resources](https://modelcontextprotocol.io/docs/concepts/resources), tools are identified by unique names and can include descriptions to guide their usage. However, unlike resources, tools represent dynamic operations that can modify state or interact with external systems.
>

与 `resources` 一样，`tools` 名称必须是唯一的。

并且，MCP 官方也给出了工具名称冲突的[最佳实践](https://modelcontextprotocol.io/docs/concepts/tools#tool-name-conflicts)。如果有两个 MCP 服务 分别为 `web1` 和 `web2`，并且都要公开一个叫 `search_web` 的工具。那么可以采用如下策略来实现 MCP 工具名称全局唯一：

+ 将服务名称与工具名称使用连接符连起来。如 `web1___search_web` 和 `web2___search_web`。
+ 为工具名称生成随机前缀。如 `jrwxs___search_web` 和 `6cq52___search_web`。
+ 使用服务器 URI 作为工具名称的前缀。如 `web1.example.com:search_web` 和 `web2.example.com:search_web`。

你更喜欢哪种方式呢？欢迎留言区交流。

希望此文能对你有所启发。

**延伸阅读**

+ OpenAPI Specification：[https://swagger.io/specification/](https://swagger.io/specification/)
+ OpenAI Agents SDK：[https://openai.github.io/openai-agents-python/](https://openai.github.io/openai-agents-python/)
+ MCP 协议 HTTP with SSE：[https://modelcontextprotocol.io/specification/2024-11-05/basic/transports#http-with-sse](https://modelcontextprotocol.io/specification/2024-11-05/basic/transports#http-with-sse)
+ MCP Tool 名称冲突：[https://modelcontextprotocol.io/docs/concepts/tools#tool-name-conflicts](https://modelcontextprotocol.io/docs/concepts/tools#tool-name-conflicts)
+ MCP Inspector：[https://modelcontextprotocol.io/docs/tools/inspector](https://modelcontextprotocol.io/docs/tools/inspector)
+ 使用 MCP Gateway 一键将你的 HTTP 接口转换成 MCP Server：[https://mp.weixin.qq.com/s/JU10VIvZnCvca1kTkjZufQ](https://mp.weixin.qq.com/s/JU10VIvZnCvca1kTkjZufQ)
+ Higress OpenAPI to MCP Server 工具：[https://github.com/higress-group/openapi-to-mcpserver](https://github.com/higress-group/openapi-to-mcpserver)
+ 本文永久地址：[https://jianghushinian.cn/2025/07/05/agent-mcp-empty-result/](https://jianghushinian.cn/2025/07/05/agent-mcp-empty-result/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

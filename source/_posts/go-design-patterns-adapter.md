---
title: '适配器模式在 Go 语言中的应用'
date: 2024-08-04 09:15:28
tags:
- Go
- 设计模式
categories:
- Go
---

前段时间我负责对一个项目进行临时性的技术方案改造，用到了适配器模式，今天就来跟大家简单分享下适配器模式在 Go 语言中的应用。

<!-- more -->

### 适配器模式

适配器模式（Adapter Pattern）是 23 种经典设计模式中的一种，属于行为型模式，**它允许不兼容的接口协同工作**。该模式通过创建一个适配器类，封装不兼容的接口，并对外提供一个兼容的接口。

[《设计模式：可复用面向对象软件的基础》](https://book.douban.com/subject/34262305/)一书中对适配器的意图定义如下：

> 将一个类的接口转换成客户希望的另外一个接口。Adapter 模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

这么说比较抽象，我来举一个现实生活中的例子，帮助你理解。

#### 生活中的适配器模式

在现实生活中，我们平时使用的手机、电脑等充电器，实际上叫电源适配器（Power adapter）。

下图中插座上有一个白色的电源适配器和一个黑色的普通插头：

![电源适配器](adapter.jpeg)

因为手机和电脑无法直接接收 220V 交流电，一般只会接收如 9V3A 这种直流小电压输入才能进行充电。所以不能直接使用一个普通插头来为手机充电，而是需要使用电源适配器进行转换。

这就是现实生活中的**适配器模式**。

#### Go 语言中的适配器模式

那么在 Go 代码中如何实现适配器呢？我们以一个简单的支付系统作为示例进行讲解。

比如，我们有一个支付的功能，核心代码如下：

```go
// PaymentProcessor 是一个统一的支付接口
type PaymentProcessor interface {
	Pay(amount float64)
}

// OldPaymentSystem 是旧的支付系统
type OldPaymentSystem struct{}

func (ops *OldPaymentSystem) Pay(amount float64) {
	fmt.Printf("Processing payment of %.2f using old payment system\n", amount)
}
```

可以这样使用：

```go
// 声明支付接口
var processor PaymentProcessor

// 使用旧支付系统
processor = &OldPaymentSystem{}
processor.Pay(100)
```

现在我们需要接入一个新支付系统，核心代码如下：

```go
// NewPaymentSystem 是新的支付系统
type NewPaymentSystem struct{}

func (nps *NewPaymentSystem) MakePayment(amount float64) {
	fmt.Printf("Making payment of %.2f using new payment system\n", amount)
}
```

新的支付系统没有 `Pay` 方法，所以没有实现统一支付接口 `PaymentProcessor`。

我们不能按照原来使用旧的支付系统的方式来使用新的支付系统：

```go
processor = &NewPaymentSystem{}
processor.Pay(100)
```

这段代码是行不通的，会编译报错。

因为 `NewPaymentSystem` 并没有实现 `PaymentProcessor` 接口，无法赋值给 `processor` 变量。

此时，我们**可以创建一个适配器，来解决此问题**：

```go
// NewPaymentAdapter 是新支付系统的适配器
type NewPaymentAdapter struct {
	// 内部持有新支付系统
	NewSystem *NewPaymentSystem
}

func (npa *NewPaymentAdapter) Pay(amount float64) {
	npa.NewSystem.MakePayment(amount)
}
```

定义 `NewPaymentAdapter` 作为新支付系统 `NewPaymentSystem` 的适配器，其内部持有新的支付系统 `NewPaymentSystem`，并为适配器定义 `Pay` 方法。

有了适配器，我们就可以按照原来使用旧的支付系统的方式来使用新的支付系统了：

```go
// 使用新支付系统
newPayment := &NewPaymentSystem{}
// 使用适配器模式
processor = &NewPaymentAdapter{NewSystem: newPayment}
processor.Pay(200)
```

因为适配器 `NewPaymentAdapter` 实现了 `PaymentProcessor` 接口，所以经过它包装的新支付系统 `NewPaymentSystem` 可以赋值给 `processor` 变量。

在调用 `processor.Pay(200)` 时，`NewPaymentAdapter.Pay` 方法内部会调用 `npa.NewSystem.MakePayment(200)` 方法，将请求转发到新支付系统。

我们也就实现了使用统一的支付接口，来完成使用新支付系统进行支付。

**这里的 `NewPaymentAdapter` 就是适配器模式中的「适配器」**。

**`NewPaymentAdapter` 实现了将不兼容的接口（`NewPaymentSystem`）转换为成客户希望的另外一个接口（`PaymentProcessor`），使得不兼容的二者可以协同工作**。

这就是一个简单的适配器模式在 Go 语言中的应用示例。

### 生产实践

通过前文的描述，我们对适配器模式有了一定了解。

接下来我带你看下我在生产实践过程中是如何使用适配器模式的。

#### 多云管理平台的应用

我曾经参与开发过一个多云管理平台，这是一个用 Go 编写的管理多个第三方云主机的平台。可以在管理平台上操作如阿里云、腾讯云、AWS 等平台的云主机，实现创建、续费、删除等。

每家云主机厂商都提供了非常方便的 Go SDK 来操作云主机，遗憾的是，每家的 SDK 接口又都不一样。

为了统一操作，抹平不同云主机厂商之间 SDK 接口的差异，我们定义了如下接口：

```go
// Provider 定义云厂商统一接口
type Provider interface {
	// Type 返回 Provider 类型
	Type() ProviderType
	// RunInstance 创建云主机
	RunInstance(r *RunInstanceRequest) (*RunInstanceResponse, error)
	// ...
}
```

> NOTE: 这里我仅列出接口的了 `Type` 和 `RunInstance` 两个方法，代码意图不变。

所有云主机相关操作，都必须遵循 `Provider` 接口。

为了区分不同云主机厂商类型，我们还需要为每个云主机厂商定义一个常量：

```go
type ProviderType string

const (
	ProviderTypeAliyun  ProviderType = "aliyun"
	ProviderTypeTencent ProviderType = "tencent"
	// ...
)
```

接下来就要为每一个云主机厂商都各自封装一个 `XxxProvider` 来适配 `Provider` 接口。

阿里云 `Provider` 定义如下：

```go
// AliCloudProvider 阿里云 Provider
type AliCloudProvider struct {
	typ ProviderType
	// 包装 alibaba-cloud-sdk-go
}

func NewAliCloudProvider(typ ProviderType) *AliCloudProvider {
	return &AliCloudProvider{
		typ: typ,
		// ...
	}
}

func (a AliCloudProvider) Type() ProviderType {
	return a.typ
}

// RunInstance https://help.aliyun.com/zh/ecs/developer-reference/api-ecs-2014-05-26-runinstances
func (a AliCloudProvider) RunInstance(r *RunInstanceRequest) (*RunInstanceResponse, error) {
	panic("implement me")
}
```

`AliCloudProvider` 是对阿里云提供的 `alibaba-cloud-sdk-go` 的包装，其实现了 `Provider` 接口，并将 `Provider` 所有方法转换成对 `alibaba-cloud-sdk-go` 的调用。

腾讯云 `Provider` 定义如下：

```go
// TencentCloudProvider 腾讯云 Provider
type TencentCloudProvider struct {
	typ ProviderType
	// 包装 tencentcloud-sdk-go
}

func NewTencentCloudProvider(typ ProviderType) *TencentCloudProvider {
	return &TencentCloudProvider{
		typ: typ,
		// ...
	}
}

func (t TencentCloudProvider) Type() ProviderType {
	return t.typ
}

// RunInstance https://cloud.tencent.com/document/api/213/15730
func (t TencentCloudProvider) RunInstance(r *RunInstanceRequest) (*RunInstanceResponse, error) {
	panic("implement me")
}
```

同理，`TencentCloudProvider` 是对腾讯云提供的 `tencentcloud-sdk-go` 的包装，其实现了 `Provider` 接口，并将 `Provider` 所有方法转换成对 `tencentcloud-sdk-go` 的调用。

定义 `Provider` 构造函数如下：

```go
func NewProvider(typ ProviderType) Provider {
	switch typ {
	case ProviderTypeAliyun:
		return NewAliCloudProvider(typ)
	case ProviderTypeTencent:
		return NewTencentCloudProvider(typ)
	default:
		panic("unknown provider")
	}
	return nil
}
```

在使用时，我们可以根据多云管理平台前端用户选择的不同云主机厂商 `ProviderType`，来构造不同的 `Provider` 对象，然后去操作云主机，实现创建、删除等。

使用示例如下：

```go
var p Provider

// 阿里云
p = NewProvider(ProviderTypeAliyun)
resp, err := p.RunInstance(&RunInstanceRequest{})
fmt.Println(resp, err)

// 腾讯云
p = NewProvider(ProviderTypeTencent)
resp, err = p.RunInstance(&RunInstanceRequest{})
fmt.Println(resp, err)
```

以上，就是我从多云管理平台生产实践中抽离出来的适配器模式使用示例。

这里去掉了业务逻辑，仅保留了代码整体思路。这个示例中适配器代码的命名并没有使用 `Adapter`，但这其实也是一种适配器模式。

`AliCloudProvider` 和 `TencentCloudProvider` 都是适配器，它们分别包装了阿里云和腾讯云的 SDK，然后适配云厂商统一操作接口 `Provider`。

在使用时，可以根据需要，指定 `ProviderType` 构造不同类型的 `Provider`，然后操作对应厂商的云主机。

#### 模型训练平台的应用

再举一个我在生产实践中使用适配器模式的例子。

前段时间我负责对一个项目进行临时性的技术方案改造，就用到了适配器模式。

这是一个支持大语言模型训练和推理的综合平台，本小结以部署模型推理任务为例进行讲解。

部署推理任务步骤如下：

1. 构造推理任务的 Kubernetes `Deployment` 资源。

2. 部署推理任务 `Deployment` 到 Kubernetes 集群。

示例代码大致如下：

```go
import appsv1 "k8s.io/api/apps/v1"

func BuildDeployment() *appsv1.Deployment {
	// ...
	return nil
}

func DeployPredictService(deployment *appsv1.Deployment) error {
	// ...
	return nil
}
```

`BuildDeployment` 用于构造 `Deployment` 资源对象。

`DeployPredictService` 用于将这个 `Deployment` 资源对象部署到 Kubernetes 中，来启动模型推理服务。

所以部署模型推理服务代码流程如下：

```go
deployment := BuildDeployment()
err := DeployPredictService(deployment)
fmt.Println(err)
```

现在要对这个平台代码进行临时性改造，需要新建一个 `feature` 分支，使用微软开源的 [OpenPAI](https://github.com/microsoft/pai) 平台来部署模型推理服务，而不再直接使用 Kubernetes `Deployment` 资源进行部署。

> NOTE: OpenPAI 是一个提供完整的人工智能模型训练和资源管理能力开源平台，它易于扩展，支持各种规模的 on-premise、on-cloud 和混合环境。

OpenPAI 平台有自己的 CRD（Custom Resource Definition）来定义和管理部署资源，并且 OpenPAI 提供了 RESTful 接口，可以创建 CR（Custom Resource） 来生成 OpenPAI 的资源对象。

因为是临时性的改造，未来有很大不确定性，并且旧有代码经过多人次维护，结构变来变去早已成为屎山，借着这一次的需求改造，索性引入适配器模式，来规范代码结构。

首先，抽象出一个模型推理任务统一操作接口 `Predictor`：

```go
type Predictor interface {
	Deploy(deployment *appsv1.Deployment) error
	Scale(namespace, name string, replicas int) error
	Delete(namespace, name string) error
	// ...
}
```

`Predictor` 接口定义了模型推理服务的所有操作，包括部署、伸缩、删除等。

然后为 OpenPAI 提供一个适配器来适配 `Predictor` 接口：

```go
type OpenPAIAdapter struct {
	// ...
}

func NewOpenPAIAdapter() *OpenPAIAdapter {
	return &OpenPAIAdapter{}
}

func (o *OpenPAIAdapter) Deploy(deployment *appsv1.Deployment) error {
	// 将 K8s Deployment 资源转换成 OpenPAI 的 RESTful 接口调用
	panic("implement me")
}

func (o *OpenPAIAdapter) Scale(namespace, name string, replicas int) error {
	panic("implement me")
}

func (o *OpenPAIAdapter) Delete(namespace, name string) error {
	panic("implement me")
}
```

我们可以在 `OpenPAIAdapter` 的每个方法中，将对 Kubernetes `Deployment` 资源的每种操作，转换成对应的 OpenPAI 的 RESTful 接口调用。

使用 `OpenPAIAdapter` 适配器部署模型推理服务代码如下：

```go
// 推理任务统一接口
var predictor Predictor

// 根据业务逻辑构造不同的适配器
// switch expr {
// case:
predictor = NewOpenPAIAdapter()
// }

// 部署推理服务
deployment := BuildDeployment()
err := predictor.Deploy(deployment)
fmt.Println(err)
```

通过引入适配器模式，来支撑 OpenPAI 资源的部署，能够最小化改动代码，且定义统一的规范。

这里 `Predictor` 接口其实是围绕着 Kubernetes `Deployment` 资源的操作来定义的。

之所以这样定义，实际上是为了对代码只做最小化改动。因为之前的代码全部都是围绕一个 `Deployment` 资源来写的，且开发周期有限，抽象出 `Predictor` 接口已经足以引入适配器模式，以此来方便部署 OpenPAI 资源。

并且，将来哪一天想要改回来，再次使用 Kubernetes `Deployment` 资源部署模型推理服务，只需要将旧代码也进行适配，写一个 `DeploymentAdapter` 适配器即可。

以后可能还会对接其他平台，都可以参照 `OpenPAIAdapter` 来设计新的适配器。

以上，同样是我从生产实践中抽离出来的适配器模式使用示例。

### 总结

本文主要介绍了什么是适配器模式，以及适配器模式在 Go 语言中的应用。

既然这个设计模式叫「适配器」模式，那么其作用必然是为了适配，更多的时候是**作为一种事后补偿机制**。所以其实这个设计模式往往并不是首选。

我列举的适配器模式应用场景，都是我在生产实践中的探索。

在讲解什么是适配器模式的示例中，为了兼容旧版本支付接口，我们将新版本支付接口做了修改，对旧版本支付接口进行适配，这其实就是一种所谓的事后补偿机制。

在多云管理平台使用示例中，适配器模式的应用属于统一多个外部系统编程接口。还有一种场景最为常见，就是对接多个第三方支付系统，如支付宝、微信等。

在模型训练平台使用示例中，适配器模式的应用场景属于替换依赖的外部系统，当我们把项目中依赖的一个外部系统替换为另一个外部系统的时候，使用适配器模式，可以减少对代码的改动。

本文介绍的这几种应用场景算是适配器模式典型的使用场景了。

你认为适配器模式还有哪些应用场景，可以一起交流学习。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/design-patterns/adapter) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

- 《设计模式：可复用面向对象软件的基础》：https://book.douban.com/subject/34262305/
- 51 | 适配器模式：代理、适配器、桥接、装饰，这四个模式有何区别？：https://time.geekbang.org/column/article/205912
- Kubernetes 中的 Adapter 示例：https://github.com/kubernetes/kubernetes/blob/v1.30.0/pkg/controller/replication/conversion.go#L56
- Go 常见设计模式之选项模式：https://jianghushinian.cn/2021/12/12/Golang-常见设计模式之选项模式/
- Go 常见设计模式之装饰模式：https://jianghushinian.cn/2022/02/28/Golang-常见设计模式之装饰模式/
- Go 常见设计模式之单例模式：https://jianghushinian.cn/2022/03/04/Golang-常见设计模式之单例模式/
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/design-patterns/adapter

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn

---
title: 在 Go 语言单元测试中如何解决 Redis 存储依赖问题
date: 2023-07-18 12:20:48
tags:
- Go
- Test
categories:
- Go
---

在编写单元测试时，除了 MySQL 这个外部存储依赖，Redis 应该是另一个最为常见的外部存储依赖了。我在[《在 Go 语言单元测试中如何解决 MySQL 存储依赖问题》](https://jianghushinian.cn/2023/07/16/how-to-resolve-mysql-dependencies-in-go-testing/)一文中讲解了如何解决 MySQL 外部依赖，本文就来讲解下如何解决 Redis 外部依赖。

<!-- more -->

### 登录程序示例

在 Web 开发中，登录需求是一个较为常见的功能。假设我们有一个 `Login` 函数，可以实现用户登录功能。它接收用户手机号 + 短信验证码，然后根据手机号从 Redis 中获取保存的验证码（验证码通常是在发送验证码这一操作时保存的），如果 Redis 中验证码与用户输入的验证码相同，则表示用户信息正确，然后生成一个随机 token 作为登录凭证，之后先将 token 写入 Redis 中，再返回给用户，表示登录操作成功。

程序代码实现如下：

```go
func Login(mobile, smsCode string, rdb *redis.Client, generateToken func(int) (string, error)) (string, error) {
	ctx := context.Background()

	// 查找验证码
	captcha, err := GetSmsCaptchaFromRedis(ctx, rdb, mobile)
	if err != nil {
		if err == redis.Nil {
			return "", fmt.Errorf("invalid sms code or expired")
		}
		return "", err
	}

	if captcha != smsCode {
		return "", fmt.Errorf("invalid sms code")
	}

	// 登录，生成 token 并写入 Redis
	token, _ := generateToken(32)
	err = SetAuthTokenToRedis(ctx, rdb, token, mobile)
	if err != nil {
		return "", err
	}

	return token, nil
}
```

`Login` 函数有 4 个参数，分别是用户手机号、验证码、Redis 客户端连接对象、辅助生成随机 token 的函数。

Redis 客户端连接对象 `*redis.Client` 属于 `github.com/redis/go-redis/v9` 包。

我们可以使用如下方式获得：

```go
func NewRedisClient() *redis.Client {
	return redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
	})
}
```

`generateToken` 用来生成随机长度 token，定义如下：

```go
func GenerateToken(length int) (string, error) {
	token := make([]byte, length)
	_, err := rand.Read(token)
	if err != nil {
		return "", err
	}
	return base64.URLEncoding.EncodeToString(token)[:length], nil
}
```

我们还要为 Redis 操作编写几个函数，用来存取 Redis 中的验证码和 token：

```go
var (
	smsCaptchaExpire    = 5 * time.Minute
	smsCaptchaKeyPrefix = "sms:captcha:%s"

	authTokenExpire    = 24 * time.Hour
	authTokenKeyPrefix = "auth:token:%s"
)

func SetSmsCaptchaToRedis(ctx context.Context, redis *redis.Client, mobile, captcha string) error {
	key := fmt.Sprintf(smsCaptchaKeyPrefix, mobile)
	return redis.Set(ctx, key, captcha, smsCaptchaExpire).Err()
}

func GetSmsCaptchaFromRedis(ctx context.Context, redis *redis.Client, mobile string) (string, error) {
	key := fmt.Sprintf(smsCaptchaKeyPrefix, mobile)
	return redis.Get(ctx, key).Result()
}

func SetAuthTokenToRedis(ctx context.Context, redis *redis.Client, token, mobile string) error {
	key := fmt.Sprintf(authTokenKeyPrefix, mobile)
	return redis.Set(ctx, key, token, authTokenExpire).Err()
}

func GetAuthTokenFromRedis(ctx context.Context, redis *redis.Client, token string) (string, error) {
	key := fmt.Sprintf(authTokenKeyPrefix, token)
	return redis.Get(ctx, key).Result()
}
```

`Login` 函数使用方式如下：

```go
func main() {
	rdb := NewRedisClient()
	token, err := Login("13800001111", "123456", rdb, GenerateToken)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(token)
}
```

### 使用 redismock 测试

现在，我们要对 `Login` 函数进行单元测试。

`Login` 函数依赖了 `*redis.Client` 以及 `generateToken` 函数。

由于我们设计的代码是 `Login` 函数直接依赖了 `*redis.Client`，没有通过接口来解耦，所以不能使用 `gomock` 工具来生成 Mock 代码。

不过，我们可以看看 `go-redis` 包的源码仓库有没有什么线索。

很幸运，在 `go-redis` 包的 [README.md](https://github.com/redis/go-redis#ecosystem) 文档里，我们可以看到一个 [Redis Mock](https://github.com/go-redis/redismock) 链接：

![Redis Mock](redis-mock.png)

点击进去，我们就来到了一个叫 `redismock` 的仓库，`redismock` 为我们实现了一个模拟的 Redis 客户端。

使用如下方式安装 `redismock`：

```bash
$ go get github.com/go-redis/redismock/v9
```

使用如下方式导入 `redismock`：

```go
import "github.com/go-redis/redismock/v9"
```

切记安装和导入的 `redismock` 包版本要与 `go-redis` 包版本一致，这里都为 `v9`。

可以通过如下方式快速创建一个 Redis 客户端 `rdb`，以及客户端 Mock 对象 `mock`：

```go
rdb, mock := redismock.NewClientMock()
```

在测试代码中，调用 `Login` 函数时，就可以使用这个 `rdb` 作为 Redis 客户端了。

`mock` 对象提供了 `ExpectXxx` 方法，用来指定 `rdb` 客户端预期会调用哪些方法以及对应参数。

```go
// login success
mock.ExpectGet("sms:captcha:13800138000").SetVal("123456")
mock.ExpectSet("auth:token:Ta5EVtRgUD-HFmRwrujAwKZnx247lFfe", "13800138000", 24*time.Hour).SetVal("OK")
```

`mock.ExpectGet` 表示期待一个 Redis `Get` 操作，Key 为 `sms:captcha:13800138000`，`SetVal("123456")` 用来设置当前 `Get` 操作返回值为 `123456`。

同理，`mock.ExpectSet` 表示期待一个 Redis `Set` 操作，Key 为 `auth:token:Ta5EVtRgUD-HFmRwrujAwKZnx247lFfe`，Value 为 `13800138000`，过期时间为 `24*time.Hour`，返回 `OK` 表示这个 `Set` 操作成功。

以上指定的两个预期方法调用，是用来匹配 `Login` 成功时的用例。

`Login` 函数还有两种失败情况，当通过 `GetSmsCaptchaFromRedis` 函数查询 Redis 中验证码不存在时，返回 `invalid sms code or expired` 错误。当从 Redis 中查询的验证码与用户传递进来的验证码不匹配时，返回 `invalid sms code` 错误。

这两种用例可以按照如下方式模拟：

```go
// invalid sms code or expired
mock.ExpectGet("sms:captcha:13900139000").RedisNil()
// invalid sms code
mock.ExpectGet("sms:captcha:13700137000").SetVal("123123")
```

现在，我们已经解决了 Redis 依赖，还需要解决 `generateToken` 函数依赖。

这时候 Fake object 就派上用场了：

```go
func fakeGenerateToken(int) (string, error) {
	return "Ta5EVtRgUD-HFmRwrujAwKZnx247lFfe", nil
}
```

我们使用 `fakeGenerateToken` 函数来替代 `GenerateToken` 函数，这样生成的 token 就固定下来了，方便测试。

`Login` 函数完整单元测试代码实现如下：

```go
func TestLogin(t *testing.T) {
	// mock redis client
	rdb, mock := redismock.NewClientMock()

	// login success
	mock.ExpectGet("sms:captcha:13800138000").SetVal("123456")
	mock.ExpectSet("auth:token:Ta5EVtRgUD-HFmRwrujAwKZnx247lFfe", "13800138000", 24*time.Hour).SetVal("OK")

	// invalid sms code or expired
	mock.ExpectGet("sms:captcha:13900139000").RedisNil()

	// invalid sms code
	mock.ExpectGet("sms:captcha:13700137000").SetVal("123123")

	type args struct {
		mobile  string
		smsCode string
	}
	tests := []struct {
		name    string
		args    args
		want    string
		wantErr string
	}{
		{
			name: "login success",
			args: args{
				mobile:  "13800138000",
				smsCode: "123456",
			},
			want: "Ta5EVtRgUD-HFmRwrujAwKZnx247lFfe",
		},
		{
			name: "invalid sms code or expired",
			args: args{
				mobile:  "13900139000",
				smsCode: "123459",
			},
			wantErr: "invalid sms code or expired",
		},
		{
			name: "invalid sms code",
			args: args{
				mobile:  "13700137000",
				smsCode: "123457",
			},
			wantErr: "invalid sms code",
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := Login(tt.args.mobile, tt.args.smsCode, rdb, fakeGenerateToken)
			if tt.wantErr != "" {
				assert.Error(t, err)
				assert.Equal(t, tt.wantErr, err.Error())
			} else {
				assert.NoError(t, err)
				assert.Equal(t, tt.want, got)
			}
		})
	}
}
```

这里使用了表格测试，提供了 3 个测试用例，覆盖了登录成功、验证码无效或过期、验证码无效 3 种场景。

使用 `go test` 来执行测试函数：

```bash
$ go test -v .                 
=== RUN   TestLogin
=== RUN   TestLogin/login_success
=== RUN   TestLogin/invalid_sms_code_or_expired
=== RUN   TestLogin/invalid_sms_code
--- PASS: TestLogin (0.00s)
    --- PASS: TestLogin/login_success (0.00s)
    --- PASS: TestLogin/invalid_sms_code_or_expired (0.00s)
    --- PASS: TestLogin/invalid_sms_code (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/redis    0.152s
```

测试通过。

`Login` 函数将 `*redis.Client` 和 `generateToken` 这两个外部依赖定义成了函数参数，而不是在函数内部直接使用这两个依赖。

这主要参考了「依赖注入」的思想，将依赖当作参数传入，而不是在函数内部直接引用。

这样，我们才有机会使用 Fake 对象 `fakeGenerateToken` 来替代真实对象 `GenerateToken`。

而对于 `*redis.Client`，我们也能够使用 `redismock` 提供的 Mock 对象来替代。

`redismock` 不仅能够模拟 RedisClient，它还支持模拟 RedisCluster，更多使用示例可以在[官方示例](https://github.com/go-redis/redismock/blob/master/example)中查看。

### 使用 Testcontainers 测试

虽然我们使用 `redismock` 提供的 Mock 对象解决了 `Login` 函数对 `*redis.Client` 的依赖问题。

但这需要运气，当我们使用其他数据库时，也许找不到现成的 Mock 库。

此时，我们还有另一个强大的工具「容器」可以使用。

如果程序所依赖的某个外部服务，实在找不到现成的 Mock 工具，自己实现 Fack object 又比较麻烦，这时就可以考虑使用容器来运行一个真正的外部服务了。

[Testcontainers](https://github.com/testcontainers/testcontainers-go) 就是用来解决这个问题的，我们可以用它来启动容器，运行任何外部服务。

`Testcontainers` 非常强大，不仅支持 Go 语言，还支持 Java、Python、Rust 等其他主流编程语言。它可以很容易地创建和清理基于容器的依赖，常被用于集成测试和冒烟测试。所以这也提醒我们在单元测试中慎用，因为容器也是一个外部依赖。

我们可以按照如下方式使用 `Testcontainers` 在容器中启动一个 Redis 服务：

```go
import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

// 在容器中运行一个 Redis 服务
func RunWithRedisInContainer() (*redis.Client, func()) {
	ctx := context.Background()

	// 创建容器请求参数
	req := testcontainers.ContainerRequest{
		Image:        "redis:6.0.20-alpine",                      // 指定容器镜像
		ExposedPorts: []string{"6379/tcp"},                       // 指定容器暴露端口
		WaitingFor:   wait.ForLog("Ready to accept connections"), // 等待输出容器 Ready 日志
	}

	// 创建 Redis 容器
	redisC, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	if err != nil {
		panic(fmt.Sprintf("failed to start container: %s", err.Error()))
	}

	// 获取容器中 Redis 连接地址，e.g. localhost:50351
	endpoint, err := redisC.Endpoint(ctx, "") // 如果暴露多个端口，可以指定第二个参数
	if err != nil {
		panic(fmt.Sprintf("failed to get endpoint: %s", err.Error()))
	}

	// 连接容器中的 Redis
	client := redis.NewClient(&redis.Options{
		Addr: endpoint,
	})

	// 返回 Redis Client 和 cleanup 函数
	return client, func() {
		if err := redisC.Terminate(ctx); err != nil {
			panic(fmt.Sprintf("failed to terminate container: %s", err.Error()))
		}
	}
}
```

代码中我写了比较详细的注释，就不带大家一一解释代码内容了。

我们可以将容器的启动和释放操作放到 `TestMain` 函数中，这样在执行测试函数之前先启动容器，然后进行测试，最后在测试结束时销毁容器。

```go
var rdbClient *redis.Client

func TestMain(m *testing.M) {
	client, f := RunWithRedisInContainer()
	defer f()
	rdbClient = client
	m.Run()
}
```

使用容器编写的 `Login` 单元测试函数如下：

```go
func TestLogin_by_container(t *testing.T) {
	// 准备测试数据
	err := SetSmsCaptchaToRedis(context.Background(), rdbClient, "18900001111", "123456")
	assert.NoError(t, err)

	// 测试登录成功情况
	gotToken, err := Login("18900001111", "123456", rdbClient, GenerateToken)
	assert.NoError(t, err)
	assert.Equal(t, 32, len(gotToken))

	// 检查 Redis 中是否存在 token
	gotMobile, err := GetAuthTokenFromRedis(context.Background(), rdbClient, gotToken)
	assert.NoError(t, err)
	assert.Equal(t, "18900001111", gotMobile)
}
```

现在因为有了容器的存在，我们有了一个真实的 Redis 服务。所以编写测试代码时，无需再考虑如何模拟 Redis 客户端，只需要使用通过 `RunWithRedisInContainer()` 函数创建的真实客户端 `rdbClient` 即可，一切操作都是真实的。

并且，我们也不再需要实现 `fakeGenerateToken` 函数来固定生成的 token，直接使用 `GenerateToken` 生成真实的随机 token 即可。想要验证得到的 token 是否正确，可以直接从 Redis 服务中读取。

执行测试前，确保主机上已经安装了 Docker，`Testcontainers` 会使用主机上的 Docker 来运行容器。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestLogin_by_container"
2023/07/17 22:59:34 github.com/testcontainers/testcontainers-go - Connected to docker: 
  Server Version: 20.10.21
  API Version: 1.41
  Operating System: Docker Desktop
  Total Memory: 7851 MB
2023/07/17 22:59:34 🐳 Creating container for image docker.io/testcontainers/ryuk:0.5.1
2023/07/17 22:59:34 ✅ Container created: 92e327ad7b70
2023/07/17 22:59:34 🐳 Starting container: 92e327ad7b70
2023/07/17 22:59:35 ✅ Container started: 92e327ad7b70
2023/07/17 22:59:35 🚧 Waiting for container id 92e327ad7b70 image: docker.io/testcontainers/ryuk:0.5.1. Waiting for: &{Port:8080/tcp timeout:<nil> PollInterval:100ms}
2023/07/17 22:59:35 🐳 Creating container for image redis:6.0.20-alpine
2023/07/17 22:59:35 ✅ Container created: 2b5e40d40af0
2023/07/17 22:59:35 🐳 Starting container: 2b5e40d40af0
2023/07/17 22:59:35 ✅ Container started: 2b5e40d40af0
2023/07/17 22:59:35 🚧 Waiting for container id 2b5e40d40af0 image: redis:6.0.20-alpine. Waiting for: &{timeout:<nil> Log:Ready to accept connections Occurrence:1 PollInterval:100ms}
=== RUN   TestLogin_by_container
--- PASS: TestLogin_by_container (0.00s)
PASS
2023/07/17 22:59:36 🐳 Terminating container: 2b5e40d40af0
2023/07/17 22:59:36 🚫 Container terminated: 2b5e40d40af0
ok      github.com/jianghushinian/blog-go-example/test/redis    1.545s
```

测试通过。

根据输出日志可以发现，我们的确在主机上创建了一个 Redis 容器来运行 Redis 服务：

```log
Creating container for image redis:6.0.20-alpine
```

容器 ID 为 `2b5e40d40af0`：

```log
Container created: 2b5e40d40af0
```

并且测试结束后清理了容器：

```log
Container terminated: 2b5e40d40af0
```

以上，我们就利用容器技术，为 `Login` 函数登录成功情况编写了一个测试用例，登录失败情况的测试用例就留做作业交给你自己来完成吧。

### 总结

本文向大家介绍了在 Go 中编写单元测试时，如何解决 Redis 外部依赖的问题。

值得庆幸的是 `redismock` 包提供了模拟的 Redis 客户端，方便我们在测试过程中替换 Redis 外部依赖。

但有些时候，我们可能找不到这种现成的第三方包。`Testcontainers` 库则为我们提供了另一种解决方案，运行一个真实的容器，以此来提供 Redis 服务。

不过，虽然 `Testcontainers` 足够强大，但不到万不得已，不推荐使用。毕竟我们又引入了容器这个外部依赖，如果网络情况不好，如何拉取 Redis 镜像也是需要解决的问题。

更好的解决办法，是我们在编写代码时，就要考虑如何写出可测试的代码，好的代码设计，能够大大降低编写测试的难度。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/redis) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn

**参考**

- Redis client for Go：https://github.com/redis/go-redis
- Redis client Mock：https://github.com/go-redis/redismock
- Testcontainers：https://github.com/testcontainers/testcontainers-go
- Testcontainers 文档：https://golang.testcontainers.org/quickstart/

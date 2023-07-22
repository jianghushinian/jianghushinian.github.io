---
title: 在 Go 中如何编写出可测试的代码
date: 2023-07-22 08:17:53
tags:
- Go
- Test
categories:
- Go
---

之前写了几篇文章，介绍在 Go 中如何编写测试代码，以及如何解决被测试代码中的外部依赖问题。但其实在编写测试代码之前，还有一个很重要的点，容易被忽略，就是什么样的代码是可测试的代码？为了更方便的编写测试，我们在编码阶段就应该要考虑到，自己写出来的代码是否能够被测试。本文就来聊一聊在 Go 中如何写出可测试的代码。

<!-- more -->

本文不讲理论，只讲我在实际开发过程中的经验和思考，使用几个实际的案例，来演示怎样从根上解决测试代码难以编写的问题。

### 使用变量来定义函数

假设我们编写了一个 `Login` 函数，用来实现用户登录，示例代码如下：

```go
func Login(u User) (string, error) {
	// ...
	token, err := GenerateToken(32)
	if err != nil {
		// ...
	}
	// ...
	return token, nil
}
```

`Login` 函数接收 `User` 信息，并在内部通过 `GenerateToken(32)` 函数生成一个 32 位长度的随机 `token` 作为认证信息，最终返回 `token`。

这个函数只编写了大体框架，具体细节没有实现，但我们可以发现，`Login` 函数内部依赖了 `GenerateToken` 函数。

`GenerateToken` 函数定义如下：

```go
func GenerateToken(n int) (string, error) {
	token := make([]byte, n)
	_, err := rand.Read(token)
	if err != nil {
		return "", err
	}
	return base64.URLEncoding.EncodeToString(token)[:n], nil
}
```

现在我们要为 `Login` 函数编写单元测试，可以写出如下测试代码：

```go
func TestLogin(t *testing.T) {
	u := User{
		ID:     1,
		Name:   "test1",
		Mobile: "13800001111",
	}
	token, err := Login(u)
	assert.NoError(t, err)
	assert.Equal(t, 32, len(token))
}
```

可以发现，在调用 `Login` 函数后，我们只能断言获得的 `token` 长度，而无法断言 `token` 具体内容，因为 `GenerateToken` 函数每次随机生成的 `token` 值是不一样的。

这看起来似乎没什么问题，但通常情况下，我们应该尽量避免测试代码中出现随机性的值。并且，有可能被测试代码较为复杂，比如我们要测试的是调用 `Login` 函数的上层函数，那么这个函数可能还会使用 `token` 去做其他的事情。此时，就会出现代码无法被测试的情况。

所以，在编写测试时，我们应该让 `GenerateToken` 函数的返回结果固定下来，但现在定义的 `GenerateToken` 函数显然无法做到这一点。

要解决这个问题，我们需要重新定义下 `GenerateToken` 函数：

```go
var GenerateToken = func(n int) (string, error) {
	token := make([]byte, n)
	_, err := rand.Read(token)
	if err != nil {
		return "", err
	}
	return base64.URLEncoding.EncodeToString(token)[:n], nil
}
```

`GenerateToken` 函数内部逻辑没变，不过换了一种定义方式。`GenerateToken` 不再是函数名，而是一个变量名，这个变量指向了一个匿名函数。

现在我们就有机会在测试 `Login` 的时候，将 `GenerateToken` 变量进行替换，实现一个只会返回固定输出的 `GenerateToken` 函数。

新版单元测试代码实现如下：

```go
func TestLogin(t *testing.T) {
	u := User{
		ID:     1,
		Name:   "test1",
		Mobile: "13800001111",
	}
	token, err := Login(u)
	assert.NoError(t, err)
	assert.Equal(t, 32, len(token))
	assert.Equal(t, "jCnuqKnsN5UAM9-LgEGS_COvJWp15RDv", token)
}

func init() {
	GenerateToken = func(n int) (string, error) {
		return "jCnuqKnsN5UAM9-LgEGS_COvJWp15RDv", nil
	}
}
```

我们利用 `init` 函数，在测试文件执行一开始就替换了 `GenerateToken` 变量的指向，新的匿名函数返回固定的 `token`。这样一来，在测试时 `Login` 函数内部调用的就是 `GenerateToken` 变量所指向的函数了，其返回值已经被固定，因此，我们可以对其进行断言操作。

### 使用依赖注入来解决外部依赖

现在我们有一个 `GenerateJWT` 函数，用来生成 [JSON Web Token](https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)，其实现如下：

```go
func GenerateJWT(issuer string, userId string, expire time.Duration, privateKey *rsa.PrivateKey) (string, error) {
	nowSec := time.Now().Unix()
	token := jwt.NewWithClaims(jwt.SigningMethodRS512, jwt.MapClaims{
		"expiresAt": nowSec + int64(expire.Seconds()),
		"issuedAt":  nowSec,
		"issuer":    issuer,
		"subject":   userId,
	})

	return token.SignedString(privateKey)
}
```

这个函数使用当前时间戳作为 `payload`，并且使用了 `RS512`，来生成 JWT。

此时，我们要为这个函数编写一个单元测试，代码如下：

```go
func TestGenerateJWT(t *testing.T) {
	key, err := jwt.ParseRSAPrivateKeyFromPEM([]byte(privateKey))
	assert.NoError(t, err)

	token, err := GenerateJWT("jianghushinian", "1234", 2*time.Hour, key)
	assert.NoError(t, err)
	assert.Equal(t, 499, len(token))
}
```

因为 `GenerateJWT` 函数生成 `token` 所使用的 `payload` 是依赖当前时间的（`time.Now().Unix()`），故每次生成的 `token` 都会不同。所以同之前的 `GenerateToken` 函数一样，我们也无法断言 `GenerateJWT` 返回的 `token` 内容，只能断言其长度。

但这是不合理的，断言 `token` 长度仅能表示这个 `token` 生成出来了，但是不保证正确。因为 JWT 有很多算法，假如在编写 `GenerateJWT` 函数时选错了算法，比如选成了 `RS256`，那么 `TestGenerateJWT` 函数就无法测试出来这个 BUG。

为了提高 `GenerateJWT` 函数的测试覆盖率，我们需要解决 `time.Now().Unix()` 依赖问题。

这次我们不再采用变量 + `init` 函数的方式，而是采用依赖注入的思想，将外部依赖当做函数的参数传递进来：

```go
func GenerateJWT(issuer string, userId string, nowFunc func() time.Time, expire time.Duration, privateKey *rsa.PrivateKey) (string, error) {
	nowSec := nowFunc().Unix()
	token := jwt.NewWithClaims(jwt.SigningMethodRS512, jwt.MapClaims{
		"expiresAt": nowSec + int64(expire.Seconds()),
		"issuedAt":  nowSec,
		"issuer":    issuer,
		"subject":   userId,
	})

	return token.SignedString(privateKey)
}
```

可以发现，所谓的依赖注入，就是当 `GenerateJWT` 函数依赖当前时间时，我们不再通过 `GenerateJWT` 函数内部直接调用 `time.Now()` 来获取，而是使用参数（`nowFunc`）的方式，将 `time.Now` 函数传递进来，当函数内部需要获取当前时间时，就调用传递进来的函数参数。

这样，我们便实现了将依赖移动到函数外部，在调用函数时，将依赖从外部注入到函数内部来使用。

现在实现的单元测试代码就可以断言生成的 `token` 是否正确了：

```go
func TestGenerateJWT(t *testing.T) {
	key, err := jwt.ParseRSAPrivateKeyFromPEM([]byte(privateKey))
	assert.NoError(t, err)

	nowFunc := func() time.Time {
		return time.Unix(1689815972, 0)
	}

	actual, err := GenerateJWT("jianghushinian", "1234", nowFunc, 2*time.Hour, key)
	assert.NoError(t, err)

	expected := "eyJhbGciOiJSUzUxMiIsInR5cCI6IkpXVCJ9.eyJleHBpcmVzQXQiOjE2ODk4MjMxNzIsImlzc3VlZEF0IjoxNjg5ODE1OTcyLCJpc3N1ZXIiOiJqaWFuZ2h1c2hpbmlhbiIsInN1YmplY3QiOiIxMjM0In0.NmCDxFaBfAPPgWQ0zVMl8ON1UQMeIVNgFCn1vtbppsunb-VrOMCdnJlguvPnNc6fMD9EkzMYM3Ux8zFnTiICDMRX23UlhAo2Zb3DorThdrBcNWHMUd26DBNI9n_oUY5B6NPqtrutvqCex9lQH0vUYOt2O5dOyZ-H9cVNY1r3fJHNkYuNWxmoZRfka5o1oSWvUw8hBJfgjANOzZ5ACIi0q5hnou5hQ8VljjFsP4zj2a2lU6w5Db8_rOA04BxilkfurdExcPeaAVCtA-Km0zNwL3gGwJB21gwyb4MRHsEf-ra-4-V7O5_JGiSOQgfkNB63RoASljRXpD6q-gakm0e0fA"
	assert.Equal(t, expected, actual)
}
```

在单元测试中，调用 `GenerateJWT` 函数时，我们可以使用一个返回固定值的 `nowFunc` 函数来作为 `time.Now` 的替代品。这样当前时间就被固定下来，因而 `GenerateJWT` 函数的返回结果也就被固定下来，就可以断言 `GenerateJWT` 函数生成的 `token` 是否正确了。

> 提示：`expected` 的值可以在[这个网站](https://jwt.io/) 生成，测试所用到的 `private.pem` 和 `public.pem` 文件我都放在了[这里](https://github.com/jianghushinian/blog-go-example/tree/main/test/testable/testdata)。

对于 `GenerateJWT` 函数，我还编写了一个 `JWT.GenerateToken` 方法版本，代码如下：

```go
type JWT struct {
	privateKey *rsa.PrivateKey
	issuer     string
	// nowFunc is used to mock time in tests
	nowFunc func() time.Time
}

func NewJWT(issuer string, privateKey *rsa.PrivateKey) *JWT {
	return &JWT{
		privateKey: privateKey,
		issuer:     issuer,
		nowFunc:    time.Now,
	}
}

func (j *JWT) GenerateToken(userId string, expire time.Duration) (string, error) {
	nowSec := j.nowFunc().Unix()
	token := jwt.NewWithClaims(jwt.SigningMethodRS512, jwt.MapClaims{
		// map 会对其进行重新排序，排序结果影响签名结果，签名结果验证网址：https://jwt.io/
		"issuer":    j.issuer,
		"issuedAt":  nowSec,
		"expiresAt": nowSec + int64(expire.Seconds()),
		"subject":   userId,
	})

	return token.SignedString(j.privateKey)
}
```

对于 `TestJWT_GenerateToken` 单元测试函数的实现，就交给你自己来完成了。

### 使用接口来解耦代码

我们有一个 `GetChangeLog` 函数可以返回项目的 ChangeLog，实现如下：

```go
var version = "dev"

type ChangeLogSpec struct {
	Version   string
	ChangeLog string
}

func GetChangeLog(f *os.File) (ChangeLogSpec, error) {
	data, err := io.ReadAll(f)
	if err != nil {
		return ChangeLogSpec{}, err
	}

	return ChangeLogSpec{
		Version:   version,
		ChangeLog: string(data),
	}, nil
}
```

`GetChangeLog` 函数接收一个文件对象 `*os.File`，使用 `io.ReadAll(f)` 从文件对象中读取全部的 ChangeLog 内容并返回。

如果要测试这个函数，我们需要在单元测试中创建一个临时文件，测试完成后还要对临时文件进行清理，实现代码如下：

```go
func TestGetChangeLog(t *testing.T) {
	expected := ChangeLogSpec{
		Version: "v0.1.1",
		ChangeLog: `
# Changelog
All notable changes to this project will be documented in this file.
`,
	}

	f, err := os.CreateTemp("", "TEST_CHANGELOG")
	assert.NoError(t, err)
	defer func() {
		_ = f.Close()
		_ = os.RemoveAll(f.Name())
	}()

	data := `
# Changelog
All notable changes to this project will be documented in this file.
`
	_, err = f.WriteString(data)
	assert.NoError(t, err)
	_, _ = f.Seek(0, 0)

	actual, err := GetChangeLog(f)
	assert.NoError(t, err)
	assert.Equal(t, expected, actual)
}
```

在测试时，为了构造一个 `*os.File` 对象，我们不得不创建一个真正的文件。好在 Go 提供了 `os.CreateTemp` 方法能够在操作系统的临时目录创建文件，方便清理工作。

其实，我们还有更好的方式来实现这个 `GetChangeLog` 函数：

```go
func GetChangeLog(reader io.Reader) (ChangeLogSpec, error) {
	data, err := io.ReadAll(reader)
	if err != nil {
		return ChangeLogSpec{}, err
	}

	return ChangeLogSpec{
		Version:   version,
		ChangeLog: string(data),
	}, nil
}
```

我对 `GetChangeLog` 函数进行了小改造，函数参数不再是一个具体的文件对象，而是一个 `io.Reader` 接口类型。

`GetChangeLog` 函数内部代码无需改变，函数和它的外部依赖，就已经通过接口完成了解耦。

现在，测试过程中我们可以使用 Fake obejct 或者 Mock object 来替换真实的 `*os.File` 对象。

使用 Fake obejct 实现测试代码如下：

```go
type fakeReader struct {
	data   string
	offset int
}

func NewFakeReader(input string) io.Reader {
	return &fakeReader{
		data:   input,
		offset: 0,
	}
}

func (r *fakeReader) Read(p []byte) (int, error) {
	if r.offset >= len(r.data) {
		return 0, io.EOF // 表示数据已读取完毕
	}

	n := copy(p, r.data[r.offset:]) // 将数据从字符串复制到 p 中
	r.offset += n

	return n, nil
}

func TestGetChangeLogByIOReader(t *testing.T) {
	expected := ChangeLogSpec{
		Version: "v0.1.1",
		ChangeLog: `
# Changelog
All notable changes to this project will be documented in this file.
`,
	}

	data := `
# Changelog
All notable changes to this project will be documented in this file.
`
	reader := NewFakeReader(data)
	actual, err := GetChangeLogByIOReader(reader)
	assert.NoError(t, err)
	assert.Equal(t, expected, actual)
}
```

这一次，我们没有直接创建一个真实的文件对象，而是提供一个实现了 `io.Reader` 接口的 `fakeReader` 对象。

在测试时，可以使用这个 `fakeReader` 来替代文件对象，而不必在操作系统中创建文件。

此外，因为使用了接口来解耦，我们还可以使用 Mock 技术来编写测试代码。

不过 `io.Reader` 是一个 Go 语言内置接口，[gomock](https://github.com/uber-go/mock) 无法直接为其生成 Mock 代码。

解决办法是，我们可以为其起一个别名：

```go
type IReader io.Reader
```

然后再为 `IReader` 接口实现 Mock 代码。

还可以对 `io.Reader` 进行一层包装：

```go
type ReaderWrapper interface {
	io.Reader
}
```

然后再为 `ReaderWrapper` 接口实现 Mock 代码。

两种方式都可行，你可以根据自己的喜好进行选择。

Mock 测试代码就交给你自己来完成了。

### 总结

如何编写测试代码，不仅仅是在业务代码实现以后，写单元测试时才要考虑的问题。而是在编写业务代码的过程中，时刻都要思考的问题。好的代码，能够大大降低编写测试的难度和周期。

在编写测试时，我们应该尽量固定所依赖对象的返回值，这就要求依赖对象的代码能够方便替换。如果依赖对象是一个函数，我们可以将其定义为一个变量，测试时将变量替换成返回固定值的临时对象。

我们也可以采用依赖注入的思想，将被测试代码内部的依赖，移动到函数参数中来，这样在测试时，可以将依赖对象进行替换。

在 Go 语言中，使用接口来对代码进行解耦，是惯用方法，同时也是解决测试依赖的突破口，使用接口，我们才有机会使用 Fake 和 Mock 测试。

此外，在我们自己编写业务代码时，如果代码实现方能够提供 Fake object，那么也能为编写测试代码的人提供便利。这一点可以参考 [K8s client-go](https://github.com/kubernetes/client-go/blob/master/kubernetes/fake) 项目，K8s 团队在实现 `client-go` 时提供了对应的 Fake object，如果我们的代码依赖了 `client-go`，那么就可以直接使用 K8s 提供的 Fake object 了，而不必自己来创建 Fake object，非常方便，值得借鉴。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/testable) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn

---
title: 使用 Faker 库生成假数据
date: 2020-01-27 18:07:28
tags:
- Python
- 命令行
categories:
- Python
---

在做 Web 开发时，经常需要构造一些测试数据，每次都手动构造数据都是非常麻烦的操作，并且效率很低。在 Python 界有一个叫 `Faker` 的库就是用来解决这个问题的。

<!-- more -->

### Faker 简单使用

可以使用 `pip` 来安装 `Faker`：

```python
pip install Faker
```

`Faker` 库使用起来非常简单，示例代码如下：

```python
>>> from faker import Faker
>>> fake = Faker()
>>> fake.name()
'Emily Jones DDS'
>>> fake.address()
'4204 Wallace Highway Suite 512\nJohnsonborough, ME 69851'
>>> fake.text()
'Mean through magazine black. Where exist deal parent.\nDifference though during too same.\nWhy result character seem car course. Seven east meeting scientist.'
```

`Faker` 提供了一个 `Faker` 类，使用前需要实例化为一个 `fake` 对象，需要造什么类型的假数据，只需要调用实例对象的方法即可。

`fake.name()` 生成一个随机人名，`fake.address()` 生成随机地址，`fake.text()` 生成一段随机文本。

可以看到以上生成的数据都为英文，实际上 `Faker` 支持多种语言，可以在 [GitHub](https://github.com/joke2k/faker#localization) 仓库上面查看所有支持的语言。如果想要生成其他语言数据，只需要在实例化 `Faker` 类时传入代表对应语言的字符串即可，比如要生成中文只需要传入 `zh_CN`：

```python
>>> fake = Faker('zh_CN')
>>> fake.name()
'刘建平'
>>> fake.address()
'广东省南京县南溪天津街A座 827319'
>>> fake.text()
'发生一定工具.之后查看数据然后专业地区.还是以后程序一些.\n数据活动发布运行威望因此.下载影响各种等级很多.\n关系可是选择责任.提供生活如何不能然.\n因此加入目前.\n虽然大小决定那么这么.开发你的研究.\n组织大家谢谢工程.运行一定等级什么.个人开发网站.'
```

### Faker Providers

`Faker` 实例对象 `fake` 能够生成多种假数据，但它本身并没有诸如 `name`、`address` 等方法。`fake` 对象实际上是一个代理对象，真正能够生成假数据的对象为 `Provider`。`Faker` 库内置了多个 `Provider` 用来生成不同类型的假数据。`fake.providers` 能够查看所有内置的 [`Provider`](https://faker.readthedocs.io/en/master/locales/zh_CN.html)：

```python
>>> fake.providers
[<faker.providers.user_agent.Provider object at 0x109f19d90>, <faker.providers.ssn.zh_CN.Provider object at 0x109f19b10>, <faker.providers.python.Provider object at 0x109e97c50>, <faker.providers.profile.Provider object at 0x109e97b90>, <faker.providers.phone_number.zh_CN.Provider object at 0x109e97b10>, <faker.providers.person.zh_CN.Provider object at 0x109f19b90>, <faker.providers.misc.en_US.Provider object at 0x109f007d0>, <faker.providers.lorem.zh_CN.Provider object at 0x109f006d0>, <faker.providers.job.zh_CN.Provider object at 0x109f00590>, <faker.providers.isbn.Provider object at 0x109f00750>, <faker.providers.internet.zh_CN.Provider object at 0x109f004d0>, <faker.providers.geo.en_US.Provider object at 0x109e97590>, <faker.providers.file.Provider object at 0x109e977d0>, <faker.providers.date_time.en_US.Provider object at 0x109e97690>, <faker.providers.currency.Provider object at 0x109e97490>, <faker.providers.credit_card.en_US.Provider object at 0x109e97c10>, <faker.providers.company.zh_CN.Provider object at 0x109e97350>, <faker.providers.color.en_US.Provider object at 0x109e97cd0>, <faker.providers.barcode.Provider object at 0x109e97bd0>, <faker.providers.bank.en_GB.Provider object at 0x109e7dc50>, <faker.providers.automotive.en_US.Provider object at 0x109e81c90>, <faker.providers.address.zh_CN.Provider object at 0x109e97c90>]
```

根据 `Provider` 名称大概就能猜到其能够生成的假数据类型，如 `faker.providers.user_agent.Provider` 用来生成随机的 `User-Agent` 字符串：

```python
>>> fake.chrome(version_from=13, version_to=63, build_from=800, build_to=899)
'Mozilla/5.0 (Linux; Android 2.2.2) AppleWebKit/535.0 (KHTML, like Gecko) Chrome/36.0.846.0 Safari/535.0'
>>> fake.firefox()
'Mozilla/5.0 (iPhone; CPU iPhone OS 5_1_1 like Mac OS X) AppleWebKit/531.2 (KHTML, like Gecko) FxiOS/17.0a5677.0 Mobile/97O054 Safari/531.2'
>>> fake.safari()
'Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10_12_6 rv:2.0; am-ET) AppleWebKit/532.30.3 (KHTML, like Gecko) Version/5.0.2 Safari/532.30.3'
>>> fake.internet_explorer()
'Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 5.01; Trident/4.0)'
>>> fake.user_agent()
'Mozilla/5.0 (compatible; MSIE 8.0; Windows NT 6.2; Trident/3.1)'
```

可以看到一些方法还支持传入参数来定义生成数据。

另外还可以增加一些扩展 `Provider` 来丰富能够生成的假数据类型，具体方法可查看[官方文档](https://faker.readthedocs.io/en/master/communityproviders.html)。

### 常用方法

以下是我个人用的比较多的方法。

```python
>>> fake.name()  # 人名
'李杨'
>>> fake.sentence()  # 句子
'数据你们学生选择.'
>>> fake.text(200)  # 段落
'类型制作业务进行非常浏览有些软件.没有男人不会这么.喜欢电话工作处理社区.\n对于重要她的不能环境.方面通过科技内容组织日期提高由于.关系专业事前可以全国.状态你们最后女人.\n网络根据而且主要不过.大小软件对于所有.我们设计如此感觉客户其中新闻.'
>>> fake.date_time_this_year(before_now=True)  # 早于当前时间
datetime.datetime(2020, 1, 3, 0, 15, 10)
>>> fake.date_time_this_year(after_now=True)  # 晚于当前时间
datetime.datetime(2020, 4, 12, 14, 6, 51)
>>> fake.email(domain='test.com')  # 邮箱
'dgu@test.com'
>>> fake.company()  # 公司
'飞海科技传媒有限公司'
>>> fake.phone_number()  # 手机号
'18980694882'
```


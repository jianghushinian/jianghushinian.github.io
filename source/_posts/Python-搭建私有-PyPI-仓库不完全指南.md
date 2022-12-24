---
title: Python 搭建私有 PyPI 仓库不完全指南
date: 2022-09-10 14:53:54
tags:
- Python
categories:
- Python
---

由于某些不可抗力因素的存在，当我们通过 `pip install xxx` 的方式安装第三方 Python 包时，经常出现速度不理想的情况，而这个时候通常解决方案是使用诸如 [`豆瓣源`](https://pypi.doubanio.com/simple/)、[`阿里源`](https://mirrors.aliyun.com/pypi/simple/) 等来进行下载 Python 包。其实我们也可以自己搭建一个私有的 PyPI 仓库来进行 Python 包管理，本文就来介绍下如何搭建一个私有的 PyPI 仓库。

<!-- more -->

### 优势

先解释下已经有了 [`豆瓣源`](https://pypi.doubanio.com/simple/)、[`阿里源`](https://mirrors.aliyun.com/pypi/simple/) 的存在，我们为什么还要自己搭建 PyPI 仓库，自建 PyPI 仓库有哪些优势：

1. 速度更快，如果自己搭建的 PyPI 仓库只在内网使用，那么理论速度将会更快。

2. 可掌控，第三方 PyPI 源一般都是定时同步 Python 官方的 PyPI 仓库，有时候会出现某些包不存在的情况，如果是我们自己搭建的私有 PyPI 仓库，则可以自己手动上传 Python 包。

3. 隐私性，有些内部库不打算开源，但需要团队之间共享，如果只用私有 `Git` 仓库来管理分发则装包时就会比较麻烦，既然是私有 PyPI 仓库，我们就可以控制只有内部人员可以使用。

### 搭建并使用

我们将使用 [`pypiserver`](https://github.com/pypiserver/pypiserver) 开源项目来搭建私有 PyPI 仓库，`pypiserver` 使用 `Bottle` 这个 Python Web 框架实现了一个轻量版的 PyPI。

`pypiserver` 搭建方式有多种，可以直接通过 `pip install pypiserver` 来安装并使用，不过我今天想要介绍的是使用 Docker 的方式来安装。

#### 启动 `pypiserver`

```bash
➜ docker run --rm -p 80:8080 pypiserver/pypiserver:latest -P . -a . --fallback-url https://pypi.douban.com/simple
```

各参数说明：

- `docker run` 表示通过 Docker 启动一个容器（进程）

- `--rm` 表示这个容器用完即删，非常适合测试阶段

- `-p 80:8080` 表示主机监听到 `80` 端口的请求将会转发给容器内部的 `8080` 端口，也就是会被 `pypiserver` 接收到

- `pypiserver/pypiserver:latest` 表示容器镜像，版本为 `latest`

- `-P . -a .` 是启动 `pypiserver` 的参数，表名不使用用户名和密码进行认证

- `--fallback-url https://pypi.douban.com/simple` 下载包时，如果在私有 PyPI 仓库中没有找到某个包，将请求转发到 `豆瓣源`

启动后访问 `http://127.0.0.1/` 将得打如下界面：

![Welcome to pypiserver](welcome_to_pypiserver.jpg)

表示已经成功搭建私有 PyPI 仓库。

#### 上传 Python 包到 `pypiserver`

以 [`python_packaging_tutorial`](https://github.com/jianghushinian/python_packaging_tutorial) 项目为例，将其打包并上传到 `pypiserver`。

项目目录结构如下：

```bash
python_packaging_tutorial
├── LICENSE
├── README.md
├── pyproject.toml
├── requirements.txt
├── src
│   └── example_package
│       ├── __init__.py
│       └── example.py
└── tests
    ├── __init__.py
    └── test_example.py
```

##### 构建项目包

构建命令如下：

```bash
# 安装构建工具
➜ python3 -m pip install --upgrade build
# 构建（打包）
➜ python3 -m build
```

现在目录结构如下：

```bash
python_packaging_tutorial
├── LICENSE
├── README.md
├── dist
│   ├── example_package-0.0.1-py3-none-any.whl
│   └── example_package-0.0.1.tar.gz
├── pyproject.toml
├── requirements.txt
├── src
│   ├── example_package
│   │   ├── __init__.py
│   │   └── example.py
│   └── example_package.egg-info
│       ├── PKG-INFO
│       ├── SOURCES.txt
│       ├── dependency_links.txt
│       ├── requires.txt
│       └── top_level.txt
└── tests
    ├── __init__.py
    └── test_example.py
```

可以看到在 `dist` 目录下生成了两个包文件 `example_package-0.0.1.tar.gz`、`example_package-0.0.1-py3-none-any.whl`。前者只对项目代码进行了打包和压缩工作，后者是编译好的二进制包。为了保证跨平台性，我们这里只会将 `example_package-0.0.1.tar.gz` 上传到 `pypiserver`。

**Tip:** 如果想在本地直接安装构建好的包，执行 `python3 -m pip install ./dist/example_package-0.0.1.tar.gz` 命令即可。

##### 上传项目包

创建 `~/.pypirc` 文件，供下面介绍的分发工具 `twine` 来使用

```
[distutils]
index-servers=private-pypi

[private-pypi]
repository = http://127.0.0.1:80
```

上传构建好的 Python 包

```bash
# Python 现在推荐使用 twine 上传分发包，所以需要先安装它
➜ python3 -m pip install --upgrade twine
# 上传 Python 包到 PyPI 仓库
# --repository private-pypi 指定私有 PyPI 仓库，从 `~/.pypirc` 中读取 URL 地址
# -u "" -p "" 分别指定用户名和密码为空
➜ python3 -m twine upload --repository private-pypi dist/example_package-0.0.1.tar.gz -u "" -p ""
```

现在访问 `http://127.0.0.1/simple/example-package/` 将看到已经上传成功的包

![Example Package](example-package.jpg)

##### 从 `pypiserver` 下载并使用包

有如下测试代码 `test_package.py`：

```python
from example_package import example

example.hello_world()

response = example.httpbin_get()
print(response.status_code)
```

从私有 PyPI 仓库安装 `example_package`

```bash
➜ pip install example-package -i http://127.0.0.1/simple/
```

**注意：** `example_package` 项目依赖了外部第三方包，在 `pyproject.toml` 可以看到如下配置：

```
dependencies = [
    "certifi==2022.6.15",
    "charset-normalizer==2.1.1",
    "idna==3.3",
    "requests==2.28.1",
    "urllib3==1.26.12",
]
```

我们搭建的私有 PyPI 仓库只上传了 `example_package` 包，并不包含以上这几个外部第三方包，当在 `pypiserver` 中找不到要下载的包时，请求会被转发到 `--fallback-url` 指定的镜像源地址。

执行测试程序

```bash
➜ python3 test_package.py
Hello World!
200
```

没有问题。

### 启用认证

#### 生成认证文件

现在虽然已经通过 `pypiserver` 搭建并使用了私有 PyPI 仓库，但还不够私有化，用现在比较流行的说法：私有了，但没完全私有。因为只要在同一内网中的其他用户就可以随意将自己的 Python 包上传到 `pypiserver` 中，这可能会导致有人随意上传和 `example_package` 同名的包，来覆盖我们自己上传的 Python 包。

现在我们需要给 `pypiserver` 增加认证功能，这样只有拥有账号密码的用户才可以上传 Python 包。

这里需要借助 `Apache htpasswd` 来生成一个认证文件供 `pypiserver` 服务使用

```bash
➜ htpasswd -c ~/.htpasswd username
New password: # 输入密码
Re-type new password: # 再次输入密码
Adding password for user username
```

使用 `htpasswd` 来生成一个用户名为 `username` 密码为 `password` 的认证文件 `~/.htpasswd`。

其文件内容如下：

```bash
➜ cat ~/.htpasswd
username:$apr1$Yob6NUQf$M8vpxJ.PpNYh0FEahMLzS0
```

#### 启动带有认证功能的 `pypiserver`

先使用 `Ctrl+C` 停掉之前使用 `docker run ...` 命令启动的 `pypiserver` 服务，还记得启动时指定的 `--rm` 参数吗，这个参数此时就会发挥作用，它会自动删除当前停掉的容器。

再使用如下命令启动新的 `pypiserver` 服务：

```bash
➜ docker run --rm -p 80:8080 -v ~/.htpasswd:/data/.htpasswd pypiserver/pypiserver:latest --fallback-url https://pypi.douban.com/simple -P .htpasswd packages
```

这次的启动命令先通过 `-v ~/.htpasswd:/data/.htpasswd` 参数将认证文件 `~/.htpasswd` 复制到容器内部，然后再通过 `-P .htpasswd` 来供 `pypiserver` 服务使用，就可以开启私有 PyPI 仓库的认证功能。最后的 `packages` 参数指明存放 Python 包的目录，不指定也没关系，它必须是容器内部已经存在的目录，默认为 `/data/packages` 目录。

现在上传 Python 包则需要指明用户名和密码才可以上传成功：

```bash
➜ python3 -m twine upload --repository private-pypi dist/example_package-0.0.1.tar.gz -u username -p password
```

否则将得到如下错误：

```bash
➜ python3 -m twine upload --repository private-pypi dist/example_package-0.0.1.tar.gz -u "" -p ""            
Uploading distributions to http://127.0.0.1:80
Uploading example_package-0.0.1.tar.gz
100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 7.0/7.0 kB • 00:00 • ?
WARNING  Error during upload. Retry with the --verbose option for more details.                                                           
ERROR    HTTPError: 401 Unauthorized from http://127.0.0.1:80/                                                                            
         Unauthorized         
```

#### 优化认证过程

你一定不想每次使用私有 PyPI 仓库上传 Python 包都输入用户名和密码，即使通过使用环境变量来保存用户名和密码的方式也会比较繁琐，不过有一种非常简单的方法可以做到避免输入用户名和密码的麻烦。

修改 `~/.pypirc` 文件：

```
[distutils]
index-servers=private-pypi

[private-pypi]
repository = http://127.0.0.1:80
username = <username>
password = <password>
```

*对应的 `<username>`、`<password>` 部分替换成我们用 `.htpasswd` 创建认证文件时指定的用户名和密码*

现在直接使用如下命令即可上传 Python 包到私有仓库：

```bash
➜ python3 -m twine upload --repository private-pypi dist/example_package-0.0.1.tar.gz
```

### 总结

本文介绍了使用 Docker 搭建私有 PyPI 仓库的基本流程，如果正式部署 `pypiserver` 项目，则在使用 `docker run ...` 命令启动的 `pypiserver` 服务时，建议使用如下命令：

```bash
➜ docker run -d --name pypiserver -p 80:8080 -v ~/packages:/data/packages -v ~/.htpasswd:/data/.htpasswd pypiserver/pypiserver:latest --fallback-url https://pypi.douban.com/simple -P .htpasswd packages
```

- 这条命令将 `--rm` 参数移除，这样当想要停止或者重启容器时容器不会被清理掉

- 新增加的 `-d` 参数让容器以后台进程的方式运行，这样即使关闭终端，容器依然在运行

- `--name pypiserver` 参数给启动容器取名为 `pypiserver`，方便维护

- `-v ~/packages:/data/packages` 参数将主机 `~/packages` 目录挂载到容器 `/data/packages` 目录，这样即使删除容器，我们上传过的 Python 包依然存在，不会丢失，重新运行新的容器时再次使用此参数挂载容器，那么新启动的 `pypiserver` 服务依然能够读取以前上传的 Python 包

如果你对 Docker 不熟悉，官方文档也有直接通过 `pip` 安装 `pypiserver` 的方式。如果需要更高级的功能可以在 [`pypiserver`](https://github.com/pypiserver/pypiserver) GitHub 官方仓库查看更详细的文档。

如果你有多个私有仓库需要配置，那么 `~/.pypirc` 文件可以这样写：

```
[distutils]
index-servers =
  local
  5.14

[local]
repository = http://127.0.0.1:8080
username = username
password = password

[5.14]
repository = http://10.0.5.14:31578
username = fake
password = fake
```

本文也顺带讲了一点构建 Python 包的知识，你也许对使用 `pyproject.toml` 文件来构建 Python 包比较陌生，现在较知名的 Python 开源项目由于历史原因，使用此方式的的确不多，不过还是推荐你了解一下。关于 Python 包构建可讲的东西其实也比较多，简单一句话概括：`setup.py` 是过去，`setup.cfg` 是现在，`pyproject.toml` 是未来。

希望这篇文档对你有所帮助。

### 参考

> https://packaging.python.org/en/latest/tutorials/packaging-projects/
> https://peps.python.org/pep-0631/
> https://peps.python.org/pep-0517/

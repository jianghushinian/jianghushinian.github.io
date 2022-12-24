---
title: 搭建 Jupyter Notebook 服务
date: 2019-09-13 21:27:16
tags:
- Python
- 工具
- 开发环境
categories:
- Python
---

工欲善其事，必先利其器。`Jupyter Notebook`  在 Python 生态中的地位想必不用我多讲，看到下图你就会明白它有多强大。

<!-- more -->

![labpreview](labpreview.png)

通常我们只在自己本地的电脑上安装 `Jupyter Notebook` 来作为开发工具，但是当我们换了台新电脑或者是使用其他人的的电脑的时候，想用 `Jupyter Notebook` 来开发程序的话，就需要在新电脑上重新装一遍开发环境。如果只是临时换一台电脑的话，就会显得比较麻烦。所以，我们可以考虑在远程服务器上搭建一套 `Jupyter Notebook` 的开发环境。这样不管我们换到哪台电脑上，都可以远程连接服务器上的 `Jupyter Notebook` 环境来进行开发。

搭建 `Jupyter Notebook` 服务的目的，就是要统一开发环境，以实现在任何地方只要你有一个能够联网的 `浏览器`，就能够进行 Python 代码的开发。

### 准备工作

要搭建 `Jupyter Notebook` 服务，首先你需要有一台云服务器。我这里使用 `Ubuntu 18.04` 来作为演示。

至于 Python 的版本，我选择的是 `Python 3.6.7`。主要是因为这是我的 `Ubuntu 18.04` 云服务器自带的 Python 版本。你也可以选择其他版本，不过我不建议使用 Python2 的版本，毕竟到 `2020-01-01` 开始官方就停止维护了。

### 安装并配置 `Jupyter`

首先需要登录到云服务器，然后安装 `Jupyter`。

```
$ pip3 install jupyter
```

安装好后，我们需要对其进行配置。在服务器当前登录用户的家目录下创建一个 `notebooks` 目录作为 `Jupyter Notebook` 启动后的工作目录。

（假设我当前登录的用户名为 `pythonic`，后文不再提示，你可以根据你自己的登录用户名来更改路径。）

```
$ mkdir /home/pythonic/notebooks
```

在 `/home/pythonic/.jupyter/` 目录下创建名为 `jupyter_notebook_config.py` 的配置文件，用来管理 `Jupyter Notebook` 的配置。我们并不需要做额外的工作，`Jupyter Notebook` 启动时会自动找到并加载这个配置文件。

```
$ sudo vim /home/pythonic/.jupyter/jupyter_notebook_config.py
```

将以下内容写入到所创建的配置文件中。

```
c.NotebookApp.port = 8888  # 默认启动端口，根据需要自行修改
c.NotebookApp.ip = '*'  # 允许任何 IP 访问
c.NotebookApp.notebook_dir = '/home/pythonic/notebooks'  # 工作目录
c.NotebookApp.open_browser = False  # 启动时不自动打开浏览器
```

写好配置文件，为了安全起见，我们还需要开启 `Jupyter Notebook` 的密码功能。这样不至于任何人都可以随意访问并使用。执行如下命令，根据提示输入两次密码即可。

```
$ jupyter notebook password
```

成功生成密码以后，`Jupyter Notebook` 会自动在 `/home/pythonic/.jupyter/` 目录下生成 `jupyter_notebook_config.json` 文件。里面存储的实际上是密码的哈希值，内容格式如下。

```
{
  "NotebookApp": {
    "password": "sha1:b82df400d1fc:d0dc30b222bf84e8974a17206b37d3ae58b071ee"
  }
}
```

此时，`Jupyter Notebook` 的基本配置就算完成。你可以在命令行下运行服务。

```
$ jupyter-notebook
```

如果顺利的话，此时打开你本地电脑的浏览器，地址栏输入远程云服务器的 `IP:8888` 就能够访问 `Jupyter Notebook` 了。

![login](login.png)

![index](index.png)

不过需要注意的是，如果你的云服务器的 `8888` 端口没有对外开放，那么本地是无法访问云服务器上的 `Jupyter Notebook` 环境的。

例如我的云服务器使用了 `UFW` 防火墙，默认 `8888` 端口是不对外开放的，可以通过命令将其开启。

```
sudo ufw allow 8888
```

你可以根据自己的实际情况来处理可能遇到的类似问题。

现在环境已经初步搭好，不过还有一个重要的问题需要处理。我们现在通过命令行来启动 `Jupyter Notebook`，当命令行关闭，`Jupyter Notebook` 也会被同时关闭。这并不是我们想要的结果，我们需要的是无论任何时候都让 `Jupyter Notebook` 一直运行下去，这样才能在任何地点、任何时间通过浏览器来访问它。

### 将 `Jupyter Notebook` 配置为系统服务

要解决上面的问题办法有很多，我这里选择将其配置为系统服务。这样只要不关机，`Jupyter Notebook` 就能一直运行下去。

这里涉及到 `Linux` 系统中 `Systemd` 的使用，它可以用来启动并管理 `守护进程` 。你并不需要对它有深入的了解，事实上，你只需要跟着我的步骤对其进行配置即可。如果你对其感兴趣，可以自行搜索相关资料进行学习。

先将上面运行的 `Jupyter Notebook` 程序通过按快捷键 `Ctrl + c` 停止掉，然后用如下命令来创建 `Jupyter Notebook` 系统服务的配置文件。

```
$ sudo vim /lib/systemd/system/jupyter.service
```

写入内容如下。

```
[Unit]
# 简短描述
Description=Juyper-NoteBook

[Service]
# 以 fork 方式从父进程创建子进程，创建后父进程会立即退出
Type=forking
# 启动当前服务的命令
ExecStart=/usr/local/bin/jupyter-notebook
# 停止当前服务的命令
ExecStop=/usr/bin/pkill jupyter-notebook
# 任何原因退出后自动重启
Restart=always
# 重启服务之前，需要等待的秒数
RestartSec=30s
# 用户
User=pythonic
# 组
Group=root

[Install]
# 该服务所在的 Target
WantedBy=multi-user.target
```

对于配置文件，为了便于理解，我已经加入了简单的注释，你需要根据自己的情况对其进行更改。

接下来，就可以通过系统服务的形式来启动 `Jupyter Notebook` 了。

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable jupyter
$ sudo systemctl start jupyter
```

依次执行以上三条命令，`Jupyter Notebook` 就会以系统服务的形式运行起来。每当云服务器重新启动，`Jupyter Notebook` 也会跟着自动启动，并且即使因为意外情况进程挂掉也会自动重启。

如果想要停止运行 `Jupyter Notebook` 服务，只需要执行以下命令。

```
$ sudo systemctl stop jupyter
```

以上，就是对搭建 `Jupyter Notebook` 服务的简单介绍。如果你还需要更高级的玩法，如 `虚拟环境` 使用、配置 `Nginx` 反向代理等，可以选择自行探索，也欢迎与我一起交流探讨。

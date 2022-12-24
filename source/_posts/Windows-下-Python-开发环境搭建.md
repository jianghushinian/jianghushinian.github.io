---
title: Windows 下 Python 开发环境搭建
date: 2020-12-01 15:04:06
tags:
- Python
- 工具
- 开发环境
categories:
- Python
---

这几天装了一个 Windows 10 虚拟机，简单记录下 Python 开发环境搭建。因为不是主力开发机，只是偶尔需要用到 Windows 环境进行代码测试，所以一切从简。

<!-- more -->


### Python

官网下载 [Python3.9](https://www.python.org/downloads/release/python-390/)

Windows10 x64 位系统选择 `Windows x86-64 executable installer` 进行下载：

![Windows x86-64 executable installer](Windows_x86-64_executable_installer.png)

双击下载下来的 `python-3.9.0-amd64.exe` 程序进行安装：

![安装Python](install_python.png)

需要注意先勾选底部的 `Add Python 3.9 to PATH`，然后再点击 `Install Now` 进行安装。如果想修改 `Python` 默认安装路径则需要点击 `Customize installation` 进行更改。

安装成功后，打开命令提示符，输入 `python` 并回车，即可进入 `Python` 交互式命令行：

![Hello World](hello_world.png)


### 使用国内镜像加速 pip

升级 `pip`：

```
λ pip install --upgrade pip
```

临时提速：

```
λ pip install pkg -i https://pypi.douban.com/simple/
```

永久提速：

```
λ pip config set global.index-url http://pypi.douban.com/simple/
Writing to C:\Users\wangjiaxing\AppData\Roaming\pip\pip.ini

λ pip config set install.trusted-host pypi.douban.com
Writing to C:\Users\wangjiaxing\AppData\Roaming\pip\pip.ini
```

*根据提示可以发现以上两条命令会生成此文件 `C:\Users\wangjiaxing\AppData\Roaming\pip\pip.ini` 内容如下：*

```
[global]
index-url = http://pypi.douban.com/simple/

[install]
trusted-host = pypi.douban.com
```

*所以如果不想使用国内镜像，需要换回默认源时，删除此文件即可。*

由于上面设置的是 `http` 源，所以需要同时设置 `trusted-host`，不然装包时会有失败提示。如果设置 `https` 源则只需要执行一条 `pip config set global.index-url https://pypi.douban.com/simple/` 命令即可。


### 虚拟环境

使用 `Python` 自带的 `venv` 创建虚拟环境

```
λ python -m venv spider  # 创建虚拟环境
λ spider\Scripts\activate.bat  # 进入虚拟环境
(spider) λ deactivate  # 退出虚拟环境
```

使用 `virtualenvwrapper` 管理虚拟环境

```
λ pip install virtualenvwrapper-win  # 安装 virtualenvwrapper
λ mkvirtualenv web  # 创建虚拟环境（创建好后会自动进入虚拟环境）
(web) λ deactivate  # 退出虚拟环境
λ workon web  # 进入虚拟环境
```

*所有虚拟环境会被创建到 `C:\Users\wangjiaxing\Envs\` 目录下。*

*安装多版本 `Python` 时，可以用 `-p` 参数指定虚拟环境需要使用的 `Python`：`mkvirtualenv -p python路径 venv` 。*


### Cmder

官网下载 [Cmder](https://cmder.net/)（选择 `Download Full`）

下载后解压到 `C:\cmder` 目录，并将此目录加入到系统环境变量。

以管理员权限打开一个终端，输入如下命令，将 `cmder` 添加到右键菜单： 

```
Cmder.exe /REGISTER ALL
```

效果如下：

![cmder](cmder.png)


### PyCharm

官网下载 [PyCharm](https://www.jetbrains.com/pycharm/download/)


### VS Code

官网下载 [Visual Studio Code](https://code.visualstudio.com/Download)


### XAMPP

官网下载 [XAMPP](https://www.apachefriends.org/index.html)

![XAMPP](XAMPP.png)

点击 `MySQL` 对应的 `Start` 按钮即可开启 `MySQL` 服务。

点击 `Shell` 开启右侧命令行，输入 `mysql -uroot` 进入 `MySQL` 命令行。
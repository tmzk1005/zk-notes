---
title: "python远程调试"
date: 2021-01-29T20:22:39+08:00
draft: false
tags: ["python"]
toc: true
---

使用vscode和ptvsd远程调试python代码

假设场景如下：

1. 本地开发机A，可以正常联网，可以使用pip安装python包，安装有Vscode的Python开发环境
2. 远程测试环境B，不能连公网，不能使用pip安装python包，一般不能随意修改运行环境的环境
3. 目录映射关系：A：/home/user/code  <--> B: /


要在A机器上远程调试B机器上的代码。依此执行下面的步骤：

## 1. 本机安装ptvsd

```bash
pip install ptvsd
```

## 2. 远程机器安装ptvsd

远程机不能联网，刚好ptvsd这个包没有其他的依赖包问题，所以可以直接把本机的包复制到B机器上。为了尽量不污染远程机器的生产环境。可以在远程机器上找一个专门的目录来做ptvsd调试相关的事

假设我们选择`/opt/pydebug`目录作为这个目录，在这个目录下建一个目录专门放python包，再写一个非常简单的脚本方便启动调试程序。

在远程机器上执行：

```bash
cd /opt
mkdir pydebug
cd pydebug
mkdir ext
touch pydebug
chmod +x pydebug
```

可执行文件`/opt/pydebug/pydebug`是方便启动的脚本，内容如下：

```bash
#!/bin/bash
export PATH=/opt/pydebug:$PATH
python -m ptvsd --host 0.0.0.0 --port 5678 --wait $*
```

(python3可能还需要设置一下PYTHONPATH)

还需要把本机用pip装的ptvsd复制到远程机器上：

(本地的ptvsd的路径可能稍有不同，scp的用户名和远程ip根据实际修改)
```
scp -r /usr/local/lib/python2.7/dist-packages/ptvsd* root@192.168.1.100:/opt/pydebug/ext
```

## 3. 远程启动等待调试

假设远程机器的代码在目录`/var/code`下面，启动脚本是`test.py`

则启动的命令就是：

```bash
cd /var/code
/opt/pydebug/pydebug test.py
```
可以把`/opt/pydebug/`加到PATH方便执行

执行后，远程机器就开始监听5678端口了，在test.py开始处暂停。等开发连接过来才开始执行，并接受开发机器的调试控制。

## 4. 开发机器新建调试配置

开发机器的配置如下：

```json
{
    "name": "Python: test",
    "type": "python",
    "request": "attach",
    "port": 5678,
    "host": "192.168.1.100",
    "pathMappings": [
        {
            "localRoot": "/home/user/dev/code/",
            "remoteRoot": "/var/code/"
        }
    ]
}
```
name取一个合适的就行，host填远程机器的ip，port保持和前面写在脚本里的一样，pathMappings配置好本地和远程的路径映射关系。

## 5. 开始调试

在vscode上点击启动调试的按钮即可。前面的配置没有配置自动在开始处停止，记得在入口处标个断点。
---
title: "Elasticsearch相关的离线文档构建"
date: 2022-07-01T15:50:46+08:00
draft: false
toc: false
tags: []
---

由于项目用到Elasticsearch，所以经常要查阅其官方文档，但是Elasticsearch的官方网站在国内访问总是
很慢，网络不好时甚至根本看不了，所以想在本地构建文档方便随时查阅。

如果只是想看Elasticsearch的离线文档，其实有现成的工具[Zeal](https://zealdocs.org/)，在Zeal下载好Elasticsearch的
文档后，就可以随时方便的在离线环境查阅了。但是Zeal中并没有全部的Elasticsearch相关的文档，比如java客户端的文档。

那怎么在本地构建Elasticsearch相关的项目的文档呢？Elastic公司开源的各个项目一般在其项目根目录下都有个`docs`目录，这个目录
就是项目的文档，一般是用`asciidoc`编写的，需要用工具把文档转换为可以部署在nginx的html文件。这个工具Elastic也提供了，地址
是[https://github.com/elastic/docs](https://github.com/elastic/docs),需要把这个项目clone到本地来使用。这个工具项
目本身有个`README.asciidoc`文件，描述了该工具如何使用。照着其步骤就可以构建本地文档了，下面以构建`elasticsearch-java`项
目的文档来简单演示。

> 下面的命令均以`/test` 目录作为公共目录来演示，实际使用命令时把`/test` 替换为实际的目录。

1. 克隆文档构建工具项目

```sh
# 当前目录: /test
git clone https://github.com/elastic/docs.git
```


2. 克隆待构建文档的项目

```sh
git clone https://github.com/elastic/elasticsearch-java.git
```

3. 使用工具构建文档

`docs`目录下有个`build_docs`可执行文件，执行命令就可以构建`elasticsearch-java`的文档。

执行`build_docs`命令时，会根据命令所在同级目录下的`Dockerfile`文件构建容器镜像（用的是containerd，不是docker）。构建镜像时
会从debian官方apt源下载安装很多软件，在国内的速度可能很慢，可以考虑执行`build_docs`命令前，先简单编辑修改下`Dockerfile`文件，配置
从国内的apt源头，因为`Dockerfile`的夫镜像是debian buster，可以配置为清华大学的源：

项目根目录下有个隐藏文件夹`.docker`, 在这个目录下的apt目录里新建`sources.list`文件，写入如下内容：

```
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-updates main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-backports main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security buster/updates main contrib non-free
```

然后修改`Dockerfile`，在` RUN install_packages apt-transport-https gnupg2 ca-certificates`这一行之后加入如下内容：

```
COPY .docker/apt/sources.list /etc/apt/
RUN apt update
```

然后开始构建：

```sh
cd elasticsearch-java
/test/docs/build_docs --doc /test/elasticsearch-java/docs/index-local.assciidoc
```

等待命令执行完成（因为容器中要下载安装比较多的软件，所以耗时稍长），生成的html文件就在新生成的`html_docs`目录，可以用来
部署在nginx中，或者直接用浏览器打开。

本地构建的文档可能打开还是不够快，这是因为仍然存在一些js，css或者字体之类的文件需要在线从cdn下载，可以下载这些文件
后，用sed命令批量替换掉html文件中的链接地址。

---
title: "生产力工具-Zeal"
date: 2022-07-06T15:08:59+08:00
draft: false
toc: false
tags: []
---

`Zeal`是一款**离线**文档查看软件，类似于MacOs上的`Dash`（事实Zeal就是山寨的Dash，它可直接使用Dash的文档格式），但是`Dash`只支持Mac系统，而`Zeal`是跨平台的，
Linux, Windows, Mac都可以运行。

下载安装地址：[Zeal官方网站](https://zealdocs.org/)

一些开源软件虽然有很好的在线文档，但是只能在线看，没有提供PDF或者HTML格式的文档下载（比如Elasticsearch的文档），在没有网络或者网络很差时就看不了。这时
通过Zeal下载已经打包好的离线文档很就方便了。

Zeal使用起来有点问题，网络不好时下载文档很慢，偶尔会下载到一半中途断线后又从头下载，这种情况下可以自己先手动下载文档，然后导入到Zeal，离线文档的下载地址可以从GitHub上的
一个开源项目得到: [https://github.com/kitty-panics/zeal-docs-downloader](https://github.com/kitty-panics/zeal-docs-downloader)

```sh
git clone https://ghproxy.com/https://github.com/kitty-panics/zeal-docs-downloader.git
```

以下载Elasticsearch的文档来演示说明：

> 下面的命令均以Linux系统来演示，如果是Windows系统，根据实际情况做一下调整。

1. 先找到地址

用`ag`命令搜索:
```sh
# 当前目录zeal-docs-downloader
ag 'elasticsearch'
```

> `ag`命令没有就用`grep`，或者安装一下`apt install silversearcher-ag`

输出大致如下：

```txt
official-london.txt
53:http://london.kapeli.com/feeds/ElasticSearch.tgz

official-sanfrancisco.txt
53:http://sanfrancisco.kapeli.com/feeds/ElasticSearch.tgz

official-newyork.txt
53:http://newyork.kapeli.com/feeds/ElasticSearch.tgz

official-tokyo.txt
53:http://tokyo.kapeli.com/feeds/ElasticSearch.tgz

official-frankfurt.txt
53:http://frankfurt.kapeli.com/feeds/ElasticSearch.tgz
```

2. 下载

从一个地理位置比较近（或者随便一个）的服务器下载文档：

```sh
wget http://tokyo.kapeli.com/feeds/ElasticSearch.tgz
```

3. 导入到Zeal

Zeal的文档在Linux中存放目录是`~/.local/share/Zeal/Zeal/docsets`

```sh
tar -zxvf ElasticSearch.tgz -C ~/.local/share/Zeal/Zeal/docsets
```

导入后要重启Zeal才会看到新加入的文档


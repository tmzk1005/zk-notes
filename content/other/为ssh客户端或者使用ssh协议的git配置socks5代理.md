---
title: "为ssh客户端或者使用ssh协议的git配置socks5代理"
date: 2022-06-30T16:47:21+08:00
draft: false
toc: false
tags: []
---

编辑文件`~/.ssh/config` (没有就新建)，加入为`github.com`这个Host的配置：
```
Host github.com
    User git
    Hostname github.com
    Port 22
    Proxycommand /usr/bin/ncat --proxy 127.0.0.1:1080 --proxy-type socks5 %h %p
```
`127.0.0.1:1080` 替换为实际的socks5代理地址

`ncat`命令可能没有， 那就需要先安装之：

```sh
sudo apt install ncat
```

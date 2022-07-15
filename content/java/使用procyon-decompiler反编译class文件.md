---
title: "使用procyon-decompiler反编译class文件"
date: 2022-07-15T15:10:33+08:00
draft: false
toc: false
tags: []
---

## 安装

```sh
sudo apt install procyon-decompiler
```

> 或者上`github`直接下载源码然后自己构建jar包：[https://github.com/mstrobel/procyon/](https://github.com/mstrobel/procyon/)

## 使用

用`dpkg -L`查看安装文件：

```sh
dpkg -L procyon-decompiler
```

输出大致如下：

```txt
/.
/usr
/usr/bin
/usr/bin/procyon
/usr/share
/usr/share/bash-completion
/usr/share/bash-completion/completions
/usr/share/bash-completion/completions/procyon
/usr/share/doc
/usr/share/doc/procyon-decompiler
/usr/share/doc/procyon-decompiler/copyright
/usr/share/java
/usr/share/java/procyon-decompiler-0.5.36.jar
/usr/share/man
/usr/share/man/man1
/usr/share/man/man1/procyon.1.gz
/usr/share/doc/procyon-decompiler/changelog.Debian.gz
/usr/share/java/procyon-decompiler.jar
```

可以看到多了一个可执行文件`/usr/bin/procyon`

使用方式如下：

```sh
procyon A.class
```

或者直接用`java -jar`

```sh
java -jar /usr/share/java/procyon-decompiler-0.5.36.jar A.class
```

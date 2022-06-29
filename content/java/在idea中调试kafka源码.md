---
title: "在idea中调试kafka源码"
date: 2021-03-28T10:00:00+08:00
draft: false
toc: true
---

kafka的源码并不像一般简单的项目从github上clone下来后用idea打开就可以调试，想要在idea中运行和打断点调试kafka的源码还小费了一番周折，特此记录一下。

# clone代码

首先当然是要从github上clone源码，不过国内从github上clone代码网速很慢，很容易clone一般就断开了。一个加速的方案是使用代理，晚上可以搜到github的代码网站，比如 https://ghproxy.com/

把这个地址加到kafka的clone地址前面：

```bash
git clone https://ghproxy.com/https://github.com/apache/kafka.git
```

# 给idea安装scala插件

这个没有什么好说的，点开idea的设置页面安装scala插件即可。

# 切换分支

最好切换到一个已经发布的稳定分支来调试和学习，而不是在trunk分支上调试。比如用2.8分支：

```bash
git checkout -b 2.8 remotes/origin/2.8
```

# 生成idea项目文件

## 修改下maven仓库地址

为了加快下载jar包的速度，最好设置下maven仓库地址：

将build.gradle中的：

```groovy
repositories {
    mavenCentral()
    jcenter()
    maven {
        url "https://plugins.gradle.org/m2/"
    }
}
```

修改为：

```groovy
repositories {
    mavenCentral()
    jcenter()
    maven {
        url "http://maven.aliyun.com/nexus/content/groups/public/"
        url "https://plugins.gradle.org/m2/"
    }
}
```

## 执行gradle任务生成idea项目文件。

在使用idea打开kafka源码目录之前，先用kafka的gradle任务自动生成idea项目文件：

```bash
./gradlew idea
```

第一次执行这条命令，gradle wrapper会去下载gradle-6.8.1-all.zip，如果下载比较慢，可以先手动下载然后放在HOME目录的`.gradle/wrapper/dists/gradle-6.8.1-all/923to48kq3drqywuppfjkcokx`目录下。

手动下载地址：https://code.aliyun.com/kar/gradle-all-zip-6.8.x/raw/master/gradle-6.8.1-all.zip

# 生成自动生成的文件

kafka有些代码是自动生成的，因此要执行gradle任务自动生成这些代码才能启动kafka成功：

```bash
./gradlew processMessages processTestMessages
```

这些自动生成的代码都放在各个子项目的`src/generated`目录下，因此代码生成后要把这个目录设置为源代码目录：右键目录 -> Mark Directory as -> Source Root


# 设置idea

此时打开类'kafka.Kafka'，执行其main方法，idea会先构建项目，但是会失败，需要修改2项配置：

1. Preferences -> Build, Execution, Deployment -> Compiler -> Scala Compiler -> Additional compiler options

设置属性：`-language:postfixOps`

2. Preferences -> Build, Execution, Deployment -> Compiler -> Scala Compiler -> Scala Compile Server -> VM options

将`-Xss1m`修改为`-Xss32m`，这项修改是因为构建步骤需要用到比较大的栈空间。


# 最后

最后可以打断点debug源码了。

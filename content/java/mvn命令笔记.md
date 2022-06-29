---
title: "mvn命令笔记"
date: 2020-12-19T11:22:39+08:00
draft: false
tags: ["java", "maven"]
toc: true
---

一般平时我们使用mvn命令，也就简单的执行个`mvn clean`和`mvn package`,以及新建项目时执行`mvn archetype:generate`

这是远远不够的，这里讲些一些有用的命令。

## mvn -h

执行`mvn -h`可以看到mvn命令的形式如下：

```
mvn [options] [<goal(s)>] [<phase(s)>]
```

mvn支持的options执行`在mvn -h`可以看到，这里主要学习下`[<goal(s)>]`和`[<phase(s)>]`

## phase

phase字面意思是“阶段”，这里表示这个参数是mvn的生命周期，maven有3条生命周期线：

- clean线

```txt
pre-clean --> clean --> post-clean
```

- build线

```txt
validate --> initialize --> generate-sources --> process-sources
--> generate-resources --> process-resources --> compile --> process-classes
--> generate-test-sources --> process-test-sources --> test-compile --> process-test-classes
--> test --> prepare-package --> package --> pre-integration-test
--> integration-test --> post-integration-test --> verify --> install --> deploy
```

- site线

```
pre-site --> site --> post-site --> site-deploy
```

当mvn跟一个`phase参数`，也就是是一个生命周期时，mvn会从该周期所在的周期线从开始依次执行该周期。例如执行`mvn package`,maven就会依次validate，initialize，... , 一直到package

那么执行到某个生命周期时具体干什么呢？每个周期的背后都默认有绑定一个插件的目标，其实执行某个周期就是执行其背后的对应的插件的目标。比如当执行`mvn package`是走到`compile`这个周期时，它绑定的是compiler插件的compile目标。

假设我们要执行`compile`这个周期背后的具体的插件目标，但又不要沿着生命线从头开始，我们可以使员工命令`mvn compiler:compile`

## goal

前面的命令`mvn compiler:compile`其实就是`mvn [<goal(s)>]`的形式。

`[<goal(s)>]`由2部分组成：插件的名字，和该插件的某一目标。（一个插件可以有多个目标）

那么有那些插件呢？每个插件又有那些目标呢？

插件可以有无数个，插件的目标也是不确定的。因为我们可以自己写代码实现我们想要的插件，定义自己的目标。

那么maven已经提供的插件有那些呢，它们各自有什么目标呢？

有下面这些（我们把它们分为3类）：

- 核心插件

```txt
核心插件一般就是上面的的各个生命周期背后的插件，maven使用它们来完成java代码的编译，部署等。

clean
compiler
deploy
failsafe
install
resources
site
surefire
verifier
```

- 打包插件

```txt
打包插件用于打各种包

ear
ejb
jar
rar
war
app-client/acr
shade
source
jlink
jmod
```

- 报告插件

```txt
报告插件用于生成报告，如javadoc，以及代码风格检查等。
changelog
changes
checkstyle
doap
docck
javadoc
jdeps
jxr
linkcheck
pmd
project-info-reports
surefire-report
```

- 工具插件

```txt
工具插件，显然提供各种有用的工具功能，例如显示super-pom,显示依赖树

antrun
archetype
assembly
dependency
enforcer
gpg
help
invoker
jarsigner
jdeprscan
patch
pdf
plugin
release
remote-resources
scm
scm-publish
stage
toolchains
```

此外还有其他第三方提供的，比如`exec`


### 获取插件的帮助

**这么多的插件，每个插件又有各自的目标，不太可能完全记得住。其实maven的插件设计的很好，都有很好的帮助文档，一般每个插件都有一个名为"help"的目标，用于显示该插件的帮助文档。**

如果一个插件，居然没有“help”目标，那么它就不是一个好用的，合格的maven插件。

例如：我们要看compiler和help这2个插件的帮助：

```bash
mvn compiler:help
mvn help:help
```

如果觉得文档还不够详细的话，一般还都可以加一个参数`-Ddetail=true`以便显示更加详细的帮助。


```bash
mvn compiler:help -Ddetail=true
mvn help:help -Ddetail=true
```


## 有用的命令举例


- 显示super-pom，也就是effective-pom

```bash
# help插件的effective-pom目标，还可以设置参数output写文件
mvn help:effective-pom -Doutput=epom.xml
```

- 显示依赖树

```bash
# dependency插件的tree目标
mvn dependency:tree
```

- 执行代码

```bash
# exec插件的java目标，需要设置参数exec.mainClass指定主类
mvn exec:java -Dexec.mainClass=com.test.Main
```

- 安装文件

```bash
mvn install:install-file -Dfile=xxxxx.jar -DgroupId=xxx.xxx.xxx -DartifactId=xxxxx -Dversion=1.0.0 -Dpackaging=jar
```

- 不依赖于一个pom.xml文件来下载jar包

使用dependency插件的get目标:

```bash
# 下载jar, 最后的“:jar”可以省略
mvn dependency:get -Dartifact=ggg:aaa:1.0.0:jar
# 下载 javadoc
mvn dependency:get -Dartifact=ggg:aaa:1.0.0:jar:javadoc
# 下载源码
mvn dependency:get -Dartifact=ggg:aaa:1.0.0:jar:sources
```

---
title: "获取shell脚本绝对路径"
date: 2020-12-20T11:22:39+08:00
draft: false
tags: ["shell"]
toc: false
---

执行一个shell脚本时，如何获取脚本所在的目录的绝对路径呢，需要考虑脚本自身被软连接的情况

下面的代码是摘自tomcat的启动脚本“catalina.sh”中的一段shell代码：

```bash
PRG="$0"

while [ -h "$PRG" ]; do
  ls=`ls -ld "$PRG"`
  link=`expr "$ls" : '.*-> \(.*\)$'`
  if expr "$link" : '/.*' > /dev/null; then
    PRG="$link"
  else
    PRG=`dirname "$PRG"`/"$link"
  fi
done
```

整理一下，解决shell_checker报错问题：

```bash
PRG="$0"

while [[ -h $PRG ]] ; do
    ls=$(ls -ld "$PRG")
    link=$(expr "$ls" : '.*-> \(.*\)$')
    if expr "$link" : '/.*' > /dev/null; then
        PRG=$link
    else
        PRG=$(dirname "$PRG")/$link
    fi
done

WORK_HOME=$(dirname "$PRG")
WORK_HOME=$(cd "$WORK_HOME" && pwd)
```

其中的一行代码`link=$(expr "$ls" : '.*-> \(.*\)$')`通过expr命令获取到了软连接文件链接到的文件的文件名，这个也可以通过解析`file`命令的输出，或者用一个``readlink`命令来实现


# cd -P

更加简单的方法：用cd命令的`-P`选项：

```bash
BINDIR=$(dirname "$0")
export XXX_HOME=`cd -P $BINDIR/..;pwd`
```

如果是在一个软连接的目录里，`cd -P`直接到软连接目录对应的物理目录去。

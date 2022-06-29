---
title: "自己动手编译和调试openjdk"
date: 2021-03-20T10:00:00+08:00
draft: false
tags: ["hotspot", "java"]
toc: true
---

# 下载源代码

第一步当然是要下载源代码，从`gitee.com`(码云)clone要比从github快一点: 地址是`git@gitee.com:mirrors/openjdk.git`

```bash
git clone git@gitee.com:mirrors/openjdk.git
```

下载源码代码后，最好checkout一个稳定的分支，或者有的参考资料使用的分支，比如`jdk_12_31`

```bash
git checkout -b jdk_12_31 origin/jdk_12_31
```

# configure

在用make编译之前需要先执行`configure`，jdk6到jdk7的时代，执行`configure`往往会报很多依赖错误，需要一个个解决，很麻烦，不过现在的jdk都17了，执行`configure`报的错少了很多，即是报错也一般会在出错日志中给出错误原因和建议解决方法，按照建议解决即可，一般就是缺少了依赖，用apt，yum，或者brew安装依赖即可。

有个要注意的地方是**disable-warning-as-errors**这个配置项目，有的版本是默认开启的，有的默认是关闭的，如果是关闭的需要开启，要不然一些warning的问题就会导致make失败。

手动编译openjdk肯定是为了学习源码，最好带上调试信息：

```bash
bash configure --enable-ccache --with-debug-level=slowdebug --disable-warnings-as-errors
```

# make

执行make编译即可：

```bash
make images
```

> `make images`即可，`make all`不是很必要，还慢，而且可能有更多问题要解决。

# debug调试

用vscode安装cpp插件即可调试编译好的hotsplot了。

```json
{
    // 使用 IntelliSense 了解相关属性。
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "hotspot-debug",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/linux-x86_64-server-slowdebug/jdk/bin/java",
            "args": [
                "-cp",
                "/home/vagrant/tmp",
                "Hello"
            ],
            "cwd": "${workspaceFolder}",
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "text": "handle SIGSEGV nostop noprint"
                }
            ]
        }
    ]
}
```

上面的配置中是调试执行`java -cp /home/vagrant/tmp Hello`，要先写个简单的Hello.java类编译为class文件放在相应的类路径目录。

接下来就可以打断点调试代码了，c++的main函数所在文件是`src/java.base/share/native/launcher/main.c`，不过不知道为什么在这个main.c文件中打断点没有用，断不下来（没有生成这个文件的符号），不过可以看到在main函数最后调用了`JLI_Launch`这个函数，它所在的文件是`src/java.base/share/native/libjli/java.c`，在这个文件中的`JLI_Launch`的开始处打断点就可以开始调试jvm源码了。




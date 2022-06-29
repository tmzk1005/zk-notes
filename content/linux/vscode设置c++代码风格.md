---
title: "vscode设置c++代码风格"
date: 2021-03-10T11:22:39+08:00
draft: false
---

使用vscode开发c++项目时如何设置符合自己喜好的代码风格呢？设置项是：`C_Cpp.clang_format_fallbackStyle`，可以设置的风格有`LLVM`，`Google`，`Visual Studio`等等，比如设置为`LLVM`：

```ini
"C_Cpp.clang_format_fallbackStyle": "LLVM"
```

假设默认提供的几种风格都没有完全符合自己喜好的怎么办呢？我们可以在某一种风格的基础上加上自定义设置，比如在`LLVM`的基础上，把对齐的空格数由2个修改为4个，并设置每行的最大长度为150个字符：

```ini
"C_Cpp.clang_format_fallbackStyle": "{ BasedOnStyle: LLVM, IndentWidth: 4, ColumnLimit: 150}"
```

可以自定义的设置项目有很多，如何知道自己想要改变的设置项的名字呢？实际上vscode背后使用的是一个叫`clang-format`的程序在格式化c++代码，可以在这个软件的官方上找到答案：<https://clang.llvm.org/docs/ClangFormat.html>

另外，还可以使用`clang-format`命令将某种风格的设置导出到文件（yml格式）：

```bash
clang-format -dump-config -style=llvm > llvm.yml
```

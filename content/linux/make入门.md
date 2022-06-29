---
title: "make入门"
date: 2020-12-20T11:22:39+08:00
draft: false
tags: ["Makefile"]
toc: true
---


代码变成可执行文件，这个过程叫做编译；大型项目里，现编译这个，还是先编译那个，即编译的顺序安排，叫做构建（build）

Make是c语言项目常用的构建工具，但是Make其实可以构建任何项目。

## 0x00 Make的概念

Make的字面意思是“制作”，在项目构建里，制作，一般就是制作出某个文件，比如我们要制作出a.txt，就执行下面的命令：

```sh
make a.txt
```

但是，如果你真的输入这条命令，a.txt并不会生成，因为Make本身并不知道该如何生成a.txt，需要我们告诉它。怎么告诉呢？通过一个叫Makefile的文件。生成a.txt的Makefile文件内容应该如下：

```makefile
a.txt:
	touch a.txt
```

**touch前面是一个tab**

在Makefile所在的目录执行`make a.txt`，则该目录下就会生成一个a.txt文件，其实在背后就是执行了shell命令`touch a.txt`而已。

## 0x01 Makefile文件的格式

前面的生成a.txt的Makefile文件包含的内容，我们把它称作为一条构建规则（fule）

### 1 概述

Makefile文件由一系列的规则构成，每条规则的形式如下：

```txt
<target>: <prerequisites>
[tab]<commands>
```

第一行冒号前面的部分，叫做“目标”（target），冒号后面的部分，叫做“前置条件”（prerequisites）；第二行必须由一个tab开始，后面跟着命令。目标是必须的，不能省略；前置条件和命令者2者至少必须存在一个，或者都存在，不能都没有。当前置条件没有时，说明该目标没有其他依赖；当命令没有时，一般用于为多个其他的目标起一个聚合目标。

### 2 目标

一个目标（target）就构成一条规则。目标通常是文件名，指明Make命令所要构建的对象，比如上文的 a.txt 。目标可以是一个文件名，也可以是多个文件名，之间用空格分隔。其实目标后面的命令并不是一定要生成目标文件，比如，前面的Makefile改成如下：

```makefile
a.txt
	echo "a.txt"
```

此时执行`make a.txt`就不会生成a.txt文件了，而仅仅是在控制台输出了一句话“a.txt”。这种情况，可以把目标理解成为一系列命令而起的一个操作的名字，我们把它叫做“伪目标”（phony target）。

但是，如果在当前目录刚好有个文件名为a.txt，跟我们的伪目标名字一样，那么该伪目标后面的命令并不会执行。为了避免这种情况，我们需要明确的声明a.txt是伪目标，写法如下：

```makefile
.PHONY: a.txt
a.txt:
	echo "a.txt"
```

这样，无论是否存在名为a.txt的文件，当执行`make a.txt`时，echo命令都会执行。

另外，当执行make命令时没有指定目标，则默认会执行Makefile文件中的第一个目标。

### 3 前置条件

前置条件，其实就是目标所依赖的其他目标。举个例子，如下：

```makefile
a:
	echo "aaa" > a.txt

b:
	echo "bbb" > b.txt

c: a b
	cat a.txt b.txt > c.txt
```

目标a生成文件a.txt，目前b生成文件b.txt，目标c生成文件c.txt，从目标c的命令可以看到，生成文件c.txt先要有文件a.txt和b.txt，所以目标c依赖于目标a和b，也就是目标a和b是目标c的前置条件。

### 4 命令

命令（commands）表示如何更新目标文件，由一行或多行的Shell命令组成。它是构建"目标"的具体指令，它的运行结果通常就是生成目标文件。

每行命令之前必须有一个tab键。如果想用其他键，可以用内置变量.RECIPEPREFIX声明。

```makefile
.RECIPEPREFIX = >
all:
> echo Hello, world
```

上面代码用.RECIPEPREFIX指定，大于号（>）替代tab键。所以，每一行命令的起首变成了大于号，而不是tab键。

需要注意的是，每行命令在一个单独的shell中执行。这些Shell之间没有继承关系。

```makefile
var-lost:
	export foo=bar
	echo "foo=[$$foo]"
```

上面代码执行后（`make var-lost`），取不到foo的值。因为两行命令在两个不同的进程执行。一个解决办法是将两行命令写在一行，中间用分号分隔:

```makefile
var-kept:
	export foo=bar; echo "foo=[$$foo]"
```

另一个解决办法是在换行符前加反斜杠转义:

```makefile
var-kept:
	export foo=bar; \
	echo "foo=[$$foo]"
```

最后一个方法是加上`.ONESHELL:`命令

```bash
.ONESHELL:
var-kept:
	export foo=bar;
	echo "foo=[$$foo]"
```

## 0x02 Makefile文件的语法

### 1 注释

井号（#）在Makefile中表示注释。

```makefile
# 这是注释
result.txt: source.txt
	# 这是注释
	cp source.txt result.txt # 这也是注释
```

### 2 回声

正常情况下，make会打印每条命令，然后再执行，这就叫做回声（echoing）。

```makefile
test:
	echo "test"
```

执行上面的规则，会得到下面的结果:

```txt
zk@zk-mbp:~/zk-lab/make » ls
Makefile
zk@zk-mbp:~/zk-lab/make » cat Makefile
test:
	echo "test"

zk@zk-mbp:~/zk-lab/make » make test
echo "test"
test
zk@zk-mbp:~/zk-lab/make »
```

`make test`的输出是：

```txt
echo "test"
test
```

即把命令本身打印出来，也把命令执行结果打印出来。

在命令的前面加上@，就可以关闭回声。

```makefile
test:
	@echo "test"
```

再次执行：

```txt
zk@zk-mbp:~/zk-lab/make » ls
Makefile
zk@zk-mbp:~/zk-lab/make » cat Makefile
test:
	@echo "test"

zk@zk-mbp:~/zk-lab/make » make test
test
zk@zk-mbp:~/zk-lab/make »
```

`make test`的输出是：

```txt
test
```

只有命令的输出，没有命令本身。

是否用开启或者关闭回声，根据情况选择。

### 3 通配符

通配符（wildcard）用来指定一组符合条件的文件名。Makefile 的通配符与 Bash 一致，主要有星号（*）、问号（？）和 [...] 。比如， *.o 表示所有后缀名为o的文件。

```makefile
clean:
	rm -f *.o
```

### 4 模式匹配

Make命令允许对文件名，进行类似正则运算的匹配，主要用到的匹配符是%。比如，假定当前目录下有 f1.c 和 f2.c 两个源码文件，需要将它们编译为对应的对象文件。

```makefile
%.o: %.c
```

等同于：

```makefile
f1.o: f1.c
f2.o: f2.c
```

使用匹配符%，可以将大量同类型的文件，只用一条规则就完成构建。

### 5 变量和赋值

Makefile 允许使用等号自定义变量

``` makefile
txt = Hello World
test:
	@echo $(txt)
```

上面代码中，变量 txt 等于 Hello World。调用时，变量需要放在 $( ) 之中。

调用Shell变量，需要在美元符号前，再加一个美元符号，这是因为Make命令会对美元符号转义。

```makefile
test:
	@echo $$HOME
```

有时，变量的值可能指向另一个变量:

```makefile
v1 = $(v2)
```

上面代码中，变量 v1 的值是另一个变量 v2。这时会产生一个问题，v1 的值到底在定义时扩展（静态扩展），还是在运行时扩展（动态扩展）？如果 v2 的值是动态的，这两种扩展方式的结果可能会差异很大。

为了解决类似问题，Makefile一共提供了四个赋值运算符 （=、:=、？=、+=）

```makefile
VARIABLE = value
# 在执行时扩展，允许递归扩展。

VARIABLE := value
# 在定义时扩展。

VARIABLE ?= value
# 只有在该变量为空时才设置值。

VARIABLE += value
# 将值追加到变量的尾端。
```

### 6 内置变量

Make命令提供一系列内置变量，比如，$(CC) 指向当前使用的编译器，$(MAKE) 指向当前使用的Make工具。这主要是为了跨平台的兼容性。

```makefile
output:
	$(CC) -o output input.c
```

当然，可以自己定义CC覆盖默认的。

### 7 自动变量

Make命令还提供一些自动变量，它们的值与当前规则有关。主要有以下几个。

#### 1 $@

`$@`指代当前目标，就是Make命令当前构建的那个目标，比如`make foo`，那么`$@`就指代foo。

```makefile
a.txt b.txt:
	touch $@
```

就等同于：

```makefile
a.txt:
	touch a.txt
b.txt:
	touch b.txt
```

#### 2 $<

`$<`指代第一个前置条件。比如，规则为 t: p1 p2，那么`$<`就指代p1

```makefile
a.txt: b.txt c.txt
	cp $< $@
```

等同于：

```makefile
a.txt: b.txt c.txt
	cp b.txt a.txt
```

#### 3 $?

`$?`指代比目标更新的所有前置条件，之间以空格分隔。比如，规则为 t: p1 p2，其中 p2 的时间戳比 t 新，`$?`就指代p2。

#### 4 $^

`$^`指代所有前置条件，之间以空格分隔。比如，规则为 t: p1 p2，那么`$^` 就指代 p1 p2 。

#### 5 $*

`$*`指代匹配符 % 匹配的部分， 比如% 匹配 f1.txt 中的f1 ，`$*`就表示 f1。

#### 6 `$(@D)` `$(@F)`

`$(@D)` 和`$(@F)`分别指向`$@`的目录名和文件名

#### 7 `$(<D)` `$(<F)`

 `$(<D)`和`$(<F)`分别指向`$<`的目录名和文件名

上面的都是比较常用的自动变量，还有其他的参见官方手册

### 8 判断和循环

Makefile使用类似Bash的语法完成判断和循环

例如：

```makefile
ifeq ($(CC),gcc)
  libs=$(libs_for_gcc)
else
  libs=$(normal_libs)
endif
```

上面代码判断当前编译器是否 gcc ，然后指定不同的库文件。

### 9 函数

Makefile 还可以使用函数，格式如下：

```bash
$(function arguments)
# 或者
${function arguments}
```

Makefile提供了许多内置函数，可供调用。下面是几个常用的内置函数。

#### 1 shell

shell 函数用来执行 shell 命令

```makefile
srcfiles := $(shell echo src/{00..99}.txt)
```

#### 2 wildcard

wildcard 函数用来在 Makefile 中，替换 Bash 的通配符。

```makefile
srcfiles := $(wildcard src/*.txt)
```

#### 3 subst

subst 函数用来文本替换，格式如下:

```makefile
$(subst from,to,text)
```

下面的例子将字符串"feet on the street"替换成"fEEt on the strEEt"。

```makefile
$(subst ee,EE,feet on the street)
```

#### 4 patsubst

patsubst 函数用于模式匹配的替换，格式如下:

```makefile
$(patsubst pattern,replacement,text)
```

下面的例子将文件名"x.c.c bar.c"，替换成"x.c.o bar.o"。

```makefile
$(patsubst %.c,%.o,x.c.c bar.c)
```

#### 5 替换后缀名

替换后缀名函数的写法是：变量名 + 冒号 + 后缀名替换规则。它实际上patsubst函数的一种简写形式。

```makefile
min: $(OUTPUT:.js=.min.js)
```

上面代码的意思是，将变量OUTPUT中的后缀名 .js 全部替换成 .min.js 。

后缀可以为空：

```makefile
min: $(OUTPUT:=.min.js)
```

上面代码的意思是，将变量OUTPUT中的每一项都加上后缀 .min.js

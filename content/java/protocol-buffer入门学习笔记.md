---
title: "protocol-buffer入门学习笔记"
date: 2021-01-30T16:22:39+08:00
draft: false
tags: ["protocol-buffer"]
toc: true
---

## protocol buffer是什么

protocol buffer（以下称为protobuf）是Google开源的一种语言无关的二进制数据表示格式。可以参考XML和json来理解，XML和json都可以表示有结构的数据，而且是语言无关的，不过它们是基于文本的，而protobuf是二进制格式，一般效率要比XML和json快不少。protobuf可以用于数据序列化，可以用于RPC。

既然是语言无关的，很多语言都可以使用protobuf。再使用protobuf时，我们需要定义一个`.proto`文件，例如：

person.proto
```
message Person {
    string name = 1;
}
```

接着，我们用一个命令行程序`protoc`, 以`person.proto`文件为参数，再加上其他的一些控制参数，可以自动生成Java代码，得到一个Java类，我们就可以使用这个类来完成数据的序列化和反序列话。

从使用方法上来讲，protobuf要比xml和json复杂些，学习成本自然也高些。我们首先要用一种和具体语言无关的数据结构表示语法来编写`.proto`文件，接着我们还要借助于一个外部工具程序`protoc`,将`.proto`文件转化为具体语言的代码，比如java代码，python代码，C代码。这个自动生成的代码为我们提供了相应的API来完成数据的序列化和反序列化。

## proto文件语法

就像python有2和3这两个版本一样，protobuf也有2和3个这两个版本。它们是有一些不一样的地方的，版本3对版本2做了一些改动。

详细的语法说明可以参见:

[protobuf-language-specification](https://developers.google.cn/protocol-buffers/docs/reference/proto3-spec?hl=en)

[protobuf-language-guide-3](https://developers.google.cn/protocol-buffers/docs/proto3)

### 语法版本设置
版本3的proto文件需要在文件的开始的第一个非空非注释的行标明语法版本：

```
syntax = "proto3";
```

### message

protobuf中定义基本的数据结构使用"message"，个人认为当作java中的Bean来理解好了。

例如，我们要表示一个Person结构，包含名字和年龄2个属性:

```
message Person {
    string name = 1;
    int32 age = 2;
}
```

定义一个结构中的数据项，要依次指定数据的：
1. 值类型
   - 简单来说就是就是是否是数组类型，非数组类型为“singular”，这是默认的，可以省略。数组类型是“repeated”
2. 字段类型
   - 就是编程语言中的基本类型，如字符串，整型等。常用的就是：string， int32，bool，详细的全部类型参见前面的链接。字段类型可以是其他的“message”
3. 在序列化后的二进制结构中的索引号
   - 就是上面的=号后面的数字，从1开始，依次递增。定义好了后这个不能轻易改变，否则就不兼容了。


## protoc的安装

- mac安装： `brew install protobuf`
- linux安装：`apt install protobuf-compiler`

安装好`protoc`后执行`protoc --help`就可以看到其使用说明了，还是比较好懂的。

## 使用protobuf进行序列化和反序列化


### 先写一个proto文件

```
syntax = "proto3";

option java_package = "com.example.proto";
option java_outer_classname = "PersonProto";

message Person {
    string name = 1;
    int32 age = 2;
}
```

proto文件可以使用option“指令”设置一些配置项目，这里配置了java_package和java_outer_classname

java_package是使用protoc命令生成的Java类的包名，protoc会为我们自动建立包相应的路径，而java_outer_classname是生成的Java类名，`message Person`也会生成一个类名为“Person”的类，Person类将会是我们这只的这个类PersonProto的静态子类。Person类的全名将会为`com.example.proto.PersonProto.Person`

### 执行protoc命令

在proto文件所在的目录执行：
```
protoc --java_out=. person.proto
```

`--java_out=.`告诉protoc我们要生成的是java代码，且代码放在当前目录。

执行完后，当前目录的结构为：

```
.
├── com
│   └── example
│       └── proto
│           └── PersonProto.java
└── person.proto
```

`PersonProto.java`就是protoc自动生成的代码。使用这个类提供的API就可以完成Person的序列化和反序列化。

### 序列化

写一个测试的类，把一个Person序列化为二进制数据后写到文件：

ProtoWriter.java
```java
import java.io.OutputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

import com.example.proto.PersonProto;

public class ProtoWriter {

    public static void main(String[] args) throws Exception {
        PersonProto.Person person = PersonProto.Person.newBuilder()
                .setName("Tom")
                .setAge(18)
                .build();

        System.out.println("name = " + person.getName());
        System.out.println("age = " + person.getAge());
        System.out.println("serialized size = " + person.getSerializedSize());

        Path path = Paths.get("person.data");
        OutputStream outputStream = Files.newOutputStream(path);
        person.writeTo(outputStream);
        outputStream.close();
    }

}

```

编译这个类需要用mvn去下载好protobuf的java库：

```
<!-- https://mvnrepository.com/artifact/com.google.protobuf/protobuf-java -->
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.11.4</version>
</dependency>
```

运行是也需要把这个jar加入到类路径。

这个类运行后将会输出：

```
name = Tom
age = 18
serialized size = 7
```

序列化后的数据我们写在了文件“person.data，序列化后的数据只有7个字节的大小。可以看看该文件的大小是否为7


### 反序列化

我们再写一个测试类读取刚才写入的文件“person.data”,完成反序列化，之后同样打印属性，看是否一致。

```java
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

import com.example.proto.PersonProto;

public class ProtoReader {

    public static void main(String[] args) throws Exception {
        Path path = Paths.get("person.data");
        InputStream inputStream = Files.newInputStream(path);

        PersonProto.Person person = PersonProto.Person.parseFrom(inputStream);
        inputStream.close();

        System.out.println("name = " + person.getName());
        System.out.println("age = " + person.getAge());
        System.out.println("serialized size = " + person.getSerializedSize());

    }
}

```

编译后执行这个类，可以看到也输出的是：

```
name = Tom
age = 18
serialized size = 7
```

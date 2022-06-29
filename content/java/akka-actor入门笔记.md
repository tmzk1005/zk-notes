---
title: "akka-actor入门笔记"
date: 2021-01-30T20:22:39+08:00
draft: false
tags: ["java", "akka", "actor"]
toc: true
---

# actor是什么

actor是一种并发模型，akka是使用scala语言实现这种并发模型的一个库。在akka中，消息在不同的actor之间传递，以此来驱动任务的执行，这和一般的方法调用的方式有明显的区别。


每个actor都有自己的“地址”，这个地址可以用来唯一的标示一个actor实例，每个actor都有一个“邮箱”，希望与该actor通信的其他actor将消息根据其“地址”发送其邮箱，邮箱可以看作是一个消息队列，actor会从其“邮箱”取出消息，根据不同的消息值做不同的逻辑处理。

下面通过例子代码来学习Actor，在这之前我们需要新建一个maven项目，并引入akka依赖：

```xml
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-actor_2.13</artifactId>
    <version>2.6.4</version>
</dependency>
```

注意，最开始akka的actor是untyped的，后来较新版的akka出了typed的，上面的依赖是untyped的，这里学习的也是untyped的。对于typed的，依赖是：
```xml
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-actor-typed_2.13</artifactId>
    <version>2.6.4</version>
</dependency>
```



# 定义一个actor

在演示actor之前我们先定义一个简单的actor

既然actor的核心逻辑就是处理其他actor发过来的消息，那么actor应该有一个方法就是用于处理消息的。

我们通过继承`UntypedAbstractActor`来实现自己的actor，通过Override名为`onReceive`的方法来处理消息。

DemoActor.java
```java
import akka.actor.UntypedAbstractActor;

public class DemoActor extends UntypedAbstractActor {

    @Override
    public void onReceive(Object message) throws Throwable {
        System.out.println("Received message : " + message.toString());
    }

}

```

上面的`DemoActor`在接收到消息后打印了消息。


# 实例化actor

actor并不支持像普通的Java类一样，通过new操作来新建一个实例。下面的代码虽然可以编译通过，但是执行时会报错：

Main.java
```
public class Main {

    public static void main(String[] args) {
        DemoActor demoActor = new DemoActor();
    }

}
```

那么怎么实例化一个actor实例呢？看下面的代码：

Main.java
```
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.actor.Props;

public class Main {

    public static void main(String[] args) {
        ActorSystem actorSystem = ActorSystem.create("test");
        ActorRef actorRef = actorSystem.actorOf(Props.create(DemoActor.class), "demo");
    }
}
```

我们不能直接得到Actor实例，只能得到ActorRef，它是一个Actor实例的引用，对Actor发消息也只能通过调用ActorRef的方法来实现。实际上，akka的ActorSystem可以是分布式的，假设有2台机器A和B，一个真正的Actor实例可以是跑在B机器上的，而在A机器上可以得到ActorRef，它是对B机器上的Actor的引用，我们对ActorRef的操作实际是操作的B机器的Actor，ActorRef为我们屏蔽了底层的网络传输。

# ActorSystem

什么是ActorSystem呢？可以把ActorSystem比喻成线程池，而Actor是线程池里面的线程。线程池里面的线程当然是线程池来负责创建，而不是显示手动创建。要得到ActorRef，我们先要得到ActorSystem。

ActorSystem是一个很重量级的对象，一般一个JVM里面应该只创建一个。创建ActorSystem时需要指定一个名字。如上面的代码所示。


ActorSystem把里面的actor组织成树状的组织结构，就像linux的文件系统一样，linux的path(目录和文件)就是树状结构，除了根path“/”以外，每个path都有唯一的父path。ActorSystem里面每个Actor都有其`ActorPath`,一个

那么我们上面的代码

```java
ActorSystem actorSystem = ActorSystem.create("untyped-actor-system");
```

创建的这个“system”里根路径是啥呢？通过lookupRoot方法可以得到“根actor“：

```java
public static void main(String[] args) {
    ActorSystem actorSystem = ActorSystem.create("test");
    ActorRef actorRef = actorSystem.lookupRoot();
    ActorPath path = actorRef.path();
    System.out.println(path);
    actorSystem.terminate();
}
```

输出：

```txt
akka://test/
```

可以看出“根actor“使用像文件系统的表示方式，可以表示为：`akka://${name}/`

使用ActorSystem的actorOf方法新添加一个actor时需要指定一个actor的名字，这个名字需要是唯一的，同一个ActorSystem下，显然不能有2个actor是一样的名字。那么新添加的actor其路径表示是啥呢？

```java
public static void main(String[] args) {
    ActorSystem actorSystem = ActorSystem.create("test");
    ActorRef actorRef = actorSystem.actorOf(Props.create(DemoActor.class), "demo");
    ActorPath path = actorRef.path();
    System.out.println(path);
    actorSystem.terminate();
}
```

输出：

```txt
akka://test/user/demo
```

可以看出路径是`akka://test/user/demo`，而不是`akka://test/demo`,中间多了一层”user“

实际上create一个ActorSystem后，在我们使用actorOf添加actor之前，ActorSystem里并不是出了根以以外一个actor都没有，至少有3个：

- akka://test/user
- akka://test/system
- akka://test/deadLetters

`akka://test/system`是“系统actor”，它还会有一些已经存在的子acotr，我们创建的acotr都是`akka://test/user`的子actor，如果在上面的demoActor中我们又创建了一个actor，名为“foo”,那么它的父亲`akka://test/user/demo`,而其自己的路径是`akka://test/user/demo/foo`

# 发送消息

发送消息使用ActorRef的tell方法。

下面演示使用tell方法发送字符串“hello”给demoActor，demoActor应该会打印出它接收到的消息。

tell方法的签名：

```java
public final void tell(final Object msg, final ActorRef sender)
```

第一个参数就是要发送的消息，第二个参数也会ActorRef，表示发送者actor，前文有说过，消息在不同的actor之间传递,也就是说一个acotr接收到的消息肯定是另一个actor发给它的。那我们需要再写一个发送者actor吗？这里不需要，tell方法不需要对方回复，我们可以使用一个“匿名”的actor发送，这个“匿名”的acotr由`ActorRef.noSender()`得到，其实它就是前面说的`akka://test/deadLetters`

```java
public static void main(String[] args) {
    ActorSystem actorSystem = ActorSystem.create("test");
    ActorRef actorRef = actorSystem.actorOf(Props.create(DemoActor.class), "demo");
    actorRef.tell("hello", ActorRef.noSender());
    actorSystem.terminate();
}
```

输出：
```txt
Received message : hello
```

# 远程Actor

## 初始化远程ActorSystem

下面的代码初始化的是本地的ActorSystem，因为create的时候只传了一个名字，没有自定义配置，全用的是默认配置，所以初始化出来的是一个Local的ActorSystem

```java
ActorSystem actorSystem = ActorSystem.create("test");
```

其实，在create的时候，可以传一个Config类型的参数：

java
```java
public Config create(String name, Config config)
```

scala
```scala
def create(name: String, config: Config): ActorSystem = apply(name, config)
```

这个Config从那里来呢？一般通过配置文件来初始化，比如在maven项目的resources目录下放置配置文件"sys.conf":

sys.conf

```
akka {
    actor {
        provider = remote
    }
    remote.artery.enabled = false
    remote.classic {
        enabled-transports = ["akka.remote.classic.netty.tcp"]
        netty {
            tcp {
                hostname = "192.168.43.104"
                port = 52552
            }
        }
    }
}
```

然后使用下面的代码来初始化ActorSystem：

```java
ActorSystem actorSystem = ActorSystem.create("sys", ConfigFactory.parseResources("sys.conf"));
```

上面的配置，设置了`akka.actor.provider = reomte`, 并配置了使用netty作为底层网络库，所以需要把netty加入依赖,当然`actor-remote`也要加入依赖：

```
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-remote_2.13</artifactId>
    <version>2.6.4</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty</artifactId>
    <version>3.10.6.Final</version>
</dependency>
```

同时配置了主机名hostname和监听的端口。


## 例子代码

项目源码结构：

```txt
.
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── tmzk
    │   │       └── akka
    │   │           └── untyped
    │   │               ├── DemoActor.java
    │   │               ├── RemoteActorSystemNode1.java
    │   │               └── RemoteActorSystemNode2.java
    │   └── resources
    │       ├── logback.xml
    │       ├── node1.conf
    │       └── node2.conf
    └── test
        └── java
```

pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>tmzk</groupId>
    <artifactId>akka</artifactId>
    <version>1.0-SNAPSHOT</version>

    <name>akka</name>
    <description>akka study</description>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>12</maven.compiler.source>
        <maven.compiler.target>12</maven.compiler.target>
        <akka.version>2.6.4</akka.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.typesafe.akka</groupId>
            <artifactId>akka-actor-typed_2.13</artifactId>
            <version>${akka.version}</version>
        </dependency>
        <dependency>
            <groupId>com.typesafe.akka</groupId>
            <artifactId>akka-actor_2.13</artifactId>
            <version>${akka.version}</version>
        </dependency>
        <dependency>
            <groupId>com.typesafe.akka</groupId>
            <artifactId>akka-remote_2.13</artifactId>
            <version>${akka.version}</version>
        </dependency>
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty</artifactId>
            <version>3.10.6.Final</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.2.3</version>
        </dependency>

    </dependencies>

</project>
```

logback.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- This is a development logging configuration that logs to standard out, for an example of a production
        logging config, see the Akka docs: https://doc.akka.io/docs/akka/2.6/typed/logging.html#logback -->
    <appender name="STDOUT" target="System.out" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>[%date{ISO8601}] [%level] [%logger] [%thread] [%X{akkaSource}] - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>1024</queueSize>
        <neverBlock>true</neverBlock>
        <appender-ref ref="STDOUT" />
    </appender>

    <root level="INFO">
        <appender-ref ref="ASYNC"/>
    </root>

</configuration>

```

node1.conf
```
akka {
    actor {
        provider = remote
    }
    remote.artery.enabled = false
    remote.classic {
        enabled-transports = ["akka.remote.classic.netty.tcp"]
        netty {
            tcp {
                hostname = "192.168.43.104"
                port = 52552
            }
        }
    }
}
```

node2.conf:
```
akka {
    actor {
        provider = remote
    }
    remote.artery.enabled = false
    remote.classic {
        enabled-transports = ["akka.remote.classic.netty.tcp"]
        netty {
            tcp {
                hostname = "192.168.43.104"
                port = 52553
            }
        }
    }
}
```

DemoActor.java
```java
package tmzk.akka.untyped;

import akka.actor.ActorRef;
import akka.actor.UntypedAbstractActor;

public class DemoActor extends UntypedAbstractActor {


    @Override
    public void onReceive(Object message) throws Throwable {
        ActorRef sender = getSender();
        System.out.println("Received message : " + message.toString() + " from " + sender.path());
        if (sender != ActorRef.noSender()) {
            Integer c = (Integer) message;
            if (c < 5) {
                ++c;
                sender.tell(c, self());
            }
        }
    }

    @Override
    public void preStart() throws Exception {
        System.out.println("DemoActor starting ...");
    }
}

```

RemoteActorSystemNode1.java
```java
package tmzk.akka.untyped;

import com.typesafe.config.ConfigFactory;
import akka.actor.ActorRef;
import akka.actor.ActorSelection;
import akka.actor.ActorSystem;
import akka.actor.Props;

public class RemoteActorSystemNode1 {

    public static void main(String[] args) {
        ActorSystem actorSystem = ActorSystem.create("sys", ConfigFactory.parseResources("node1.conf"));
        ActorRef actorRef = actorSystem.actorOf(Props.create(DemoActor.class), "demo");
        ActorSelection actorSelection = actorSystem.actorSelection("akka.tcp://sys@192.168.43.104:52552/user/demo");
        actorSelection.tell(0, actorRef);
    }
}

```

RemoteActorSystemNode2
```java
package tmzk.akka.untyped;

import com.typesafe.config.ConfigFactory;
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.actor.Props;

public class RemoteActorSystemNode2 {

    public static void main(String[] args) {
        ActorSystem actorSystem = ActorSystem.create("sys", ConfigFactory.parseResources("node2.conf"));
        ActorRef actorRef = actorSystem.actorOf(Props.create(DemoActor.class), "demo");
    }

}

```

2个节点上的ActorSystem都用听一个Actor类DemoActor实例了一个Actor，但是它们的地址是不同的，端口不同。收发消息都使用DemoActor的方法，为了防止死循环，限定了来往消息一共6条。先启动node2，在启动node1，可以看到它们各自的输出，有3个轮回的消息往来。

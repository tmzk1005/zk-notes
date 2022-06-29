---
title: "Netty-User-Guide学习笔记"
date: 2021-01-31T20:22:39+08:00
draft: false
tags: ["java", "netty"]
toc: true
---

# Writing a Discard Server

世界上最简单的协议并不是“Hello World”，而是“Discard”，这个协议将会丢弃掉任何收到的数据，而不发送任何响应数据给客户端。

要实现“Discard"协议，只需要丢弃所收到的数据包即可，让我们实现一个handler，handler在netty中专门负责处理netty产生的IO事件。

## DiscardServerHandler.java
```java
package tmzk.lab.netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

public class DiscardServerHandler extends ChannelInboundHandlerAdapter { // (1)

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) { // (2)
        // Discard the received data silently.
        ((ByteBuf) msg).release(); // (3)
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) { // (4)
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }

}
```

1. `DiscardServerHandler`继承自`ChannelInboundHandlerAdapter`，而`ChannelInboundHandlerAdapter`又继承自抽象类`ChannelHandlerAdapter`并实现类接口`ChannelInboundHandler`，接口`ChannelInboundHandler`定义了若干需要实现的接口方法，对于实现“Discard"协议的例子，通过继承`ChannelInboundHandlerAdapter`就可以了，不需要直接实现此接口。

2. 我们重写了事件处理方法`channelRead`，这个方法在当有数据包到来时会被调用，在上面的例子代码中，收到的数据包用`ByteBuf`类表示。

3. 为了实现“Discard"协议，这个Handler忽略了收到的数据包。`ByteBuf`是一个引用计数对象，这个对象必须显示地调用`release()`方法来释放。需要谨记的是，Handler对象必须释放传递给它的所有引用计数对象，一般来说，Handler的`channelRead()`方法应该像下面打代码一样来实现：

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    try {
        // Do something with msg
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```

4. 当有异常发生时，`exceptionCaught`方法会被调用，在此方法里可以对错误做出处理。实际场景根据实际的业务做出异常处理，一般都需要在此方法中关闭掉关联的Channel，就如同上面的代码一样。

实现了`DiscardServerHandler`只完成了一半任务，我们还需要实现`main`方法，启动服务进程。

## DiscardServer.java
```java
package tmzk.lab.netty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class DiscardServer {

    private final int port;

    public DiscardServer(int port) {
        this.port = port;
    }

    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap(); // (2)
            b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class) // (3)
                .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new DiscardServerHandler());
                    }
                })
                .option(ChannelOption.SO_BACKLOG, 128)          // (5)
                .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)

            // Bind and start to accept incoming connections.
            ChannelFuture f = b.bind(port).sync(); // (7)

            // Wait until the server socket is closed.
            // In this example, this does not happen, but you can do that to gracefully
            // shut down your server.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        }
        new DiscardServer(port).run();
    }
}
```

1. `NioEventLoopGroup`是是多线程“EventLoop”，专门用于处理IO操作，它是接口`EventLoopGroup`的一个实现类。Netty提供了好几种`EventLoopGroup`的实现类。当使用Netty实现一个网络服务器服务端时，一般会需要用到2个`EventLoopGroup`，第一个一般称为“boss”，专门接受新建的网络连接；第二个一般称之为“worker”，当boos建立新的连接，得到“Connection”后，boos会将Connection给到worker去处理网络流量。`EventLoopGroup`使用多少的线程，以及如何创建“Channel”由`EventLoopGroup`的实现类决定，一般可以通过构造函数的参数控制。

2. `ServerBootstrap`是一个帮助类型的工具类，方便我们建立起一个服务器。

3. 这里设置了Channel的类型为”NioServerSocketChannel“，这个类型在每个平台上都可用。我们可以判断操作系统类型来设置不同类型的Channel实现，比如linux平台可以设置为”EpollServerSocketChannel“。

4. 此处设置的Handler用于在一个新的连接建立是初始化Channel，每个新的连接建立了就会调用`initChannel`方法，在此方法设置`pipeline`,用户处理即将输入的数据。

5. 可以用`option`方法设置一些channel的参数，根据不同的服务类型可以适当的配置一些参数以优化服务器的性能。这个例子里我们写的是一个TCP服务器，因此可以设置如”tcpNoDelay“和”keepAlive“等配置。可以看“ChannelOption”的实现，看都有些什么配置项。

6. 这里也是设置option，但是和5不同的是这里的方法是`childOption`。在上面的代码中，`option`方法是配置`NioServerSocketChannel`的，而`childOption`是配置被“ServerChannel”接受的“Channel“的，在上面的代码中是“NioSocketChannel”。

7. 固定用法，设置监听的端口，启动服务器。

## 测试是否收到数据

启动`DiscardServer`，执行`netstat -tnlp`应该可以看到8080端口被监听，说明服务器启动成功。使用nc命令执行`nc localhost 8080`，然后发送一些数据，可以看到nc命令并没有退出，但是也没有数据输出显示，因为服务器没有发送响应。怎么验证服务器确实收到了数据了呢？我们可以稍微修改下`DiscardServerHandler`类的`channelRead`方法，将收到的数据打印出来。

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf in = (ByteBuf) msg;
    try {
        while (in.isReadable()) {
            System.out.print((char) in.readByte());
            System.out.flush();
        }
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```

此时重新启动服务，执行`nc localhost 8080`建立连接发送数据，可以看到服务端有数据输出，但是客户端还是没有输出。


# Writing an Echo Server

DiscardServer只打印了收到的数据，并没有给客户端发送响应。一般来说，一个服务器不会只收数据而不响应客户端，因此我们实现将收到的数据原样的会写给客户端，我们称之为“Echo“协议，要实现这个“Echo“协议，和“Discard”协议的为你区别就是稍微修改下`DiscardServerHandler`类的`channelRead`方法：

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ctx.write(msg);
    ctx.flush();
}
```

此时重新启动服务，执行`nc localhost 8080`建立连接发送数据，可以看到客户端输入的数据会原样在打印出来。

# Writing a Time Server

现在我们来实现一个新的协议：“TIME”协议。这个协议和前面的例子不同的是，当新的客户端建立连接时，服务端主动发送一条含有32比特
的信息(一个integer)，而不需要客户端发送任何数据，在服务端发送数据完了后，服务端会主动断开连接。在这个例子中将会演示创建和发送一条消息给客户端，以及如何关闭连接。

我们在这个例子中将忽略掉任何客户端发送的数据，但是在连接建立之后立即发送一条消息给客户端，这次不需要实现`hannelRead()`方法，而是实现`channelActive()`方法：

```java
package tmzk.lab.netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

public class TimeServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(final ChannelHandlerContext ctx) { // (1)
        final ByteBuf time = ctx.alloc().buffer(4); // (2)
        time.writeInt((int) (System.currentTimeMillis() / 1000L + 2208988800L));

        final ChannelFuture f = ctx.writeAndFlush(time); // (3)
        f.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) {
                assert f == future;
                ctx.close();
            }
        }); // (4)
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

1. 一旦有新的客户端建立了连接，`channelActive()`将会被Netty框架调用。我们将在这个方法中向客户端写入一个代表当前时间的32位整型。

2. 为了发送消息，我们需要建立一个包含此消息的ByteBuf，因为我们要写一个32位的整型，因此ByteBuf的大小是4字节。

3. 调用`ChannelHandlerContext`对象的`writeAndFlush`方法将`ByteBuf`发送出去。

4. 第3步会返回一个`ChannelFuture`对象，让我们在消息成功发送后得到通知，让我们有机会做一些处理，比如例子中的关闭连接。

稍微修改下DiscardServer的代码，使用TimeServerHandler。

TimeServer.java
```java
package tmzk.lab.netty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class TimeServer {

    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap(); // (2)
            b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class) // (3)
                .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new TimeServerHandler());
                    }
                })
                .option(ChannelOption.SO_BACKLOG, 128)          // (5)
                .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)
            ChannelFuture f = b.bind(8080).sync(); // (7)
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        new TimeServer().run();
    }
}
```

启动服务器后同样可以执行命令`nc localhost 8080`连接服务器。这次nc进程将会打印出一段乱码然后立即退出。立即退出是因为服务端发送完消息后主动关闭了连接，打印出乱码是因为服务端发送的4个字节被nc当做纯文本输出。

那么如何解析服务端发送的数据呢，需要自己写一个Socket客户端程序。

# Writing a Time Client

在Netty中实现服务端和客户端的最大不同之处在于使用了不同的`Bootstrap`和`Channel`的实现类，看下面的代码：

TimeClient.java
```java
package tmzk.lab.netty;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

public class TimeClient {
    public static void main(String[] args) throws Exception {
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap(); // (1)
            b.group(workerGroup); // (2)
            b.channel(NioSocketChannel.class); // (3)
            b.option(ChannelOption.SO_KEEPALIVE, true); // (4)
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new TimeClientHandler());
                }
            });

            // Start the client.
            ChannelFuture f = b.connect(host, port).sync(); // (5)

            // Wait until the connection is closed.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}
```

1. `Bootstrap`和`ServerBootstrap`相似但是又有区别，`Bootstrap`是在网络客户端编程时使用的。

2. 如果在调用`group`方法时只指定一个`EventLoopGroup`，那么它就即是“boss“也是”worker“，一般在客户端编程时只需要使用一个`EventLoopGroup`

3. 和服务端使用`NioServerSocketChannel`不同，客户端使用的是`NioSocketChannel`。

4. 和前面的服务端编程不同，客户端编程不需要使用`childOption`，因为客户端的`SocketChannel`并没有一个父Channel

5. 客户端使用`connect`建立连接，而不是服务端的`build`

还需要实现一个`TimeClientHandler`类：

TimeClientHandler.java
```java
package tmzk.lab.netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import java.util.Date;

public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf m = (ByteBuf) msg; // (1)
        try {
            long currentTimeMillis = (m.readUnsignedInt() - 2208988800L) * 1000L;
            System.out.println(new Date(currentTimeMillis));
            ctx.close();
        } finally {
            m.release();
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

这段代码一般来说可以正常工作，但是理论上有可能抛出`IndexOutOfBoundsException`异常从而并不能正常工作。至于为什么以及如何解决，在下届解释。

# Dealing with a Stream-based Transport

## TCP粘包问题

在使用TCP/IP协议传输数据时，socket收到的数据是先缓存在一定大小的缓存中的，而不是直接送到应用层。缓存可以看作是一个队列，但是它不是“数据包”的队列，而是“字节”的队列，也就是说，虽然发送端发送了2条消息，产生了2个数据包，但是操作系统并不会将这2个数据包看作是2条消息，而是把它们看作是以字节为单位的字节流。因此如何区分出2条消息，需要由应用层去解决。

如果发送端发送了3个“Frame”，这样表示: “ABC-DEF-GHI”

接收端有可能执行了4次read才读完数据，得到了4个“Frame”：“AB-CDEF-G-I”

因此，对于数据接收方来说，不管是服务端，还是客户端，需要“整理”收到的数据，划分为若干有意义的“frame”，和发送端所预计的一样。

## 第一种解决办法

在前面的例子TimeClientHandler中，就可能会出现因TCP粘包问题而报错。32个bit的数据比较小，4个字节实际上不太可能被分为多个“Frame”，但是在理论上是会出现的。怎么解决呢？最简单的方法就是创建一个ByteBuf作为缓存对象，一直读取收到的数据到这个缓存对象，直到读取到至少4个字节。TimeClientHandler代码修改如下：

TimeClientHandler.java
```java
package tmzk.lab.netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import java.util.Date;

public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    private ByteBuf buf;

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) {
        buf = ctx.alloc().buffer(4); // (1)
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) {
        buf.release(); // (1)
        buf = null;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf m = (ByteBuf) msg;
        buf.writeBytes(m); // (2)
        m.release();

        if (buf.readableBytes() >= 4) { // (3)
            long currentTimeMillis = (buf.readUnsignedInt() - 2208988800L) * 1000L;
            System.out.println(new Date(currentTimeMillis));
            ctx.close();
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

1. `ChannelHandler`有2个生命周期回调方法：`handlerAdded`和`handlerRemoved`，我可以在`handlerAdded`里初始化缓存ByteBuf，在`handlerRemoved`里释放缓存。

2. 收到的数据先不做业务处理，而是存放到建立的ByteBuf缓存

3. 数据放入缓存后，检查是不是4个字节的数据都到了，如果没有到，则不执行业务逻辑处理，等下一批数据到了后Netty框架会再次`channelRead`方法，只要发送端正常，4个字节的数据肯定会最终到达，到了之后就做业务处理。

## 第二种解决办法

虽然第一种解决办法已经解决了Time Client的问题，但是实现方式并不是那么好。试想我们要实现的不是“Time”协议，而是一个复杂的多的其他什么协议，比如发送端会依此发送一个int，一个float，一个字符串，安照第一种的实现方式，要不停的检查数据是否到达，如果到了4个字节做一些处理，如果到了8个字节，又做一些处理，代码会变得不易读也难以维护。

看看`ChannelPipeline`的API，它其实可以添加不止一个`ChannelHandler`。因此，我们可以将一个复杂的`ChannelHandler`的逻辑分成若干个更简单的，更小的模块化的`ChannelHandler`。例如，可以将`TimeClientHandler`分为2个`ChannelHandler`：

- `TimeDecoder` 专门实现解码功能，解决“frame”问题。
- `TimeClientHandler` 简化后的TimeClientHandler，专注于业务逻辑

非常幸运，Netty提供了方便实现解码功能的基础抽象类`ByteToMessageDecoder`，我们可以通过继承它来实现`TimeDecoder`：

TimeDecoder.java
```java
package tmzk.lab.netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageDecoder;
import java.util.List;

public class TimeDecoder extends ByteToMessageDecoder { // (1)

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception { // (2)
        if (in.readableBytes() < 4) {
            return; // (3)
        }
        out.add(in.readBytes(4)); // (4)
    }

}
```

1. `ByteToMessageDecoder`是一个抽象类，它实现了`ChannelInboundHandler`接口，通过继承它可以很方便的解决“frame”问题。

2. `ByteToMessageDecoder`在内部维持一个和前面第一种解决办法代码中一样的ByteBuf缓存对象，当有新的数据到来时就会先把新来的数据写如这个缓存，然后调用`decode`方法，并把ByteBuf缓存对象作为参数传入此方法。

3. 如果没有足够的数据到来，decode方法可以决定直接return，等待下一次执行。

4. `decode`方法调用List的add方法添加一个对象到out，意味着decoder成功的解码了一条消息。`ByteToMessageDecoder`将会把缓存中已经读取并处理的数据给Discard

我们现在是使用2个Handler，因此要修改下`ChannelInitializer`实现：

```java
b.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(new TimeDecoder(), new TimeClientHandler());
    }
});
```

注意先后顺序，TimeDecoder在前，TimeClientHandler在后。

# Speaking in POJO instead of ByteBuf

前面的所有例子代码中都是使用ByteBuf作为承载数据的数据结构，更好的，更面向对象的方式是用POJO来代替ByteBuf。

使用POJO来代替ByteBuf有显然的好处，更加面向对象，更明确的业务意义。对于前面的“TIME”协议，定义一个UnixTime对象：

UnixTime.java
```java
package tmzk.lab.netty;

import java.util.Date;

public class UnixTime {

    private final long value;

    public UnixTime() {
        this(System.currentTimeMillis() / 1000L + 2208988800L);
    }

    public UnixTime(long value) {
        this.value = value;
    }

    public long value() {
        return value;
    }

    @Override
    public String toString() {
        return new Date((value() - 2208988800L) * 1000L).toString();
    }
}
```

有了这个UnixTime类，可以修改TimeDecoder的decode方法如下：

```java
@Override
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    if (in.readableBytes() < 4) {
        return;
    }
    out.add(new UnixTime(in.readUnsignedInt()));
}
```

这样在下游的TimeClientHandler收到的就是UnixTime对象，而不是ByteBuf，处理起来更容易，其channelRead方法可以这样写：

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    UnixTime m = (UnixTime) msg;
    System.out.println(m);
    ctx.close();
}
```

上面的POJO替代ByteBuf只体现在了数据接收方，和数据接收方的decode阶段。同样也可以体现在数据发送方，和数据发送发的encode阶段。

修改T`imeServerHandler`的`channelActive`方法：

```java
@Override
public void channelActive(ChannelHandlerContext ctx) {
    ChannelFuture f = ctx.writeAndFlush(new UnixTime());
    f.addListener(ChannelFutureListener.CLOSE);
}
```

之前是writeAndFlush写ByteBuf，现在是写UnixTime对象，那么这对象如何序列化为字节在网络上传输呢，需要一个Encoder对象：

```java
package tmzk.lab.netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelOutboundHandlerAdapter;
import io.netty.channel.ChannelPromise;

public class TimeEncoder extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        UnixTime m = (UnixTime) msg;
        ByteBuf encoded = ctx.alloc().buffer(4);
        encoded.writeInt((int)m.value());
        ctx.write(encoded, promise); // (1)
    }
}
```

更简单的，TimeEncoder可以通过继承`MessageToByteEncoder`来实现：

```java
package tmzk.lab.netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;

public class TimeEncoder extends MessageToByteEncoder<UnixTime> {
    @Override
    protected void encode(ChannelHandlerContext ctx, UnixTime msg, ByteBuf out) {
        out.writeInt((int) msg.value());
    }
}
```

最后，TimeServer的的代码中用childHandler方法设置的ChannelInitializer实例也要相应的修改一下：

```java
new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(new TimeEncoder(), new TimeServerHandler());
    }
}
```

# Shutting Down Your Application

Shutting down a Netty application is usually as simple as shutting down all EventLoopGroups you created via `shutdownGracefully()`. It returns a Future that notifies you when the EventLoopGroup has been terminated completely and all Channels that belong to the group have been closed.


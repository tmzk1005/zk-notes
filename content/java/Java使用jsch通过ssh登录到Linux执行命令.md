---
title: "Java使用jsch通过ssh登录到Linux执行命令"
date: 2021-04-10T10:00:00+08:00
draft: false
tags: ["java"]
toc: false
---

Jsch是一个纯用Java语言实现的SSH协议客户端，通过Jsch可以在JVM进程内ssh登录到Linux机器执行shell命令，而不用起一个子进程执行shell命令。

依赖：
```xml
<dependency>
    <groupId>com.jcraft</groupId>
    <artifactId>jsch</artifactId>
    <version>0.1.55</version>
</dependency>
```

API演示：

```java
import com.jcraft.jsch.ChannelExec;
import com.jcraft.jsch.JSch;
import com.jcraft.jsch.JSchException;
import com.jcraft.jsch.Session;
import java.io.ByteArrayOutputStream;
import java.util.Objects;

public class RemoteShell {

    private final Session session;

    private RemoteShell(Session session) {
        this.session = session;
    }

    public static RemoteShell login(String username, String password, String hostname, int port) throws JSchException {
        Session session = new JSch().getSession(username, hostname, port);
        session.setPassword(password);
        session.setConfig("StrictHostKeyChecking", "no");
        session.connect();
        return new RemoteShell(session);
    }

    public void logout() {
        if (Objects.nonNull(session) && session.isConnected()) {
            session.disconnect();
        }
    }

    public String exec(String cmd) throws JSchException, InterruptedException {
        ChannelExec channel = (ChannelExec) session.openChannel("exec");
        channel.setCommand(cmd);
        ByteArrayOutputStream responseStream = new ByteArrayOutputStream();
        channel.setOutputStream(responseStream);
        channel.connect();
        while (channel.isConnected()) {
            //noinspection BusyWait
            Thread.sleep(100);
        }
        int exitCode = channel.getExitStatus();
        String output = responseStream.toString();
        if (exitCode != 0) {
            throw new JSchException(output);
        }
        return output;
    }

}
```

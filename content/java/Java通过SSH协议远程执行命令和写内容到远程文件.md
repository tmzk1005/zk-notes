---
title: "Java通过SSH协议远程执行命令和写内容到远程文件"
date: 2021-04-18T10:00:00+08:00
draft: false
tags: ["java"]
---

jsch是一个纯使用Java语言实现SSH协议的库，通过jsch可以实现一个JVM进程ssh登录到一个Linux远程机器执行命令，或者直接写内容到远程文件，以及其他很多通过openssh客户端做的事情，只不过不需要启动一个子进程去执行命令，而是直接在JVM内远程连接linux机器。

```java
import com.jcraft.jsch.ChannelExec;
import com.jcraft.jsch.JSch;
import com.jcraft.jsch.JSchException;
import com.jcraft.jsch.Session;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Objects;


class RemoteShell {

    private final Session session;

    private RemoteShell(Session session) {
        this.session = session;
    }

    public static RemoteShell login(String username, String password, String host, int port) throws JSchException {
        JSch jSch = new JSch();
        Session session = jSch.getSession(username, host, port);
        session.setPassword(password);
        session.setConfig("StrictHostKeyChecking", "no");
        session.connect();
        return new RemoteShell(session);
    }

    public static RemoteShell login(String username, String password, String host, int port) throws JSchException {
        return login(username, password, host, port);
    }

    public String exec(String cmd) throws Exception {
        ChannelExec channel = null;
        try {
            channel = (ChannelExec) session.openChannel("exec");
            channel.setCommand(cmd);
            ByteArrayOutputStream responseStream = new ByteArrayOutputStream();
            channel.setOutputStream(responseStream);
            channel.connect();
            while (channel.isConnected()) {
                Thread.sleep(100);
            }
            String output = responseStream.toString();
            if (channel.getExitStatus() != 0) {
                throw new Exception(output);
            }
            return output;
        } finally {
            if (Objects.nonNull(channel)) {
                channel.disconnect();
            }
        }
    }

    public boolean isConnected() {
        return Objects.nonNull(session) && this.session.isConnected();
    }

    public void logout() {
        if (isConnected()) {
            session.disconnect();
        }
    }

    public void scpTo(Path localFile, String remoteFile) throws JSchException, IOException {
        writeContentToRemoteFile(remoteFile, Files.newInputStream(localFile), localFile.toFile().length());
    }

    public void sEcho(byte[] content, String remoteFile) throws JSchException, IOException {
        writeContentToRemoteFile(remoteFile, new ByteArrayInputStream(content), content.length);
    }

    public void writeContentToRemoteFile(String remoteFile, InputStream contentStream, long byteCount) throws JSchException, IOException {
        ChannelExec channelExec = (ChannelExec) session.openChannel("exec");
        // 注意：这个scp不是bash里的scp命令，-t也不是scp命令的选项，因此man scp是看不到这个选项的
        // 这里的内容是用ssh通道传输二进制文件时的协议规定的，因此不要和scp命令混淆，这里体现的是scp命令更加底层的协议。
        String cmd = "scp -t " + remoteFile;
        channelExec.setCommand(cmd);
        try (
                OutputStream out = channelExec.getOutputStream();
                InputStream in = channelExec.getInputStream();
                contentStream
        ) {
            channelExec.connect();
            String msg = String.format("Write content to remote host %s failed at ", session.getHost());
            if (in.read() != 0) {
                throw new IOException(msg + "step-1");
            }
            // noName是随便取个一个名字，表示本地文件名
            cmd = "C0644 " + byteCount + " noName\n";
            out.write(cmd.getBytes(StandardCharsets.UTF_8));
            out.flush();
            if (in.read() != 0) {
                throw new IOException(msg + "step-2");
            }
            contentStream.transferTo(out);
            out.write(0);
            out.flush();
            if (in.read() != 0) {
                throw new IOException(msg + "step-3");
            }
        } finally {
            channelExec.disconnect();
        }
    }

}
```

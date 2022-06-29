---
title: "https协议和ssl证书生成"
date: 2021-01-20T11:27:39+08:00
draft: false
tags: ["http", "https", "ssl"]
toc: true
---

# 1. https协议之SSL/TLS握手

使用https通信时，在浏览器正式发起http请求之前，有一个SSL/TLS协议握手的过程，通过这个握手过程拿到加密的公钥后，才能使用公钥将http报文加密后发送出去。

这个握手的过程可以分为4步，基本过程如下：

- Step-1: 客户端发起请求，向服务器请求公钥

  这个请求携带的信息主要包含以下几项：

  - 支持的协议版本，比如TLS 1.0版
  - 一个客户端生成的随机数，稍后用于生成"对话密钥"
  - 支持的加密方法，比如RSA公钥加密
  - 支持的压缩方法

- Step-2: 服务器回应

  回应的信息主要包含：

  - 确认使用的加密通信协议版本，比如TLS 1.0版本。如果浏览器与服务器支持的版本不一致，服务器关闭加密通信。
  - 一个服务器生成的随机数，稍后用于生成"对话密钥"
  - 确认使用的加密方法，比如RSA公钥加密
  - 服务器证书

- Step-3: 客户端回应

  客户端收到服务器回应以后，首先验证服务器证书，如果证书不是可信机构颁布、或者证书中的域名与实际域名不一致、或者证书已经过期，就会向访问者显示一个警告，由其选择是否还要继续通信。如果证书没有问题，客户端就会从证书中取出服务器的公钥。然后，向服务器发送下面几项信息：

  - 一个随机数。该随机数用服务器公钥加密，防止被窃听。这个随机是整个握手阶段出现的第三个随机数，又称"pre-master key"。有了它以后，客户端和服务器就同时有了三个随机数，接着双方就用事先商定的加密方法，各自生成本次会话所用的同一把"会话密钥"
  - 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送
  - 客户端握手结束通知，表示客户端的握手阶段已经结束。这一项同时也是前面发送的所有内容的hash值，用来供服务器校验。

- Step-4: 服务其最后的回应

  服务器收到客户端的第三个随机数pre-master key之后，计算生成本次会话所用的"会话密钥"。然后，向客户端最后发送下面信息：

  - 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送
  - 服务器握手结束通知，表示服务器的握手阶段已经结束。这一项同时也是前面发送的所有内容的hash值，用来供客户端校验

至此，整个握手阶段全部结束。接下来，客户端与服务器进入加密通信，就完全是使用普通的HTTP协议，只不过会用"会话密钥"加密内容。

# 2. 证书生成

通过前面的的握手过程说明，我们知道服务端会将公钥发给客户端，同时自己需要保有私钥。典型的nginx服务器ssl配置如下：

```conf
server {
    listen 443;
    server_name localhost;

    ssl on;
    ssl_certificate server.crt;
    ssl_certificate_key server.key;

    location / {
        root https_443;
        index index.html index.htm;
    }
}
```

`server.crt`就是公钥文件，`server.key`就是私钥文件。

有时候公钥文件也以“pem”为后缀，因此见到“pem”后缀文件是不要迷惑，它其实就是一个纯文本文件。以”-----BEGIN XXX-----“开头，以“-----END XXX-----”结尾，中间是base64编码的数据。pem是“Privacy Enhanced Mail”的缩写。

例如：

```txt
-----BEGIN CERTIFICATE-----
MIIC1jCCAb4CCQDLPk9TQ2iimjANBgkqhkiG9w0BAQUFADAtMQswCQYDVQQGEwJj
bjEOMAwGA1UECAwFSHViZWkxDjAMBgNVBAcMBVd1aGFuMB4XDTIxMDEyNjE3NDQy
NVoXDTIyMDEyNjE3NDQyNVowLTELMAkGA1UEBhMCY24xDjAMBgNVBAgMBUh1YmVp
MQ4wDAYDVQQHDAVXdWhhbjCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
ALP3QEfwPhqjfYY/aoGxFd+Z1mrseSvc70AKiVaiUijLuwKjV6YYOEk1plpfOCM6
bQwonPdgX3Pvl1cL5YJLue7Mch4yP4PFrlI656lfrkRkQq/ejHIP5Mnra8/sixZt
lrPNSkSxHtsFPWRwyXn2/o1aSxlKgTF/9O2mrSxDnps1v1MwLtXhHlG2yngQoZxa
DvdC5pXBXVUjN6bxpLRFdVvOGPM1TPunJgk5piUQjOLr74nu9eHwTzRpZoCee3eb
NVnANVHKtxio4/jwTc6gr4L665KI/EeoFhCrTF4Fi35FNDJxcCQdEs922lYXRpRa
U+QHkHSCMlH588vH7kuGGJsCAwEAATANBgkqhkiG9w0BAQUFAAOCAQEAvFd/lRAY
USq17dHPNv5zggzceN7LWyUuzqnZe4Axe/IlfXncu9pXT+m66XspJTNE+xiASF66
1GZzbHPdTmfJ6v/PkBL0fscHDSjY6uiINu0pxVqX26OhhBUKqKig3NLjDxPASCPH
pAVFEgs9Dez3/qMVQJhXDQUpqw1HF4p0b0i7ssDn0Uyjs23IgisvxtEgYtpVNOVo
iMshXdRrPSKgkGLQiDfJrumsr/zm5ENAL6gzUTFVnVXhy/7VGFPzO3+FnLQ0T/RF
eppyuw2QBJRYahF1FY61EG9pCDeQ9EKiiaR7/WZNQIS7wvzZvcINsqWZ6YfQDFu4
V5+/Y88aan5H9w==
-----END CERTIFICATE-----
```

怎么生成这2个文件呢？生成ssl证书需要有一个“可信”的证书颁发机构（CA），分为3步：

1. 用openssl命令生成私钥文件server.key
```bash
  openssl genrsa -out server.key 2048
  ```

2. 用生成的server.key文件得到证书请求文件server.csr
  ```bash
  openssl req -new -key server.key -out server.csr
  ```

3. 用CA证书ca.crt和ca.key以及上一步得到的server.csr文件，得到证书文件server.crt
  ```bash
  openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt
  ```

那第3步用到的`ca.crt`和`ca.key`证书文件从哪里来呢？可以从权威机构来，测试的话也可以用openssl生成一个自签名的ca.crt,步骤和上面类似：

1. 生成ca.key

  ```bash
  openssl genrsa -out ca.key 2048
  ```

2. 生成ca.csr

  ```bash
  openssl req -new -key ca.key -out ca.csr
  ```

3. 生成ca.crt

  ```bash
  openssl x509 -req -days 3650 -in ca.csr -signkey ca.key -out ca.crt
  ```

可以用openssl命令验证生成的证书：

```bash
openssl x509 -in server.crt -text -noout
```


---
title: "本地搭建kafka离线文档"
date: 2022-06-25T10:00:00+08:00
draft: false
tags: ["java", "kafka"]
toc: true
---

`kafka`项目的官方网站[https://kafka.apache.org/](https://kafka.apache.org/)可以看kakfa的文档，但是国内访问总感觉太慢，而且没网的时候还看不了，那怎么弄个kafka的离线文档呢？

其实`kafka`项目的官方网站是个静态网站，其后台的html代码也是开源的，我们可以在本地把这个网站跑起来。

# 1. 克隆网站后台静态文档

地址是[https://github.com/apache/kafka-site.git](https://github.com/apache/kafka-site.git)

用git克隆下来:

```sh
git clone https://github.com/apache/kafka-site.git

# 或者代理加速
git clone https://ghproxy.com/https://github.com/apache/kafka-site.git
```

# 2. 用nginx运行网站

其实`kafka`项目的官方网站应该是用Apache httpd来跑的（刚才clone的代码艮目录里有.htaccess文件）。但是我个人比较喜欢用nginx, 这里说明如何配置用nginx来运行kakfa的官方文档。

## 2.1 自己编译nginx

为什么要自己编译nginx？因为`kafka`的文档用了`Server Side Include`技术，而且用了很多相对路径引用的方式，比如`../../a.html`这种形式，这种`uri`是不安全的，nginx默认是不允许的，会导致网站看起来不正常，因此需要稍稍修改下nginx的源码以便允许不安全的`uri`。

### 克隆nginx源码

```sh
git clone https://ghproxy.com/https://github.com/nginx/nginx.git
```

### 修改代码

切换到一个最新的release版本，并建立修改分支：

```sh
git checkout -b custom release-1.23.0
```

需要修改代码的地方是`src/http/ngx_http_parse.c`这个文件的`ngx_http_parse_unsafe_uri`函数，让这个函数直接返回`NGX_OK`即可。

```
diff --git a/src/http/ngx_http_parse.c b/src/http/ngx_http_parse.c
index d4f2dae8..075d8287 100644
--- a/src/http/ngx_http_parse.c
+++ b/src/http/ngx_http_parse.c
@@ -1842,6 +1842,7 @@ ngx_int_t
 ngx_http_parse_unsafe_uri(ngx_http_request_t *r, ngx_str_t *uri,
     ngx_str_t *args, ngx_uint_t *flags)
 {
+    return NGX_OK;
     u_char      ch, *p, *src, *dst;
     size_t      len;
     ngx_uint_t  quoted;
(END)
```

### 编译安装

可以自定义安装目录，这里安装目录设置为`/opt/nginx`

在nginx的源码目录执行：
```sh
auto/configure --prefix=/opt/nginx
make
make install
```
可能遇到缺少命令或者依赖包，安装即可。


## 2.2 配置nginx

需要配置的语义大致要和`kafka`文档代码目录下`.htaccess`文件apache配置的意思相当:

```
server {
    listen 10581;
    server_name localhost;
    ssi on; 
    ssi_min_file_chunk 10240k;
    ssi_types *;
    ssi_value_length 1024;
    root /path/to/kafka-site;
    index index.html;

    location / { 
        root /path/to/kafka-site;
        index index.html;
        try_files $uri $uri/ $uri/index.html $uri.html =404;
    }
}
```

端口和root路径按需修改。

# 3. 一些js文档和字体文件下载慢问题解决

经过前面2个步骤后，`kafka`的离线文档基本是能跑起来了，但是还是不完全正常。这是因为一些html文件需要下载字体文件和从cdn下
载js文件，但是下载失败。（那些会是失败，可以先把本地网站跑起来，用浏览器访问首页并打开调试工具（F12）来查看。）

字体文件可以不用管，只是字体不同而已，但是从cdn下载js失败会影响文档的阅读体验。主要
是从`https://cdnjs.cloudflare.com/`这个地址下载失败。（因为本地地址没有在这个cdn地址注册，没有权限从上面下载静态文件）。

因此我们需要把缺失的一些js文件通过别的手段下载下来，然后同样用本地的nginx来提供。

先找到以来的是那些js：

用`ag`命令在代码哭里搜索：

```sh
ag 'https://cdnjs.cloudflare.com/'
```

结果：

```txt
includes/_footer.htm
18:		<script src="https://cdnjs.cloudflare.com/ajax/libs/handlebars.js/2.0.0/handlebars.js"></script>

includes/_header.htm
22:		<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/prism/1.20.0/themes/prism.min.css" />
23:		<script defer src="https://cdnjs.cloudflare.com/ajax/libs/prism/1.20.0/components/prism-core.min.js"></script>
24:		<script defer src="https://cdnjs.cloudflare.com/ajax/libs/prism/1.20.0/plugins/autoloader/prism-autoloader.min.js"></script>
25:		<script defer src="https://cdnjs.cloudflare.com/ajax/libs/prism/1.20.0/plugins/line-numbers/prism-line-numbers.min.js"></script>
```

可以看到主要是`includes/_footer.htm`和`includes/_header.htm`这2个文件。

用`sed`命令替换成本地地址，我们在`kafka`的文档代码的根目录新建一个目录`cdnDep`来存在这些js文件，这里先把引用地址换了：

```sh
sed -i 's#https://cdnjs.cloudflare.com#/cdnDep#g' includes/_footer.htm
sed -i 's#https://cdnjs.cloudflare.com#/cdnDep#g' includes/_header.htm
```
(可以`git diff`确认修改效果)

然后想办法把这几个css文件和js文件弄到本地，按匹配的路径方法在`cdnDep`目录里。下载方法可以访问在线文档，等加载出来后复制下来。

```sh
# 假设几个文件已经放在了cdnDep目录，只是层级没匹配,可执行以下命令
mkdir -p ajax/libs/handlebars.js/2.0.0
mv handlebars.js ajax/libs/handlebars.js/2.0.0

mkdir -p ajax/libs/prism/1.20.0/themes
mv prism.min.css ajax/libs/prism/1.20.0/themes

mkdir -p ajax/libs/prism/1.20.0/components
mv prism-core.min.js ajax/libs/prism/1.20.0/components

mkdir -p ajax/libs/prism/1.20.0/plugins/autoloader
mv prism-autoloader.min.js ajax/libs/prism/1.20.0/plugins/autoloader

mkdir -p ajax/libs/prism/1.20.0/plugins/line-numbers
mv prism-line-numbers.min.js ajax/libs/prism/1.20.0/plugins/line-numbers
```

最后kakfa的离线文档基本就在本地阅读了，可以当手册一样在需要的时候查阅。

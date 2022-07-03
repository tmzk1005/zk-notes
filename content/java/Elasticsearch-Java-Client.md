---
title: "Elasticsearch Java Client"
date: 2022-07-02T23:56:21+08:00
draft: true
toc: true
tags: []
---

# 1. 安装

安装，即引入依赖。

maven
```xml
<dependencies>

    <dependency>
      <groupId>co.elastic.clients</groupId>
      <artifactId>elasticsearch-java</artifactId>
      <version>8.3.1</version>
    </dependency>

    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.12.3</version>
    </dependency>

  </dependencies>
```

gradle
```groovy
dependencies {
    implementation 'co.elastic.clients:elasticsearch-java:8.3.1'
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.12.3'
}
```

# 2. 连接

建立连接，即用各个节点的地址，得到`ElasticsearchClient`实例。

```java
// 1. 由节点地址创建RestClientBuilder
// 如果是多个节点，可以传递多个HttpHost实例
RestClientBuilder builder = RestClient.builder(new HttpHost("localhost", 9200));

// 2. 如果Elasticsearch开启了用户名密码认证，需要做相应的配置
builder.setHttpClientConfigCallback(httpClientBuilder -> {
    final BasicCredentialsProvider basicCredentialsProvider = new BasicCredentialsProvider();
    basicCredentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials("esUsername", "esPassword"));
    httpClientBuilder.setDefaultCredentialsProvider(basicCredentialsProvider);
    return httpClientBuilder;
});

// 3.  由builder创建RestClient
RestClient restClient = builder.build();

// 4. 创建ElasticsearchTransport实例
ElasticsearchTransport transport = new RestClientTransport(restClient, new JacksonJsonpMapper());

// 5. 创建客户端实例
ElasticsearchClient client = new ElasticsearchClient(transport);
```

得到`ElasticsearchClient`实例后，即可用它来做各种请求。

# 3. 常用API

下面通过一段简单的模拟业务来演示常用API的使用。

模拟业务:

一个网站，会持续的产生简单的访问日志，包含有请求开始，结束时间戳，http请求方法，请求路径，响应状态码，以及客户端IP等字段信息，访问日志需要索引到Es,因为访问日志是典型的“时间序列数据”，可以用Es的**DataStream**来存储，而且访问日志具有典型的生命周期特性：越新的日志越会被查询使用，越久远的日志越没有价值，可以被删除，很适合用**索引生命周期管理**机制来自动管理。

下面依次实现如下功能，以顺便演示和学习Elasticsearch Java Client的API：

1. 创建生命周期管理策略
2. 创建索引模板，并使用刚建立的生命周期管理策略来管理
3. 创建DataStream,并使用刚创建的索引模板
4. 网DataStream里写入数据，让Es自动创建隐藏在DataStream背后的实际索引
5. 简单过滤查询DataStream中的数据
6. 简单聚合查询DataStream中的数据

下面是演示代码会用到的`AccessLong`数据结构。

```java
public static class AccessLog {

    private long startTimestamp;

    private long endTimeStamp;

    private String clientIp;

    private String httpMethod;

    private String httpPath;

    private int responseCode;

    @JsonProperty("@timestamp")
    public long getTimestamp() {
        return startTimestamp;
    }

    public long getStartTimestamp() {
        return startTimestamp;
    }

    public void setStartTimestamp(long startTimestamp) {
        this.startTimestamp = startTimestamp;
    }

    public long getEndTimeStamp() {
        return endTimeStamp;
    }

    public void setEndTimeStamp(long endTimeStamp) {
        this.endTimeStamp = endTimeStamp;
    }

    public String getClientIp() {
        return clientIp;
    }

    public void setClientIp(String clientIp) {
        this.clientIp = clientIp;
    }

    public String getHttpMethod() {
        return httpMethod;
    }

    public void setHttpMethod(String httpMethod) {
        this.httpMethod = httpMethod;
    }

    public String getHttpPath() {
        return httpPath;
    }

    public void setHttpPath(String httpPath) {
        this.httpPath = httpPath;
    }

    public int getResponseCode() {
        return responseCode;
    }

    public void setResponseCode(int responseCode) {
        this.responseCode = responseCode;
    }
    
}
```

## 3.1 创建生命周期管理策略

TODO

## 3.2 创建索引模板

TODO

## 3.3 创建DataStream

TODO

## 3.4 写数据到DataStream

TODO

## 3.5 简单过滤查询DataStream中的数据

TODO

## 3.6 简单聚合查询DataStream中的数据 

TODO

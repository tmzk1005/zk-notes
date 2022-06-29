---
title: "EnableEurekaServer开关注解的原理"
date: 2021-05-03T10:00:00+08:00
draft: false
toc: false
---

在spring-cloud中开发一个Eureka Server作为服务注册中心非常的简单，仅仅只需要引入starter依赖，然后在main函数所在的入口类上加上一个开关注解`@EnableEurekaServer`即可，这是如何实现的呢？

这个原理其实并不复杂，简单记录一下，有需要的话，我们也可以模仿Eureka实现类似的开关注解。

从注解`@EnableEurekaServer`的源码追踪一下：

EnableEurekaServer.java
```java
package org.springframework.cloud.netflix.eureka.server;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.context.annotation.Import;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

}
```

可以看到，通过spring提供的Import机制，间接配置了`EurekaServerMarkerConfiguration`，再看看这个类的源码:

EurekaServerMarkerConfiguration.java
```java
package org.springframework.cloud.netflix.eureka.server;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration(proxyBeanMethods = false)
public class EurekaServerMarkerConfiguration {

    @Bean
    public Marker eurekaServerMarkerBean() {
        return new Marker();
    }

    class Marker {

    }

}
```

这个类是一个`@Configuration`类，配置了一个Marker类的Bean。这个Marker类的Bean正如其名，就是一个标记而已，它怎么发挥作用呢？这个要结合spring.factories自动配置机制来看，

在`spring-cloud-netflix-eureka-server`这个jar包的`META-INF`下有文件`spring.factories`，内容如下：

spring.factories
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```

配置了自动配置类`EurekaServerAutoConfiguration`，再看看它的代码：

EurekaServerAutoConfiguration.java
```java
@Configuration(proxyBeanMethods = false)
@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
        InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration implements WebMvcConfigurer {
    // ...
}
```

开关的作用就是由`@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)`这一行决定的，如果有`EurekaServerMarkerConfiguration.Marker`这个类的Bean存在，那么`EurekaServerAutoConfiguration`这个自动配置类才会有作用，如果用了`@EnableEurekaServer`这个注解，那Marker就存在，否则不存在，从而起到了开关的作用。

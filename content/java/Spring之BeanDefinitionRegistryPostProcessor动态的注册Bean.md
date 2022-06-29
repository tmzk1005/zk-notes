---
title: "Spring之BeanDefinitionRegistryPostProcessor动态注册Bean"
date: 2021-05-01T10:00:00+08:00
draft: false
toc: false
---

**问题**： 有一个写有main函数的Spring Boot应用jar包A，另外有一个jar包B中通过注解定义了Spring-MVC的一些@Controller,@Service，如何在A添加B作为依赖时，将B中定义的@Controller,@Service等注入到A应用的容器中呢？

> 解决这个问题主要是为了代码和功能模块化，将独立，内聚的一部分功能放在一个jar里

**方案一**：

在A应用中使用@ComponentScan注解，把jar包B的包路径配置为要扫描的路径。

这个方法是最简单的，但是要求改动A，而且A要知道B的包的路径。

**方案二**：

现在开发Spring应用一般都是使用Spring Boot框架了，通过Spring-Boot的自动配置机制和BeanDefinitionRegistryPostProcessor能够比较优雅的解决上面的需求。

- 实现BeanDefinitionRegistryPostProcessor接口，扫描并注册指定的包：

    ComponentScanRegister.java
    ```java
    import java.util.Objects;

    import org.springframework.beans.BeansException;
    import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
    import org.springframework.beans.factory.support.BeanDefinitionRegistry;
    import org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor;
    import org.springframework.context.annotation.ClassPathBeanDefinitionScanner;

    public class ComponentScanRegister implements BeanDefinitionRegistryPostProcessor {

        public String[] packageNames;

        public ComponentScanRegister(String... packageNames) {
            this.packageNames = packageNames;
        }

        @Override
        public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
            if (Objects.nonNull(packageNames) && packageNames.length > 0) {
                ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry);
                scanner.scan(packageNames);
            }
        }

        @Override
        public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        }

    }
    ```

- 自动配置

    JarBAutoConfiguration.java
    ```java
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;

    @Configuration(proxyBeanMethods = false)
    public class JarBAutoConfiguration {

        @Bean
        public ComponentScanRegister nglasClusterComponentScanRegister() {
            return new ComponentScanRegister(this.getClass().getPackageName());
        }

    }
    ```

    配置jar包B的spring.factory
    
    ```properties
    # AutoConfiguration
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=JarBAutoConfiguration
    ```


这种方法里，将B做成一个Spring Boot Starter，B自己是知道的自己的包扫描路径的，将之作为参数传递给ComponentScanRegister的构造函数参数即可，A不需要知道，A只要引入B作为依赖即可。

A除了引入B，可能需要因为更多的C，D ... , 因此ComponentScanRegister可以作为一个公共库，被A，B，C等依赖。

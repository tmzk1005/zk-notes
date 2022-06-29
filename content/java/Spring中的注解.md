---
title: "Spring如何知道类上有什么注解，以及注解又有什么子注解"
date: 2021-03-31T10:00:00+08:00
draft: false
toc: true
---

Spring系列的框架中有很多常用的注解，比如：

- @SpringBootApplication
- @Configuration
- @Import
- @Controller
- @Service
- @Component

用一个`@Controller`就可以标识一个类是一个Controller，一个`@Configuration`就可以标识一个类是配置类。而且更高级点的用法，用一个`@SpringBootApplication`就可以同时具有`@SpringBootConfiguration`, `@EnableAutoConfiguration`, `@ComponentScan`这3个注解的功能，因为`@SpringBootApplication`的源码是这样的：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    // ... 省略
}
```

显然，Spring框架有能力知道类上面有什么注解，以及各个注解本身的类上又有什么注解。这种能力是Spring的必备基础能力，很多上层功能的实现都依赖于此。那么Spring是如何实现这种能力的呢？

# MergedAnnotations

相关的实现源码在`org.springframework.core.annotation`这个包下。这个包下的一些底层类不是public的，不能直接调用，只有直接暴露给用户使用的类是public的，注解相关功能的学习入口就是`MergedAnnotations`这个接口。

新建一个SpringBoot应用，只写一个入口类：

App.java
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class App {

    public static void main(String[] args) {
        SpringApplication.run(App.class);
    }

}
```

我们启动这个App类，执行其main方法就可以启动一个SpringBoot应用。这背后就需要知道App类上有`@SpringBootApplication`注解，同时分析出`@SpringBootApplication`有那些子注解，以及各个子注解又有那些子注解，知道完全分析出所有的注解。

下面的一段代码可以演示这种能力：

MergedAnnotationsTest.java
```java
import java.lang.annotation.Annotation;
import org.springframework.boot.SpringBootConfiguration;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.MergedAnnotations;

public class MergedAnnotationsTest {

    public static void main(String[] args) {
        Annotation[] annotations = App.class.getDeclaredAnnotations();
        MergedAnnotations mergedAnnotations = MergedAnnotations.from(annotations);
        boolean springBootApplicationIsPresent = mergedAnnotations.isPresent(SpringBootApplication.class);
        System.out.println(springBootApplicationIsPresent);
        boolean configurationIsPresent = mergedAnnotations.isPresent(Configuration.class);
        System.out.println(configurationIsPresent);
        boolean springBootConfigurationIsPresent = mergedAnnotations.isPresent(SpringBootConfiguration.class);
        System.out.println(springBootConfigurationIsPresent);
        boolean componentScanIsPresent = mergedAnnotations.isPresent(ComponentScan.class);
        System.out.println(componentScanIsPresent);

        mergedAnnotations.stream().forEach(item -> System.out.println(item.getType()));
    }

}
```

执行这段代码，输出如下：

```txt
true
true
true
true
interface org.springframework.boot.autoconfigure.SpringBootApplication
interface org.springframework.boot.SpringBootConfiguration
interface org.springframework.boot.autoconfigure.EnableAutoConfiguration
interface org.springframework.context.annotation.ComponentScan
interface org.springframework.context.annotation.Configuration
interface org.springframework.boot.autoconfigure.AutoConfigurationPackage
interface org.springframework.context.annotation.Import
interface org.springframework.stereotype.Component
interface org.springframework.context.annotation.Import
interface org.springframework.stereotype.Indexed
```

可以看到，给定了App.class这个类，我们有能力知道此类上出现了那些注解，从上面的输出可以看到，一共有10个注解。当然，也有能力知道每个注解的各个属性值，稍微修改下上面的代码就可以展示这种能力。

```java
import java.lang.annotation.Annotation;
import java.util.Map;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.MergedAnnotation;
import org.springframework.core.annotation.MergedAnnotations;

public class MergedAnnotationsTest {

    public static void main(String[] args) {
        Annotation[] annotations = App.class.getDeclaredAnnotations();
        MergedAnnotations mergedAnnotations = MergedAnnotations.from(annotations);
        MergedAnnotation<Configuration> mergedAnnotation = mergedAnnotations.get(Configuration.class);
        Map<String, Object> map = mergedAnnotation.asMap(MergedAnnotation.Adapt.ANNOTATION_TO_MAP);
        for(String key : map.keySet()) {
            System.out.println(key + " = " + map.get(key));
        }
    }

}
```

输出：
```
proxyBeanMethods = true
value =
```
> value是空字符串

# MergedAnnotations实现原理

一切功能的入口都是前面代码中用到的`MergedAnnotations`接口的静态from方法：

```java
static MergedAnnotations from(Annotation... annotations) {
    return from(annotations, annotations);
}
```

最终调用的是：

```java
static MergedAnnotations from(Object source, Annotation[] annotations, RepeatableContainers repeatableContainers) {
    return TypeMappedAnnotations.from(source, annotations, repeatableContainers, AnnotationFilter.PLAIN);
}
```

可以看到，实际实现功能的方法是`TypeMappedAnnotations`类的静态`from`方法。`TypeMappedAnnotations`这个类不是public的，因此只能通过MergedAnnotations接口来间接使用它。

调用`from`方法的时候并不会立即就把所有的注解信息解析出来，这背后有一个**懒解析**的思想，第一次真正用的时候才会去解析，比如前面代码中调用`isPresent`方法或者`get`方法时。解析出来后的结果会放在一个缓存map里，后面再获取注解信息时会直接从缓存获取。核心代码如下：

```java
private static final Map<AnnotationFilter, Cache> standardRepeatablesCache = new ConcurrentReferenceHashMap<>();

// ...

if (repeatableContainers == RepeatableContainers.standardRepeatables()) {
    return standardRepeatablesCache.computeIfAbsent(annotationFilter,
            key -> new Cache(repeatableContainers, key)).get(annotationType);
}

```

java.utils.Map.java
```java
default V computeIfAbsent(K key,
        Function<? super K, ? extends V> mappingFunction) {
    Objects.requireNonNull(mappingFunction);
    V v;
    if ((v = get(key)) == null) {
        V newValue;
        if ((newValue = mappingFunction.apply(key)) != null) {
            put(key, newValue);
            return newValue;
        }
    }

    return v;
}
```

这个**懒解析**的功能，其实就是jdk的Map接口的实现，倒并不是Spring专门实现的。从`computeIfAbsent`方法可以看到，第一次get到null值，因此会执行给定的回调方法，第二次就直接拿缓存了。

继续往下看源码，下一步的重要逻辑是`Cache`类的`get`方法：

```java
AnnotationTypeMappings get(Class<? extends Annotation> annotationType) {
    return this.mappings.computeIfAbsent(annotationType, this::createMappings);
}
```

核心逻辑在`Cache`类的`createMappings`方法，此方法调用了`AnnotationTypeMappings`的构造方法：

```java
private AnnotationTypeMappings(RepeatableContainers repeatableContainers,
        AnnotationFilter filter, Class<? extends Annotation> annotationType) {

    this.repeatableContainers = repeatableContainers;
    this.filter = filter;
    this.mappings = new ArrayList<>();
    addAllMappings(annotationType);
    this.mappings.forEach(AnnotationTypeMapping::afterAllMappingsSet);
}
```

最终的核心逻辑就在这个`addAllMappings`方法：

```java
private void addAllMappings(Class<? extends Annotation> annotationType) {
    Deque<AnnotationTypeMapping> queue = new ArrayDeque<>();
    addIfPossible(queue, null, annotationType, null);
    while (!queue.isEmpty()) {
        AnnotationTypeMapping mapping = queue.removeFirst();
        this.mappings.add(mapping);
        addMetaAnnotationsToQueue(queue, mapping);
    }
}
```

这里用了一个**双端队列广度遍历**的思想，从一个初始的注解（比如@SpringBootApplication）出发，用反射的方式遍历出了所有的相关注解。

最终，我们就具有了通过调用`MergedAnnotations`接口的方法得到注解信息的能力。

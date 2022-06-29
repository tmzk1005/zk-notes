---
title: "Spring容器的初始化过程（以SpringBoot应用为例）"
date: 2021-04-01T10:00:00+08:00
draft: false
toc: true
---

# 引子
先写一个包含3个类的简单SrpingBoot应用:

Bar.java
```java
import org.springframework.stereotype.Component;

@Component
public class Bar {
}
```

Foo.java
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class Foo {

    @Autowired
    Bar bar;

}
```

App.java
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

@SpringBootApplication
public class App {

    public static void main(String[] args) {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(App.class);
        Foo foo = applicationContext.getBean("foo", Foo.class);
        Bar bar = applicationContext.getBean("bar", Bar.class);
        System.out.println(foo.bar == bar);
    }

}
```

运行App.java会输出“true”，`SpringApplication.run`返回的`ConfigurableApplicationContext`就是一个初始化好了的Bean容器，从这个applicaitonContext中可以调用`BeanFactory`接口的`getBean`等方法取出想要的Bean。那么Spring内部（更确切的说，在这个例子里以SpringBoot应用来演示）是如何一步一步初始化好这个应用上下文的呢？

**首先明确下“初始化好了”这种状态是一种什么样的状态**。下面是我个人的理解：

上面的代码中返回的是`ConfigurableApplicationContext`接口，实现类是`AnnotationConfigApplicationContext`,它是`GenericApplicationContext`类的子类，我们以`GenericApplicationContext`这个类的状态来描述“初始化好了”这种状态。这种状态的标志之一就是其内部的beanbeanFactory属性已经装载好了各种Bean。beanbeanFactory内部有几个Map类型的字段（有些来自父类）：

```java
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

刚刚新建的ApplicationContext的singletonObjects是一个空Map，**所有的单例Bean**实例化之后都放入了singletonObjects这个Map中是BeanFactory初始化好的重要标志。那Spring是如何知道定义了那些Bean，然后实例化好之后放入singletonObjects的呢？这个就是要学习的Spring容器的初始化过程。

# 填充singletonObjects前先填充beanDefinitionMap

要实例化各种Bean放入singletonObjects，先要知道需要实例化那些Bean，这些Bean都是什么类型，依赖关系是怎么样的。Spring把需要知道的这些信息用接口`BeanDefinition`来描述，如果说Bean实例是一道菜，那么对应的`BeanDefinition`就是做出这道菜的菜谱。

那又从哪里知道需要准备那些`BeanDefinition`呢？这就有各种各样的手段了，比如古老的xml配置文件定义，比如基于注解的配置。现在主流的都使用SpringBoot框架，一切的启动点都在于**SpringBootApplication**这个注解。以上面的App.java为例子来说，Spring内部的逻辑应该是这样的：

1. SpringApplication.run的时候把App.class作为参数传递了，Spring就知道App.class上有SpringBootApplication注解，继而发现它有3个子注解：@SpringBootConfiguration，@EnableAutoConfiguration，@ComponentScan

2. @SpringBootConfiguration间接包含了@Configuration，@Configuration有间接包含了@Component，因此App.class本身就要实例化一个Bean，因此Spring会为App.class准备一个对应的BeanDefinition

3. @EnableAutoConfiguration是Spring自动配置相关的，这个注解会导致SpringBoot（不是SpringFramework）会读取当前类路径上所有jar包里的`META-INFO/spring.factories`文件中定义的自动配置类，从中读取需要的Bean信息。

4. @ComponentScan告诉了Spring需要扫描当前App.class所在包的所有子包。

5. 于是Spring就扫描子包所有类，发现Foo.class和Bar.class上有注解@Component，并且有依赖关系，于是为它们准备相应的BeanDefinition

6. 最后，beanDefinitionMap中就包含了（但不只）3个BeanDefinition，分别用于实例化App.class，Foo.class，Bar.class

7. 在执行ApplicationContext的`refresh`方法时就会根据BeanDefinition来实例化所有Bean了。

# 第一个BeanDefinition

对于SpringBoot应用来说，传递给SpringApplication.run的App.class叫做“PrimarySource”，它将会是“第一个”放入beanDefinitionMap的BeanDefinition

> 严格的说，不是绝对的第一个，而是我们自己写的类里的第一个，因为Spring把默认放一些它自己要用到的BeanDefinition先放到beanDefinitionMap里

在SpringApplication的run方法里，新建一个ApplicationContext实例的代码是下面代码中的**第14行**：`context = createApplicationContext();`，刚刚新建的ApplicationContext只有少数Spring自动带着的BeanDefinition，而第一个BeanDefinition就是在执行接下来的第17行`prepareContext`时产生的。

```java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);
        context = createApplicationContext();
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

`prepareContext`还有比较深的调用栈才到真正注册App.class类型的Bean的代码：

```txt
at org.springframework.context.annotation.AnnotatedBeanDefinitionReader.doRegisterBean(AnnotatedBeanDefinitionReader.java:253)
at org.springframework.context.annotation.AnnotatedBeanDefinitionReader.registerBean(AnnotatedBeanDefinitionReader.java:147)
at org.springframework.context.annotation.AnnotatedBeanDefinitionReader.register(AnnotatedBeanDefinitionReader.java:137)
at org.springframework.boot.BeanDefinitionLoader.load(BeanDefinitionLoader.java:157)
at org.springframework.boot.BeanDefinitionLoader.load(BeanDefinitionLoader.java:136)
at org.springframework.boot.BeanDefinitionLoader.load(BeanDefinitionLoader.java:128)
at org.springframework.boot.SpringApplication.load(SpringApplication.java:691)
at org.springframework.boot.SpringApplication.prepareContext(SpringApplication.java:392)
```

> 行号来自与SpringBoot 2.3.2.RELEASE的源码，版本不同，行号可能有点区别。

真正放入第一个Beandefinition的代码落脚到AnnotatedBeanDefinitionReader的doRegisterBean方法：

```java
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
        @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
        @Nullable BeanDefinitionCustomizer[] customizers) {

    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setInstanceSupplier(supplier);
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    if (customizers != null) {
        for (BeanDefinitionCustomizer customizer : customizers) {
            customizer.customize(abd);
        }
    }

    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

这个方法的第一个行，以App.class为参数new了一个AnnotatedGenericBeanDefinition实例， 中间经过一些操作设置一些属性后，最后一行代码`BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);`完成了把BeanDefinition放到beanDefinitionMap的动作 （this.registry就是beanDefinitionMap所属于的BeanFactory，由上层代码一层一层传下来）。这最后一行代码之后还有几层调用栈，最终是在`DefaultListableBeanFactory`的`registerBeanDefinition`方法中执行了`this.beanDefinitionMap.put(beanName, beanDefinition);`

# refreshContext

知道了单链表的头节点，就可以遍历完单链表所有节点；知道了树的根节点就可以遍历完树中的所有节点。有了App.class这个”种子“BeanDefinition （给SpringApplication传参的时候其实可以指定不止一个”种子“，但是至少一个），以此为出发点，就可以找到所有应该所有的BeanDefinition，进而实例化所有的Bean

SpringBoot中`refreshContext()`实际就是调用了`AbstractApplicationContext`类的`refresh()`方法。

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

这里面的逻辑很复杂，包含实现很多框架层面的机制，我们就只关注**何时填充好了beanDefinitionMap**，之后又**何时填充好了singletonObjects**

## 填充好beanDefinitionMap

填充好beanDefinitionMap发生在`invokeBeanFactoryPostProcessors(beanFactory);`这一行

这个阶段会用到Spring的`BeanFactoryPostProcessor`机制。

### BeanFactoryPostProcessor

BeanFactoryPostProcessor是Spring框架内部设计的一种机制，Spring在执行`invokeBeanFactoryPostProcessors`时，会执行注册在ApplicationContext上的BeanFactoryPostProcessor实现类的`void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;`接口方法。在这个方法里就有机会对已经在beanDefinitionMap中的BeanDefinition做出修改，或者删除beanDefinitionMap中的某些BeanDefinition，当然，也可以往beanDefinitionMap添加全新的BeanDefinition。postProcessBeanFactory方法里应该只根据需要修改beanDefinitionMap中的BeanDefinition，而不应该去修改singletonObjects这个Map中已经实例化好的Bean。

那到底ApplicationContext上注册了那些BeanFactoryPostProcessor呢？在这个最简单的非Web SpringBoot应用中有3个：

- SharedMetadataReaderFactoryContextInitializer类的静态内部类CachingMetadataReaderFactoryPostProcessor
- ConfigurationWarningsApplicationContextInitializer类的静态内部类ConfigurationWarningsPostProcessor
- ConfigFileApplicationListener类的静态内部类PropertySourceOrderingPostProcessor

另外除了主动注册到ApplicationContext上的BeanFactoryPostProcessor（会放在beanFactoryPostProcessors这个List中），Spring还会自动的检测beanDefinitionMap中已经定义的BeanDefinition所对应的类类型是不是BeanFactoryPostProcessor接口的实现类，如果是，也会在实例化Bean之前执行其postProcessBeanFactory方法。

**填充好beanDefinitionMap就发生在某一个BeanFactoryPostProcessor实现类的postProcessBeanFactory方法中**，前面说过了在postProcessBeanFactory方法中有机会往beanDefinitionMap加入新的BeanDefinition。

那么是那个BeanDefinition的实现类呢？在这简单的例子里都不是，既不是主动放在beanFactoryPostProcessors里的，也不是自动检测的。而是另外的一个机制：Spring容器会检查在beanDefinitionMap中的BeanDefinition，如果有某个类型是被@Configuration注解的的，那么Spring就会自动加入一个类型为的**ConfigurationClassPostProcessor**的BeanFactoryPostProcessor，执行其postProcessBeanDefinitionRegistry方法。这个方法里就会发现App.class上有自动扫描@ComponentScan注解，于是就会扫描类路径，发现更多的需要定义的BeanDefinition，然后放在beanDefinitionMap中。

> 至于Spring是如何发现App.class上有@SpringBootApplicaiton这个注解，又如何知道其子注解中有@Configuration和@ComponentScan，这是因为Spring实现了“注解发现机制”，参见另外一篇学习笔记：[Spring如何知道类上有什么注解，以及注解又有什么子注解](/java/spring中的注解)


## 填充好singletonObjects

所有的菜谱都准备好了，就要准备依此把菜全部做出来了。

beanDefinitionMap中已经准备好了所有的BeanDefinition，开始实例化Bean，并尝试解决**循环依赖**的问题。

一些SpringBoot框架内部定义的Bean，会在`onRefresh()`里实例化好然后放在了singletonObjects里。一般，我们自己用@Component注解，或者@Configuration加@Bean注解定义的Bean会在`finishBeanFactoryInitialization(beanFactory)`这一行完成实例化后放在singletonObjects里。

Spring会在一个for循环里实例化出Bean，类似如下：

```java
for (String beanName : beanNames) {
    // ...
    getBean(beanName);
}
```

关键就在这个`getBean(beanName)`，getBean调用了`doGetBean`方法，这个doGetBean相当复杂，是创建Bean实例的核心。

其实一个ApplicationContext初始化好后，当调用其getBean从里面取Bean时，也是调用的doGetBean方法，走的逻辑是一样的。只不过，ApplicationContext初始化好,直接从singletonObjects取，而初始化的时候，第一次到singletonObjects取时取到的是null，于是Spring就会执行`createBean`真正创建Bean后放在singletonObjects里。（从这里也可以看出Spring可以实现Bean的“懒实例化”）

在执行createBean之前，还会有个**插队**的机制，也就是注解@DependsOn的作用，可以定义A实例DependsOn一个B实例，这样在创建A的时候，发现B还没有实例化，就会先实例化B。

在执行createBean的时候会解决循环依赖的问题，比如正在创建A，但是从A的BeanDeinition中得知依赖于B（用了@Autowired），于是就会先创建B，那B又有依赖呢？继续**递归调用**doGetBean。在递归的时候，本来是在创建A，建里一般转而去创建B了，Spring会记录下这个中间状态，有一个Set\<String\>中存在正处在创建过程中的Bean的名字。至于该如何新建实例，使用静态工厂方法，还是反射调用默认构造函数，或者是有参数构造函数，这些早在生成BeanDefinition时就分析并决定好了。创建好单例Bean后就会放在singletonObjects了。

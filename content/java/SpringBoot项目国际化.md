---
title: "SpringBoot项目国际化"
date: 2021-05-16T10:00:00+08:00
draft: false
toc: true
---

# 定义国际化工具类

如果是在一个`Bean`（被Spring容器管理的类实例）里要输出国际化字符串，可以用`@Autowired`注入`MessageSource`实例，然后调用其`getMessage`方法，但是有时候要输出的国际化字符串的类不是一个Bean，个人喜欢做一个工具类提供`static`的方法来输出国际化字符串：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;

public class I18nUtils {

    private static MessageSource messageSource;

    public static String i18n(String code, Object... args) {
        return messageSource.getMessage(code, args, LocaleContextHolder.getLocale());
    }

    @Autowired
    public void setMessageSource(MessageSource messageSource) {
        I18nUtils.messageSource = messageSource;
    }

}
```

# 国际化自动配置

做一个spirng boot自动配置类来配置国际化，设置中文为默认的语言，同时初始化前面写的`I18nUtils`，国际化工具类。

```java
import java.util.Locale;

import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor;
import org.springframework.web.servlet.i18n.SessionLocaleResolver;

@Configuration
@ConditionalOnClass(name = "org.springframework.web.servlet.LocaleResolver")
public class I18nAutoConfiguration {

    public I18nAutoConfiguration() {
    }

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver sessionLocaleResolver = new SessionLocaleResolver();
        sessionLocaleResolver.setDefaultLocale(Locale.CHINA);
        return sessionLocaleResolver;
    }

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor localeChangeInterceptor = new LocaleChangeInterceptor();
        localeChangeInterceptor.setParamName("language");
        return localeChangeInterceptor;
    }

    @Bean
    public WebMvcConfigurer i18nWvcConfigure() {
        return new WebMvcConfigurer() {
            @Override
            public void addInterceptors(InterceptorRegistry registry) {
                registry.addInterceptor(localeChangeInterceptor());
            }
        };
    }

    @Bean
    public I18nUtils i18nUtils() {
        return new I18nUtils();
    }

}
```

这2个类可以做成一个`spring-boot-starter`，用`spring.factories`来配置。

# 语言文件

spring-boot配置国际化语言相关的配置的前缀是`spring.messages`，可以在`org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration`中看到：

```java
@Bean
@ConfigurationProperties(prefix = "spring.messages")
public MessageSourceProperties messageSourceProperties() {
    return new MessageSourceProperties();
}
```

最重要是配置`basename`，其默认值是"messages":

```java
public class MessageSourceProperties {

    private String basename = "messages";

    // ....

}
```

- 一个注意的点:

    按照默认配置，在类路径下（在maven项目里就直接在resources目录下）应该有messages_XXX.preperties等文件， 如: `messages.properties`,`messages_zh_CN.properties`等，IDE（IDEA）会自动把这些文件显示为一个`Resource Bundle messages`,看起来这些文件像是在一个叫`messages`的目录里，其实不是的！有一次，在项目中配置了`spring.messages.basename: i18n/messages`, 给人感觉是所有国际化文件要放在`i18n/messages`目录下，实际不是的，所有的文件是放在`i18n`目录下，后面的messages是`Bundle名`！。如果把文件放在resources目录下的路径实际是`i18n/messages`目录下就找不到国际化文件了。

- 多个模块

    如果，一个主jar一来了多个jar，这多个jar中都有国际化文件，如果他们它们的名字都一样，主jar只会加载到一个jar中的国际化文件，因此，每个jar最好加个前醉，比如：

    模块foo，配置成这样：
    ```
    i18n
    ├── fooMessages.properties
    └── fooMessages_zh_CN.properties
    ```
    
    模块bar，配置成这样：
    ```
    i18n
    ├── barMessages.properties
    └── barMessages_zh_CN.properties
    ```

    然后在主jar中定义：`spring.messages.basename: i18n/fooMessages,i18n/barMessages`,这样2个jar包中的国际化文件都能加载到。

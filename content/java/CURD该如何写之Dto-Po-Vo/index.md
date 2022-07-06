---
title: "CURD该如何写之Dto-Po-Vo"
date: 2022-07-06T17:49:34+08:00
draft: false
toc: false
tags: []
---

时常看到一些CRUD代码在Controller里大段大段的逻辑，Dto，Po，Vo也用的不太对，这里记录下我自己对与CRUD代码应该怎么写的一些见解。

> 以下对CRUD的写法代表的是个人的体会，或者说是习惯，并不是标准答案，是具有个人风格的，不同的人可能完全不这样写，仅供参考。

CRUD的代码应该遵循如下原则：

1. Controller层只负责接受参数（**DTO**），校验参数（很复杂的话，校验也可以放到Service层去做），以及返回响应（**Vo**）。

    Controller里的代码应该是非常的精简，一个函数的代码应该不超过20行，简单的甚至不超过10行。

    Controller从客户端接进来的是**Dto**，给到Service层的也是**Dto**，返回给客户端的是**Vo**

2. Service写主要逻辑

    Controller接进来的是**Dto**,给到Dao层的是**Po**，返回给Controller的也是**Po**

3. Dao层也负责和数据库交互，不要有任何的业务上的逻辑，逻辑放在Service层。

    create的时候，从Service传过来的各种属性已经设置好的**Po**，直接save即可，没有逻辑；query的时候，从service传过来的是直接
    能用的过滤条件参数，返回给Service层的是**Po**

4. Dto，Po，Vo这3种数据结构很可能很多的属性重叠，甚至完全一模一样，但是建议还是建立3个完全**没有继承或包含关系**的简单类，虽然一眼看上去冗余。


![](./img1.svg)

> Po可以有多种Vo，比如在Pc端和移动端展示不一样的数据结构，或者做AB-Test等，或者其他可能的场景，背后的思想就是原始数据一样的情况下可以有不同的展现形式。


上图的关系用代码来表示出来：

- Dto

Dto.java
```java
public interface Dto {
}
```

- Po

Po.java
```java
public interface Po<D extends Dto> {

    Po<D> fromDto(D dtoInstance);

}
```

- Vo

Vo.java
```java
public interface Vo<P extends Po<? extends Dto>> {

    Vo<P> fromPo(P poInstance);

}
```


下面是一个Person类的CURD演示代码：

> 为减少演示代码量，Pojo类属性均设置为public

- Dto

PersonDto.java
```java
public class PersonDto implements Dto {

    public String firstName;

    public String lastName;

    public int age;

}
```
- Po

PersonPo.java
```java
import java.util.UUID;

public class PersonPo implements Po<PersonDto> {

    public String id;

    public String firstName;

    public String lastName;

    public int age;

    @Override
    public PersonPo fromDto(PersonDto dtoInstance) {
        this.id = UUID.randomUUID().toString();
        this.firstName = dtoInstance.firstName;
        this.lastName = dtoInstance.lastName;
        this.age = dtoInstance.age;
        return this;
    }

}
```

- Vo

PersonVo.java
```java
public class PersonVo implements Vo<PersonPo> {

    public String id;

    public String firstName;

    public String lastName;

    public int age;

    public String getFirstName() {
        return firstName + lastName;
    }

    @Override
    public PersonVo fromPo(PersonPo personPo) {
        this.id = personPo.id;
        this.firstName = personPo.firstName;
        this.lastName = personPo.lastName;
        this.age = personPo.age;
        return this;
    }

}
```

- Dao

PersonDao.java
```java
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class PersonDao {

    Map<String, PersonPo> database = new ConcurrentHashMap<>();

    public void create(PersonPo personPo) {
        database.put(personPo.id, personPo);
    }

    public List<PersonPo> list() {
        return new ArrayList<>(database.values());
    }

}
```

- Service

PersonService.java
```java
import java.util.List;

public class PersonService {

    private final PersonDao personDao = new PersonDao();

    public void create(PersonDto personDto) {
        personDao.create(new PersonPo().fromDto(personDto));
    }

    public List<PersonPo> list() {
        return personDao.list();
    }

}
```

- Controller

PersonController.java
```java
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.util.List;

public class PersonController {

    private final PersonService personService = new PersonService();

    public void create(@ResponseBody PersonDto personDto) {
        // 自己校验，或者用框架AOP实现对Dto的合法性的校验
        personService.create(personDto);
    }

    public List<PersonVo> list() {
        final List<PersonPo> poList = personService.list();
        return poList.stream().map(personPo -> new PersonVo().fromPo(personPo)).toList();
    }

    @Target(ElementType.PARAMETER)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface ResponseBody {
    }

}
```
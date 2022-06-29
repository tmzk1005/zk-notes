---
title: "简单对象基于AVRO实现序列化和反序列化"
date: 2021-10-27T10:22:39+08:00
draft: true
toc: false
---

### mvn avro依赖

```xml
<dependency>
    <groupId>org.apache.avro</groupId>
    <artifactId>avro</artifactId>
    <version>1.10.2</version>
</dependency>
```

### 接口定义

AvroSerializer.java

```java
public interface AvroSerializer<T> {

    /**
     * serialize object to byte array
     *
     * @param data Object to serializer
     * @return byte array
     */
    byte[] serialize(T data);

}
```

AvroDeserializer.java

```java
public interface AvroDeserializer<T> {

    /**
     * deserializer byte array to object
     *
     * @param bytes byte array to deserializer
     * @return object
     */
    T deserialize(byte[] bytes);

}
```

### 抽象实现

AbstractAvroSerializer.java

```java
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.lang.reflect.ParameterizedType;

import org.apache.avro.io.BinaryEncoder;
import org.apache.avro.io.DatumWriter;
import org.apache.avro.io.EncoderFactory;
import org.apache.avro.reflect.ReflectDatumWriter;

public abstract class AbstractAvroSerializer<T> implements AvroSerializer<T> {

    public static final int DEFAULT_MAX_BYTE_SIZE = 4 * 1024 * 1024;

    protected final DatumWriter<T> datumWriter;

    protected final ByteArrayOutputStream byteArrayOutputStream;

    protected final BinaryEncoder binaryEncoder;

    @SuppressWarnings("unchecked")
    protected AbstractAvroSerializer(int maxSerializerByteSize) {
        Class<T> actualTypeArgument = (Class<T>) ((ParameterizedType) getClass().getGenericSuperclass()).getActualTypeArguments()[0];
        datumWriter = new ReflectDatumWriter<>(actualTypeArgument);
        byteArrayOutputStream = new ByteArrayOutputStream(maxSerializerByteSize);
        binaryEncoder = EncoderFactory.get().binaryEncoder(byteArrayOutputStream, null);
    }

    protected AbstractAvroSerializer() {
        this(DEFAULT_MAX_BYTE_SIZE);
    }

    public byte[] serialize(T data) {
        byteArrayOutputStream.reset();
        try {
            datumWriter.write(data, binaryEncoder);
            binaryEncoder.flush();
        } catch (IOException ioE) {
            return new byte[0];
        }
        return byteArrayOutputStream.toByteArray();
    }

}
```

AbstractAvroDeserializer.java

```java
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.ParameterizedType;
import java.util.Objects;

import org.apache.avro.io.BinaryDecoder;
import org.apache.avro.io.DatumReader;
import org.apache.avro.io.DecoderFactory;
import org.apache.avro.reflect.ReflectDatumReader;

public abstract class AbstractAvroDeserializer<T> implements AvroDeserializer<T> {

    protected final DatumReader<T> datumReader;

    protected final DecoderFactory decoderFactory;

    protected final BinaryDecoder binaryDecoder;

    private final Constructor<T> constructor;

    @SuppressWarnings("unchecked")
    protected AbstractAvroDeserializer() {
        Class<T> actualTypeArgument = (Class<T>) ((ParameterizedType) getClass().getGenericSuperclass()).getActualTypeArguments()[0];
        try {
            constructor = actualTypeArgument.getDeclaredConstructor();
        } catch (NoSuchMethodException exception) {
            throw new RuntimeException(exception);
        }
        datumReader = new ReflectDatumReader<>(actualTypeArgument);
        decoderFactory = DecoderFactory.get();
        binaryDecoder = decoderFactory.binaryDecoder(new byte[0], null);
    }

    @Override
    public T deserialize(byte[] bytes) {
        decoderFactory.binaryDecoder(bytes, binaryDecoder);
        T instance = newInstance();
        if (Objects.isNull(instance)) {
            return null;
        }
        try {
            return datumReader.read(newInstance(), binaryDecoder);
        } catch (IOException ioException) {
            return null;
        }
    }

    protected T newInstance() {
        // 这里使用反射，子类可以覆盖此实现提升性能
        try {
            return constructor.newInstance();
        } catch (InstantiationException | IllegalAccessException | InvocationTargetException exception) {
            return null;
        }
    }

}
```

### 具体实现和使用例子测试

AbstractAvroSerializerAndDeserializerTest.java

```java
import org.junit.Assert;
import org.junit.Test;

public class AbstractAvroSerializerAndDeserializerTest {

    @Test
    public void testSerializerAndDeserializer() {
        // init person instance
        Person person = new Person();
        person.name = "alice";
        person.age = 18;
        PersonAvroSerializer personAvroSerializer = new PersonAvroSerializer();

        // serialize
        byte[] bytes = personAvroSerializer.serialize(person);

        // test deserializer case 1
        PersonAvroDeserializer personAvroDeserializer = new PersonAvroDeserializer();
        Person deserializePerson = personAvroDeserializer.deserialize(bytes);

        Assert.assertEquals(person.name, deserializePerson.name);
        Assert.assertEquals(person.age, deserializePerson.age);
        Assert.assertNotEquals(person.hashCode(), deserializePerson.hashCode());

        // test deserializer case 2
        PersonAvroDeserializer2 personAvroDeserializer2 = new PersonAvroDeserializer2();
        Person deserializePerson2 = personAvroDeserializer2.deserialize(bytes);

        Assert.assertEquals(person.name, deserializePerson2.name);
        Assert.assertEquals(person.age, deserializePerson2.age);
        Assert.assertNotEquals(person.hashCode(), deserializePerson2.hashCode());
    }

    static class Person {
        private String name;
        private int age;
    }

    static class PersonAvroSerializer extends AbstractAvroSerializer<Person> {
    }

    static class PersonAvroDeserializer extends AbstractAvroDeserializer<Person> {
    }

    static class PersonAvroDeserializer2 extends AbstractAvroDeserializer<Person> {

        @Override
        protected Person newInstance() {
            return new Person();
        }

    }

}
```
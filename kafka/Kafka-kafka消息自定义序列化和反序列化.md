# Kafka-kafka消息自定义序列化和反序列化

[TOC]

## 简介：

​	kafka内部将消息转化成byte[]，通过rpc在网络间传输，实际上归功于kafka底层实现对消息的序列化和反序列化机制。kafka自己实现了int，long，string等基础数据类型的序列化和反序列化。



## kafka生产者和消费者配置序列化方式：

###### Producer配置：

```java
//配置producer生产的消息key和value序列话方式都为StringSerializer(根据实际情况而定)
kafkaProerties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,StringSerializer.class);
       kafkaProerties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,StringSerializer.class);
```

###### Consumer配置：

```java
//配置consumer生产的消息key和value序列话方式都为StringSerializer(根据实际情况而定)   
consumerProperties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,StringDeserializer.class);
       consumerProperties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,StringDeserializer.class);
```



## 自定义序列化方式

有的时候需要发送消息的数据类型是自定义对象，此时kafka没有相应数据类型的内置序列化方式。此时我们需要自定义序列化和反序列化方式。

##### 如何自定义序列化

自定义序列化类只需要实现kafka内部接口org.apache.kafka.common.serialization.Serializer和org.apache.kafka.common.serialization.Deserializer接口，重写里面的方法即可。

```java
public class person{
    
    private String name;
    private int age;
    
    public ... get(){}
    
    public void set(){}
    
}
```



```java
//自定义person序列化类
public class PersonSerializer implements Serializer<Person> {
    @Override
    public void configure(Map<String, ?> map, boolean b) {

    }

    @Override
    public byte[] serialize(String topic, Person person) {
        byte[] bytes = null;
        try {
            ByteArrayOutputStream byteArray = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(byteArray);
            oos.writeObject(person);
            oos.flush();
            bytes = byteArray.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return bytes;
    }

    @Override
    public void close() {

    }
}
```

```java
//自定义person反序列化类
public class PersonDeserializer implements Deserializer<Person> {
    @Override
    public void configure(Map<String, ?> map, boolean b) {

    }

    @Override
    public Person deserialize(String s, byte[] bytes) {
        Person person = null;
        try {
            ByteArrayInputStream in  = new ByteArrayInputStream(bytes);
            ObjectInputStream ois = new ObjectInputStream(in);
            person = (Person) ois.readObject();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return person;
    }

    @Override
    public void close() {

    }
}

```

##### 配置自定义序列化类

###### producer

```JAVA
//配置一个key 为string类型，value为person类型的producer
kafkaProerties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,StringSerializer.class);
       kafkaProerties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,PersonSerializer.class);
```



###### consumer

```JAVA
//配置一个key 为string类型，value为person类型的consumer
consumerProperties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,StringDeserializer.class);
       consumerProperties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,StringDeserializer.class);
```





**Over。。。。**
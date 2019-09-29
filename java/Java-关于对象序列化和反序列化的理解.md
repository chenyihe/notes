# Java-关于对象序列化和反序列化的理解

###### 

- #### 什么时候会用到序列化

  有的时候需要将某个对象持久化本地保存起来，但是如何保存一个自定义的对象，这个时候就需要用到序列化，而将我们需要持久化的对象变成一连串的字节描述过程就是序列化，这时候对象的字节码便可以保存在本地。将保存的字节码反向生成对象的过程则是反序列化，发序列化是序列化的逆向工程

- #### 如何实现序列化

  需要序列化的类可以直接实现**Serializable**接口即可。

  ```java
  public class person implements Serializable{
      
      private String name;
      private int age;  
      
      public Person(String name,Int age){
          this.name = name;
          this.age = age;
      }
  }
  ```

  ```java
  public class SerializableTest{
      
      public void static serializablePerson(){
          Person person = new Person("Cyril",18);
          ObjectOutputStream oos = new ObjectOutputStrean(new FileOutputStream(new File("person.txt")))
          oos.writeObject(person);
          System.out.println("序列化成功");
          oos.close();
      }
      
      public Person static derserializaPerson(){
          ObjectInputStream ois = new ObjectInputStream(new FileInputStream(new File("person.txt")))
          Person person = (Person) ois.readObject();
          System.out.println("反序列化成功");
          ois.close();
      }
      
      public static void main(String[] args){
          serializablePerson();
          Person person = derserializaPerson();
      }
      
  }
  ```

  

如果某个对象有不需要序列化的属性，需要用[**transient**]() 关键字修饰该属性。



```java
public class Person implements Serializable{
    transient private int hight;  //身高字段不会被序列化
    private String name;
    private int age;
}
```



- #### 注意事项

  1、序列化时，只对对象的状态进行保存，而不管对象的方法；

  2、当一个父类实现序列化，子类自动实现序列化，不需要显式实现Serializable接口；

  3、当一个对象的实例变量引用其他对象，序列化该对象时也把引用对象进行序列化；

  4、并非所有的对象都可以序列化，至于为什么不可以，有很多原因了，比如：

  安全方面的原因，比如一个对象拥有private，public等field，对于一个要传输的对象，比如写到文件，或者进行RMI传输等等，在序列化进行传输的过程中，这个对象的private等域是不受保护的；

  资源分配方面的原因，比如socket，thread类，如果可以序列化，进行传输或者保存，也无法对他们进行重新的资源分配，而且，也是没有必要这样实现；

  5、声明为static和transient类型的成员数据不能被序列化。因为static代表类的状态，transient代表对象的临时数据。

  6、序列化运行时使用一个称为 serialVersionUID 的版本号与每个可序列化类相关联，该序列号在反序列化过程中用于验证序列化对象的发送者和接收者是否为该对象加载了与序列化兼容的类。为它赋予明确的值。显式地定义serialVersionUID有两种用途：

  在某些场合，希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有相同的serialVersionUID；

  在某些场合，不希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有不同的serialVersionUID。

  7、Java有很多基础类已经实现了serializable接口，比如String,Vector等。但是也有一些没有实现serializable接口的；

  8、如果一个对象的成员变量是一个对象，那么这个对象的数据成员也会被保存！这是能用序列化解决深拷贝的重要原因；
  原文链接：https://blog.csdn.net/xlgen157387/article/details/79840134




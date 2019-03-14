title: 补充阅读4-Class文件中的字段表集合--field字段在class文件中是怎样组织的
tag: JVM
---
继续讲class文件中的字段表集合。
<!-- more -->

## 1. 字段表集合概述

字段表集合是指由若干个字段表（field_info）组成的集合。对于在类中定义的若干个字段，经过JVM编译成class文件后，会将相应的字段信息组织到一个叫做字段表集合的结构中，字段表集合是一个类数组结构，如下图所示：

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B581)

注意：这里所讲的字段是指在类中定义的静态或者非静态的变量，而不是在类中的方法内定义的变量。请注意区别。
比如，如果某个类中定义了5个字段，那么，JVM在编译此类的时候，会生成5个字段表（field_info）信息,然后将字段表集合中的字段计数器的值设置成5，将5个字段表信息依次放置到字段计数器的后面。

## 2. 字段表集合在class文件中的位置

字段表集合紧跟在class文件的接口索引集合结构的后面，如下图所示：

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B582)


## 3.  Java中的一个Field字段应该包含那些信息？------字段表field_info结构体的定义   

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B583)

针对上述的字段表示，JVM虚拟机规范规定了field_info结构体来描述字段，其表示信息如下：

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B584)

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B585)

下面我将一一讲解FIeld_info的组成元素：访问标志（access_flags）、名称索引（name_index）、描述索引（descriptor_index）、属性表集合

## 4. field字段的访问标志

如上图所示定义的field_info结构体，field字段的访问标志(access_flags)占有两个字节，它能够表述的信息如下所示：

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B586)

举例：如果我们在某个类中有定义field域：private static String str;，那么在访问标志上，第15位ACC_PRIVATE和第13位ACC_STATIC标志位都应该为1。field域str的访问标志信息应该是如下所示：

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B587)


## 5. 字段的数据类型表示和字段名称表示

class文件对数据类型的表示如下图所示：

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B588)

field字段名称，我们定义了一个形如private static String str的field字段，其中"str"就是这个字段的名称。
class文件将字段名称和field字段的数据类型表示作为字符串存储在常量池中。在field_info结构体中，紧接着访问标志的，就是字段名称索引和字段描述符索引，它们分别占有两个字节，其内部存储的是指向了常量池中的某个常量池项的索引，对应的常量池项中存储的字符串，分别表示该字段的名称和字段描述符。

## 6. 属性表集合-----静态field字段的初始化

在定义field字段的过程中，我们有时候会很自然地对field字段直接赋值，如下所示：


```java
public static final int MAX=100;  
public  int count=0;  
```
对于虚拟机而言，上述的两个field字段赋值的时机是不同的：

> 对于非静态（即无static修饰）的field字段的赋值将会出现在实例构造方法<init>()中

> 对于静态的field字段，有两个选择：1、在静态构造方法<cinit>()中进行；2 、使用ConstantValue属性进行赋值

Sun javac编译器对于静态field字段的初始化赋值策略：

> 如果使用final和static同时修饰一个field字段，并且这个字段是基本类型或者String类型的，那么编译器在编译这个字段的时候，会在对应的field_info结构体中增加一个ConstantValue类型的结构体，在赋值的时候使用这个ConstantValue进行赋值；

> 如果该field字段并没有被final修饰，或者不是基本类型或者String类型，那么将在类构造方法<cinit>()中赋值。

对于上述的public static final init MAX=100：

> javac编译器在编译此field字段构建field_info结构体时，除了访问标志、名称索引、描述符索引外，会增加一个ConstantValue类型的属性表。

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B589)

## 7. 实例解析

定义如下一个简单的Simple类，然后通过查看Simple.class文件内容并结合javap -v Simple 生成的常量池内容，分析str field字段的结构：


```java
public class Simple {  
  
    private  transient static final String str ="This is a test";  
}  
```

![image](http://javajvm.oursnail.cn/field%E5%AD%97%E6%AE%B5810)

> 1. 字段计数器中的值为0x0001,表示这个类就定义了一个field字段

> 2. 字段的访问标志是0x009A,二进制是00000000 10011010，即第9、12、13、15位标志位为1，这个字段的标志符有：ACC_TRANSIENT、ACC_FINAL、ACC_STATIC、ACC_PRIVATE;

> 3. 名称索引中的值为0x0005,指向了常量池中的第5项，为“str”,表明这个field字段的名称是str；

> 4. 描述索引中的值为0x0006,指向了常量池中的第6项，为"Ljava/lang/String;"，表明这个field字段的数据类型是java.lang.String类型；

> 5.属性表计数器中的值为0x0001,表明field_info还有一个属性表；

> 6.属性表名称索引中的值为0x0007,指向常量池中的第7项，为“ConstantValue”,表明这个属性表的名称是ConstantValue，即属性表的类型是ConstantValue类型的；

> 7.属性长度中的值为0x0002，因为此属性表是ConstantValue类型，它的值固定为2；

> 8.常量值索引 中的值为0x0008,指向了常量池中的第8项，为CONSTANT_String_info类型的项，表示“This is a test” 的常量。在对此field赋值时，会使用此常量对field赋值。
title: 彻底理解java反射机制
tag: java基础
---
反射机制这一块也是面试经常会被问到的，我从反射的基本概念到反射的一些面试题出发，好好理一理反射的知识。
<!-- more -->

## 1. 什么是反射

标准定义：JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性；这种动态获取信息以及动态调用方法的功能成为反射机制。

注意几个关键字：运行状态中，动态获取。

## 2. Class对象和实例对象

想要理解反射首先需要知道`Class`这个类，它的全称是`java.lang.Class`类。java是面向对象的语言，讲究万物皆对象，即使强大到一个类，它依然是另一个类（`Class`类）的对象，换句话说，普通类是`Class`类的对象，即`Class`是所有类的类（`There is a class named Class`）。


我们知道java世界是运行在JVM之上的，我们编写的类代码，在经过编译器编译之后，会为每个类生成对应的`.class`文件，这个就是JVM可以加载执行的字节码。

运行时期间，当我们需要实例化任何一个类时，JVM会首先尝试看看在内存中是否有这个类，如果有，那么会直接创建类实例；如果没有，那么就会根据类名去加载这个类，当加载一个类，或者当加载器(`class loader`)的`defineClass()`被JVM调用，便会为这个类产生一个`Class`对象（一个`Class`类的实例），用来表达这个类，该类的所有实例都共同拥有着这个`Class`对象，而且是唯一的。

也就是说，加载`.class`文件之后会生成一个对应的`Class`对象。下面说说如何获取这个`Class`对象。

## 3. 取得Class对象的三种方式

我们假设有这么一个类叫`MyClass`：

```java
public class MyClass {  }
```

* 第一种方式：通过“类名.class”的方式取得


```java
Class classInstance= MyClass.class;
```

例如：

```
Class clazz = Car.class;
Class cls1 = int.class;
Class cls2 = String.class;
```


* 第二种方式：通过类创建的实例对象的`getClass`方法取得


```java
MyClass myClass = new MyClass();
Class classInstance = myClass.getClass();
```


* 第三种方式：通过`Class`类的静态方法`forName`方法取得（参数是带包名的完整的类名）

```java
try {
  Class classInstance = Class.forName("mypackage.MyClass");
} catch (ClassNotFoundException e) {
  e.printStackTrace();
}
```


上面三种方法取得的对象都是相同的，所以效果上等价。

`classInstance`是类类型，通过类类型可以得到一个类的属性和方法等参数，这是反射的基础。

## 4. 利用反射API全面分析类的信息——方法，成员变量，构造器

反射的一大作用是用于分析类的结构，或者说用于分析和这个类有关的所有信息。而这些信息就是类的基本的组成： 方法，成员变量和构造器。


在java种万物皆对象，一个类中的方法，成员变量和构造器也分别对应着一个对象
 
1. 每个方法都对应有一个保存和该方法有关信息的**Method对象**， 这个对象所属的类是`java.lang.reflect.Method`;
2. 每个成员变量都对应有一个保存和该变量有关信息的**Field对象**，这个对象所属的类是 `java.lang.reflect.Field`
3. 每个构造器都对应有一个保存和该构造器有关信息的**Constructor对象**，这个对象所属的类是`java.lang.reflect.Constructor`


假设c是一个类的Class对象：
 
* 通过 `c.getDeclaredMethods()`可取得这个类中所有声明方法对应的`Method`对象组成的数组
* 通过 `c.getDeclaredFields()`可取得这个类中所有声明的成员变量对应的`Field`对象组成的数组
* 通过 `c.getConstructors()`; 可取得这个类中所有构造函数所对应的`Constructor`对象所组成的数组


```java
Method [] methods = c.getDeclaredMethods(); // 获取方法对象列表
 
Field [] fields = c.getDeclaredFields();   // 获取成员变量对象列表

Constructor [] constructors = c.getConstructors();  // 获取构造函数对象列表

xxx.getName()就可以打印出对应的名字了。
```
## 5. 更多的反射api

##### getMethods和getDeclaredMethods方法

* `getMethods`取得的`method`对应的方法**包括从父类中继承的那一部分**，而
* `getDeclaredMethods`取得的`method`对应的方法**不包括从父类中继承的那一部分**


一个普通的类，他们的基类都是`Object`，那么如果用`getMethods`，遍历得到的结果，会发现`Object`中的基础方法名都会被打印出来。

诸如`wait()`,`equals()`,`toString()`,`getClass()`,
`notify()`,`notifyAll()`,`hashCode()`等等。

##### 通过method.getReturnType()获取方法返回值对应的Class对象

```java
Class returnClass = method.getReturnType(); // 获取方法返回值对应的Class对象
String returnName = returnClass.getName();  //获取返回值所属类的类名——也即返回值类型
```

##### 通过method.getParameterTypes()获取方法各参数的Class对象组成的数组


```java
Class [] paramsClasses = method.getParameterTypes();
for (Class pc: paramsClasses) {
    String paramStr = pc.getName(); // 获取当前参数类型
    paramsStr+=paramStr + "  ";
}
```
##### 获取成员变量类型对应的的Class对象


```java
Field field = c.getDeclaredField("name");  // 取得名称为name的field对象
field.setAccessible(true); // 这一步很重要！！！设置为true才能访问私有成员变量name的值！
String nameValue = (String) field.get(obj); // 获取obj中name成员变量的值
```

##### 通过getType方法读取成员变量类型的Class对象


```java
Field field = class1.getDeclaredField(number");
System.out.print(field.getType().getName());
```

因为java权限的原因，直接读取私有成员变量的值是非法的（加了`field.setAccessible(true)`后就可以了），但仍可以直接读取私有成员变量的类型

##### 利用反射API分析类中构造器信息


```java
public class MyClass {
  public MyClass(int a, String str){}
}
```


```java
public static void printContructorsMessage (Object obj) {
Class c = obj.getClass();  // 取得obj所属类对应的Class对象
Constructor [] constructors = c.getDeclaredConstructors();
for (Constructor constructor : constructors) {
  Class [] paramsClasses =  constructor.getParameterTypes();
  String paramsStr = "";
  for (Class pc : paramsClasses) {
    String paramStr = pc.getName();
    paramsStr+=paramStr + "  ";
  }
  System.out.println("构造函数的所有参数的类型列表：" + paramsStr);
}
}
```
运行结果：

```java
构造函数的所有参数的类型列表：int  java.lang.String 
```

## 6. 利用反射动态加载类，并用该类创建实例对象

我们用普通的方式使用一个类的时候，类是静态加载的
，**而使用Class.forName("XXX")这种方式，则属于动态加载一个类**

静态加载的类在编译的时候就能确定该类是否存在，但动态加载一个类的时候却无法在编译阶段确定是否存在该类，而是在运行时候才能够确定是否有这个类，所以要捕捉可能发生的异常.

Class对象有一个`newInstance`方法，我们可以用它来创建实例对象

```java
Class classInstance = Class.forName("mypackage.MyClass");
MyClass myClass = (MyClass) classInstance.newInstance();
```

## 7. 总结
 
* 反射为我们提供了全面的分析类信息的能力，例如类的方法，成员变量和构造器等的相关信息，反射能够让我们很方便的获取这些信息， 而实现这个获取过程的关键是取得类的`Class`对象，然后根据`Class`对象取得相应的`Method`对象，`Field`对象和`Constructor`对象，再分别根据各自的API取得信息。
* 反射还为我们提供动态加载类的能力
* API中`getDeclaredXXX`和`getXXX`的区别在于前者只获取本类声明的XXX（如成员变量或方法），而不获取超类中继承的XXX， 后者都可以获取
* API中， `getXXXs`（注意后面的s）返回的是一个数组， 而对应的 `getXXX`（"键"）按键获取一个值（这个时候因为可能报已检查异常所以要用try*catch语句包裹）
* 私有成员变量是不能直接获取到值的！因为java本身的保护机制，允许你取得私有成员变量的类型，但是不允许直接获取值，所以要对对应的`field`对象调用`field.setAccessible(true)` 放开权限

## 8. 面试

#### 什么是反射

反射是一种能够在程序运行时动态访问、修改某个类中任意属性（状态）和方法（行为）的机制


#### 反射到底有什么具体的用处

* 操作因访问权限限制的属性和方法；
* 实现自定义注解；
* 动态加载第三方jar包，解决android开发中方法数不能超过65536个的问题；
* 按需加载类，节省编译和初始化APK的时间；

#### 反射的原理是什么

当我们编写完一个Java项目之后，每个java文件都会被编译成一个.class文件，这些Class对象承载了这个类的所有信息，包括父类、接口、构造函数、方法、属性等，这些class文件在程序运行时会被ClassLoader加载到虚拟机中。当一个类被加载以后，Java虚拟机就会在内存中自动产生一个Class对象。我们通过new的形式创建对象实际上就是通过这些Class来创建，只是这个过程对于我们是透明的而已。

反射的工作原理就是借助`Class.java`、`Constructor.java`、
`Method.java`、`Field.java`这四个类在程序运行时动态访问和修改任何类的行为和状态。

#### 如何获取Class对象

* `Class`的`forName()`方法的返回值就是`Class`类型，也就是动态导入类的`Class`对象的引用
```java
public static Class<?> forName(String className) throws ClassNotFoundException
```
* 每个类都会有一个名称为`Class`的静态属性，通过它也是可以获取到`Class`对象

```java
Class<Student> clazz = Student.class;
```
* `Object`类中有一个名为`getClass`的成员方法，它返回的是对象的运行时类的`Class`对象。因为`Object`类是所有类的父类，所以，所有的对象都可以使用该方法得到它运行时类的`Class`对象

```java
Student stu = new Student();
Class<Student> clazz = stu.getClass();
```



#### 反射的特点

> 优点

* 灵活、自由度高：不受类的访问权限限制，想对类做啥就做啥

> 缺点

* 性能问题

通过反射访问、修改类的属性和方法时会远慢于直接操作，但性能问题的严重程度取决于在程序中是如何使用反射的。如果使用得很少，不是很频繁，性能将不会是什么问题；

* 安全性问题

反射可以随意访问和修改类的所有状态和行为，破坏了类的封装性，如果不熟悉被反射类的实现原理，随意修改可能导致潜在的逻辑问题；


#### 如何提高反射性能


java应用反射的时候，性能往往是java程序员担心的地方，那么在大量运用反射的时候，性能的微弱提升，对这个系统而言都是如旱地逢甘霖。

* `setAccessible(true)`,可以防止安全性检查（做这个很费时）
* 做缓存，把要经常访问的元数据信息放入内存中，`class.forName` 太耗时
* `getMethods()` 等方法尽量少用，尽量调用`getMethod(name)`指定方法的名称，减少遍历次数


#### java面试中面试官让你讲讲反射，应该从何讲起？

先讲反射机制，反射就是程序运行期间JVM会对任意一个类洞悉它的属性和方法，对任意一个对象都能够访问它的属性和方法。依靠此机制，可以动态的创建一个类的对象和调用对象的方法。

其次就是反射相关的API，只讲一些常用的，比如获取一个`Class`对象。`Class.forName(完整类名)`。通过`Class`对象获取类的构造方法，`class.getConstructor`。根据`Class`对象获取类的方法，`getMethod`和`getMethods`。使用`Class`对象创建一个对象，`class.newInstance`等。

最后可以说一下反射的优点和缺点，优点就是增加灵活性，可以在运行时动态获取对象实例。缺点是反射的效率很低，而且会破坏封装，通过反射可以访问类的私有方法，不安全。


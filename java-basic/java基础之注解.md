title: java基础之注解
tag: java基础
---

注解是一系列元数据，它提供数据用来解释程序代码，注释是给人看的，注解是给编译器看的，因此注解只在编译器有效。注解的实现原理涉及反射和动态代理，关于反射已经在前面说过，动态代理还没说，留在下一节。

<!--more-->

## 注解语法

相信有不少的人员会认为注解的地位不高。其实同 `classs` 和 `interface` 一样，注解也属于一种类型。它是在 Java SE 5.0 版本中开始引入的概念。

## 注解的定义

注解通过 `@interface` 关键字进行定义。


```java
public @interface TestAnnotation {}
```
你可以简单理解为创建了一张名字为 `TestAnnotation` 的标签。


## 注解的使用

上面创建了一个注解，那么注解的的使用方法是什么呢。

```java
@TestAnnotation
public class Test {}
```
不过，要想注解能够正常工作，还需要介绍一下一个新的概念那就是元注解。


## 什么是元注解

元注解是可以注解到注解上的注解，或者说元注解是一种基本注解，但是它能够应用到其它的注解上面。

元标签有 `@Retention`、`@Documented`、`@Target`、`@Inherited`、`@Repeatable` 5 种。

> @Retention

`Retention` 的英文意为保留期的意思。当 `@Retention` 应用到一个注解上的时候，它解释说明了这个**注解的的存活时间**。

- `RetentionPolicy.SOURCE` 注解只在**源码阶段**保留，在编译器进行**编译时它将被丢弃**忽视。
- `RetentionPolicy.CLASS` 注解**只被保留到编译进行**的时候，它并**不会被加载到 JVM** 中。 
- `RetentionPolicy.RUNTIME` 注解可以**保留到程序运行**的时候，它**会被加载进入到 JVM** 中，所以在程序运行时可以获取到它们。


```java
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {}
```


> @Documented

顾名思义，这个元注解肯定是和文档有关。它的作用是能够将注解中的元素包含到 `Javadoc` 中去。


> @Target

`Target` 是目标的意思，`@Target` 指定了注解运用的地方。
- `ElementType.ANNOTATION_TYPE` 可以给一个注解进行注解
- `ElementType.CONSTRUCTOR` 可以给构造方法进行注解
- `ElementType.FIELD` 可以给属性进行注解
- `ElementType.LOCAL_VARIABLE` 可以给局部变量进行注解
- `ElementType.METHOD` 可以给方法进行注解
- `ElementType.PACKAGE` 可以给一个包进行注解
- `ElementType.PARAMETER` 可以给一个方法内的参数进行注解
- `ElementType.TYPE` 可以给一个类型进行注解，比如类、接口、枚举

> @Inherited

`Inherited` 是继承的意思，但是它并不是说注解本身可以继承，而是说如果一个**超类被 `@Inherited` 注解过的注解进行注解**的话，那么如果它的子类没有被任何注解应用的话，那么这个子类就继承了超类的注解。


```java
//定义一个被@Inherited注解的注解@Test
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@interface Test {}

//父类被@Test注解，即上面说的被@Inherited 注解过的注解进行注解
@Test
public class A {}

//那么class B也拥有@Test注解
public class B extends A {}
```


> @Repeatable

`Repeatable` 自然是可重复的意思。`@Repeatable` 是 Java 1.8 才加进来的，所以算是一个新的特性。

什么样的注解会多次应用呢？通常是注解的值可以同时取多个。

举个例子，一个人他既是程序员又是产品经理,同时他还是个画家。

```java
//按照规定，它里面必须要有一个 value 的属性
//属性类型是一个被 @Repeatable 注解过的注解数组，注意它是数组。
@interface Persons {
    Person[]  value();
}

//@Repeatable 后面括号中的类相当于一个容器注解
//什么是容器注解呢？就是用来存放其它注解的地方。它本身也是一个注解。
@Repeatable(Persons.class)
@interface Person{
    String role() default "";
}

//有了上面两个注解，Persons相当于一个总标签
//他里面可以放任意多个子标签，这些子标签类型是Person
//并且是存放于这个总标签的Person类型的数组中。

//那么既然有了总标签和放子标签的数组，那么，下面就可以定义子标签
//子标签的类型自然就是Person，里面这里假设定义role属性
//就是说这些子标签表示人的角色。
//自然也就支持多种角色，那么定义多次即可。如下

@Person(role="artist")
@Person(role="coder")
@Person(role="PM")
public class SuperMan{

}
```


## 注解的属性


```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {

    public int id() default -1;

    public String msg() default "Hi";

}
```
使用：


```java
//有默认值的时候，可以不执行属性
@TestAnnotation(id=3,msg="hello annotation")
public class Test {

}
```

有一些规则：

- 修饰符只能是`public` 或默认(`default`)
- 参数成员只能用基本类型`byte`,`short`,`int`,`long`,`float`,`double`,`boolean`,`char`八种基本类型和`String`,`Enum`,`Class`,`annotations`及这些类型的数组
- 如果只有一个参数成员,最好将名称设为”value”
- 注解元素必须有确定的值,可以在注解中定义默认值,也可以使用注解时指定,非基本类型的值不可为null,常使用空字符串或0作默认值
- 在表现一个元素存在或缺失的状态时,定义一下特殊值来表示,如空字符串或负值

## Java 预置的注解

> @Deprecated

这个元素是用来标记过时的元素，想必大家在日常开发中经常碰到。编译器在编译阶段遇到这个注解时会发出提醒警告，告诉开发者正在调用一个过时的元素比如过时的方法、过时的类、过时的成员变量。

> @Override

这个大家应该很熟悉了，提示子类要复写父类中被 `@Override` 修饰的方法

> @SuppressWarnings

阻止警告的意思。之前说过调用被 `@Deprecated` 注解的方法后，编译器会警告提醒，而有时候开发者会忽略这种警告，他们可以在调用的地方通过 `@SuppressWarnings` 达到目的。

> @SafeVarargs

参数安全类型注解。它的目的是提醒开发者不要用参数做一些不安全的操作,它的存在会阻止编译器产生 `unchecked` 这样的警告。

> @FunctionalInterface

函数式接口注解，这个是 Java 1.8 版本引入的新特性。函数式编程很火，所以 Java 8 也及时添加了这个特性。

函数式接口 (`Functional Interface`) 就是一个具有一个方法的普通接口。


```java
@FunctionalInterface
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}
```
我们进行线程开发中常用的 `Runnable` 就是一个典型的函数式接口，上面源码可以看到它就被 `@FunctionalInterface` 注解。

可能有人会疑惑，函数式接口标记有什么用，这个原因是函数式接口可以很容易转换为 `Lambda` 表达式。

## 注解与反射

> 注解通过反射获取。首先可以通过 Class 对象的 isAnnotationPresent() 方法判断它是否应用了某个注解


```java
public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {}
```
> 然后通过 getAnnotation() 方法来获取 Annotation 对象。


```java
public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {}
```

> 或者是 getAnnotations() 方法。



```java
public Annotation[] getAnnotations() {}
```

这里举个例子：

首先定义一个注解：


```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {

    public int id() default -1;

    public String msg() default "Hi";

}
```

然后定义一个类，打上这个注解：

```java
@TestAnnotation
public class Test {
}
```
最后再main函数中拿到注解：

```java
public class Main4 {
    public static void main(String[] args) {
        //判断Test.class中是否存在TestAnnotation注解
        boolean hasAnnotation = Test.class.isAnnotationPresent(TestAnnotation.class);
        if(hasAnnotation){
            System.out.println("注解存在...");
            //从Test类中拿出TestAnnotation注解
            TestAnnotation annotation = Test.class.getAnnotation(TestAnnotation.class);
            //拿到注解之后，可以拿出注解中的属性对应的默认值
            System.out.println(annotation.id());
            System.out.println(annotation.msg());
        }
    }
}
```
上面演示的是从类上拿到注解，对于属性、方法同样都可以用反射拿到注解。


```java
public class Test {

    @Check(value="hi")
    int a;


    @Perform
    public void testMethod(){}


    public static void main(String[] args) {
        try {
/*************拿到属性上的注解****************/
            Field a = Test.class.getDeclaredField("a");
            a.setAccessible(true);
            //获取一个成员变量上的注解
            Check check = a.getAnnotation(Check.class);

            if ( check != null ) {
                System.out.println("check value:"+check.value());
            }


/*************拿到方法上的注解****************/
            Method testMethod = Test.class.getDeclaredMethod("testMethod");

            if ( testMethod != null ) {
                // 获取方法中的注解
                Annotation[] ans = testMethod.getAnnotations();
                for( int i = 0;i < ans.length;i++) {
                    System.out.println("method testMethod annotation:"+ans[i].annotationType().getSimpleName());
                }
            }
        } catch (NoSuchFieldException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
            System.out.println(e.getMessage());
        } catch (SecurityException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
            System.out.println(e.getMessage());
        } catch (NoSuchMethodException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
            System.out.println(e.getMessage());
        }



    }

}
```

## 注解实现原理

在上面获取注解时是这样写的：


```java
TestAnnotation annotation = Test.class.getAnnotation(TestAnnotation.class);
```
它是从`class`中获取出`TestAnnotation`注解的，所以肯定是在某个时候注解被加入到`class`结构中去了。

- 首先，我们知道从`java`源码到`class`字节码是由编译器完成的，编译器会对`java`源码进行解析并生成`class`文件。
- 而注解也是在编译时由编译器进行处理，编译器会对注解符号处理并附加到`class`结构中
- 根据`jvm`规范，`class`文件结构是严格有序的格式，唯一可以附加信息到`class`结构中的方式就是保存到`class`结构的`attributes`属性中
- 我们知道对于类、字段、方法，在`class`结构中都有自己特定的表结构，而且各自都有自己的属性，而对于注解，作用的范围也可以不同，可以作用在类上，也可以作用在字段或方法上，这时编译器会对应将注解信息存放到类、字段、方法自己的属性上。
- 在我们的`Test`类被编译后，在对应的`Test.class`文件中会包含一个`RuntimeVisibleAnnotations`属性，由于这个注解是作用在类上，所以此属性被添加到类的属性集上。即`TestAnnotation`注解的键值对`value=test`会被记录起来。
- 而当`JVM`加载`Test.class`文件字节码时，就会将`RuntimeVisibleAnnotations`属性值保存到`Test`的`Class`对象中，于是就可以通过`Test.class.getAnnotation(TestAnnotation.class)`获取到`Test`注解对象，进而再通过`Test`注解对象获取到`Test`里面的属性值。
- `Test`注解对象是什么？其实注解被编译后的本质就是一个继承`Annotation`接口的接口。所以`@TestAnnotation`其实就是“public interface TestAnnotation extends Annotation”
- 当我们通过`Test.class.getAnnotation(TestAnnotation.class)`调用时，`JDK`会通过动态代理生成一个实现了`TestAnnotation`接口的对象，并把将`RuntimeVisibleAnnotations`属性值设置进此对象中，此对象即为`TestAnnotation`注解对象，通过它的`value()`方法就可以获取到注解值。

## 总结注解到底是什么以及注解到底有什么应用场景



注释是给人看的，**注解是给编译器看的**，以`@Override`注解为例，他的作用是告诉编译器他所注解的方法是重写父类中的方法，这样编译器就会去检查父类是否存在这个方法，以及这个方法的签名与父类是否相同。

**注解不会也不能影响代码的实际逻辑，仅仅起到辅助性的作用**；

也就是说，注解只是描述java代码的代码，能被编译器解析，**只有编译器或者虚拟机来主动解析他的时候，他才可能发挥作用**。

注解分为三类，元注解，java自带的标准注解以及自定义注解。


注解的使用场景：

- 生成文档，通过代码里标识的元数据生成javadoc文档。
- 编译检查，通过代码里标识的元数据让编译器在编译期间进行检查验证。
- 编译时动态处理，编译时通过代码里标识的元数据动态处理，例如动态生成代码。
- 运行时动态处理，运行时通过代码里标识的元数据动态处理，例如使用反射注入实例。


我觉得这些说的太空洞了，注解在`spring`中就是非常常用的技术，比如，我指定这个类是`@Controller`或者`@Service`之类，那么我配置包扫描将其类路径全部扫描到后，启动容器的时候，这些类就会自动被spring所管理，即自动向`spring`注册，以后要注入这些组件的时候，就直接从`spring`的`IOC`容器中取出来即可。

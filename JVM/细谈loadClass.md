title: 细谈loadClass
tag: JVM
---

本篇为学习JAVA虚拟机的第四篇文章，对于获取Class对象，其实我们不知不觉中已经接触过两种了，一种就是loadClass，一种就是[反射](http://fourcolor.oursnail.cn/2019/02/04/java-basic/%E5%BD%BB%E5%BA%95%E7%90%86%E8%A7%A3java%E5%8F%8D%E5%B0%84/)中的forName，它们到底有什么区别呢？其实涉及了类加载过程的区别。下面好好来探讨一下。

<!-- more -->

## 一、问题的提出

对于之前的 测试代码：

```java
public class Test {
    public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        MyClassLoader myClassLoader = new MyClassLoader("C:\\Users\\swg\\Desktop\\","myClassLoader");
        Class c = myClassLoader.loadClass("Robot");
        System.out.println(c.getClassLoader());
        c.newInstance();
    }
}
```

不知道大家有没有疑惑，我们这里是用了`loadClass(name)`来加载对应的`Class`对象的，最后还需要进行`newInstance()`。那么为什么要调用`newInstance()`才行呢？

##### 1.1 new的方式构建对象实例

下面要进行相应的测试。对于`Robot.java`:


![image](http://bloghello.oursnail.cn/jvm4-1.png)

首先用`new`的方式：

![image](http://bloghello.oursnail.cn/jvm4-2.png)

显示结果为：


```
hello , i am a robot!
```

##### 1.2 loadClass来获取Class对象

![image](http://bloghello.oursnail.cn/jvm4-3.png)

如果仅仅这样写，显示结果仅仅为：


```
sun.misc.Launcher$AppClassLoader@18b4aac2
```
也就是说，并不会触发`static`静态块的执行，也就是说这个类根本就没有初始化。

##### 1.3 forName来获取Class对象

![image](http://bloghello.oursnail.cn/jvm4-4.png)

显示结果为：


```
hello , i am a robot!
```
触发了静态块的执行。


## 二、类加载过程

要想说明上面区别产生的原因，这里必须要介绍一个从未使用过的类加载的过程。

类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：加载（`Loading`）、验证（`Verification`）、准备(`Preparation`)、解析(`Resolution`)、初始化(`Initialization`)、使用(`Using`)和卸载(`Unloading`)7个阶段。其中准备、验证、解析3个部分统称为连接（`Linking`）。如图所示：

![image](http://bloghello.oursnail.cn/jvm4-5.png)

加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班地开始，而解析阶段则不一定：它在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定）。



##### 2.1 加载

**在加载阶段**（可以参考`java.lang.ClassLoader`的`loadClass()`方法），虚拟机需要完成以下3件事情：

- 通过一个类的全限定名来获取定义此类的二进制字节流（并没有指明要从一个`Class`文件中获取，可以从其他渠道，譬如：网络、动态生成、数据库等）；
- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构；
- 在内存中生成一个代表这个类的`java.lang.Class`对象，作为方法区这个类的各种数据的访问入口；

加载阶段和连接阶段（`Linking`）的部分内容（如一部分字节码文件格式验证动作）是交叉进行的，加载阶段尚未完成，连接阶段可能已经开始，但这些夹在加载阶段之中进行的动作，仍然属于连接阶段的内容，这两个阶段的开始时间仍然保持着固定的先后顺序。


##### 2.2 验证

验证是连接阶段的第一步，这一阶段的目的是为了确保`Class`文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。 

**验证阶段大致会完成4个阶段的检验动作**：

> 文件格式验证：验证字节流是否符合`Class`文件格式的规范；例如：是否以魔术`0xCAFEBABE`开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中的常量是否有不被支持的类型。

> 元数据验证：对字节码描述的信息进行语义分析（注意：对比`javac`编译阶段的语义分析），以保证其描述的信息符合Java语言规范的要求；例如：这个类是否有父类，除了`java.lang.Object`之外。

> 字节码验证：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。

> 符号引用验证：确保解析动作能正确执行。

验证阶段是非常重要的，但不是必须的，它对程序运行期没有影响，如果所引用的类经过反复验证，那么可以考虑采用`-Xverifynone`参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。




##### 2.3 准备

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些变量所使用的内存都将在方法区中进行分配。这时候进行内存分配的仅包括类变量（被`static`修饰的变量），而不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在堆中。其次，这里所说的初始值“通常情况”下是数据类型的零值，假设一个类变量的定义为：


```java
public static int value=123;
```

那变量`value`在准备阶段过后的初始值为0而不是123.因为这时候尚未开始执行任何java方法，而把`value`赋值为123的`putstatic`指令是程序被编译后，存放于类构造器()方法之中，所以把`value`赋值为123的动作将在初始化阶段才会执行。 

**至于“特殊情况”是指**：`public static final int value=123`，即当类字段的字段属性是`ConstantValue`时，会在准备阶段初始化为指定的值，所以标注为`final`之后，`value`的值在准备阶段初始化为123而非0.


##### 2.4 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号引用进行。

对于这里说的：将符号引用替换为直接引用。很多人包括我第一次看到的时候感觉莫名其妙，教材上也是直接用这些专用名词，给我们的学习带来了极大的困扰。这里还是要解释一下。

比如以下代码：


```java
public static void main(String[] args) {
    String s = "abc";
}
```

`s`是符号引用，而`abc`是字面量。

此时，知道了什么是符号引用就好办了，因为符号引用一般都是放在栈中的，这个玩意肯定是依赖于实际的东西，相当于一个指针，多以我们程序需要将其解析成这个实际东西所在的真正的地址。所以，一旦解析了，那么内存中必然实际存在了这个对象，即拥有实际的物理地址了。

##### 2.5 初始化

类初始化阶段是类加载过程的最后一步，到了初始化阶段，才真正开始执行类中定义的java程序代码。在准备阶段，变量已经赋过一次系统要求的初始值，而在初始化阶段，则根据程序猿通过程序制定的主观计划去初始化类变量和其他资源，或者说：初始化阶段是执行类构造器`<clinit>()`方法的过程. 




## 三、new、loadClass、forName

正常情况下，我们一般构建对象实例是通过`new`的方式，`new`是隐式构建对象实例，不需要`newInstance()`，并且可以用带参数的构造器来生成对象实例；

对于`new`，我们有点基础的，是知道，已经一直来到了最后初始化完成的这一步，生成了可以直接使用的对象实例。由于篇幅不宜太长，不想展开讲new的过程发生了什么，这里先贴个我觉得讲的不错的链接：https://www.jianshu.com/p/ebaa1a03c594


然而`loadClass(name)`这种显示调用的方式，我们可以看到，只有加载的功能，而没有后续连接以及初始化的过程。

所以`loadClass(name)`需要进行`newInstance()`才能生成对应的对象实例，并且这个`newInstance()`方法不支持参数调用，要想实现输入参数生成实例对象，需要通过反射获取构造器对象传入参数再生成对象实例。

这里也就解释了为什么要`newInstance()`，因为不这样，`loadClass(name)`只是加载，并没有后续过程，也就是说这个类根本就没有动它，仅仅是加载进来而已。从代码层面调用`loadClass()`的时候，我们可以看到一个之前故意忽视的东西：

![image](http://bloghello.oursnail.cn/jvm4-6.png)

![image](http://bloghello.oursnail.cn/jvm4-7.png)

这个`resolve`默认是传入`false`的，那么进来看看这个`resolveClass()`方法：


![image](http://bloghello.oursnail.cn/jvm4-8.png)

再下去是`native`方法，不必关心，我们只看方法的注释即可，写的是链接指定的类，就是上面的连接过程。我们由上面知道，如果这个方法能执行，那么就会触发验证、准备、解析这三个过程，而准备阶段是会去执行静态方法或静态块，类变量会被进行初始化，即分配内存，但是仅仅赋初值即可。

所以，`loadClass(name)`有一种懒加载的思想在里面，要用了再去进行初始化，而不是一开始就初始化好。

既然已经知道了`new`和`loadClass`的区别了，下面再来看看`Class.forName()`,聪明的读者估计已经可以猜到了，没错，根据实验的结果来看，它至少要进行到连接完，实质它也完成了初始化，即已经到达第三步：

![image](http://bloghello.oursnail.cn/jvm4-9.png)

总结一下：`loadClass`仅仅是第一步的加载，而`forName`和`new`都是已经初始化好了。

## 存在的原因

所谓存在即合理，`forName`的用法，最常见的莫过于用于加载数据库驱动这，我们这里实验一下，首先引入相关的依赖：

![image](http://bloghello.oursnail.cn/jvm4-10.png)


经典写法来啦：

![image](http://bloghello.oursnail.cn/jvm4-11.png)

点进去看看：

![image](http://bloghello.oursnail.cn/jvm4-12.png)

我们这个时候发现，里面是一个`static`方法，也就是说，我们要立即创建驱动。所以这个时候必须用`forname`方法啦！

那么对于`loadClass`，其实上面已经提及了，就是懒加载，这个思想再`spring`中是到处可见的，`bean`只是加载，但是步进行初始化，等用的时候再去初始化，提高性能。
title: java基础之泛型上
tag: java基础
---

本篇文章全面介绍Java泛型中的基础及原理。本节主要介绍什么是泛型、泛型的核心特性、泛型与继承注意点、泛型与多态的原理以及泛型的使用。
<!-- more -->

## 1、什么是泛型以及为什么用泛型

直接上例子进行说明：

![image](http://bloghello.oursnail.cn/javabasic10-1.png)

毫无疑问，程序的运行结果会以崩溃结束：

```java
java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
```

为什么会出现这种问题呢？

+ 集合本身无法对其存放的对象类型进行限定，可以涵盖Java中的所有类型。缺口太大，导致各种蛇、蚁、虫、鼠通通都可以进来。

- 由于我们要使用的实际存放类型的方法，所以不可避免地要进行类型转换。小对象转大对象很容易，大对象转小对象则有很大的风险，因为在编译时，我们无从得知对象真正的类型。

泛型就是为了解决这类问题而诞生的。




## 2、泛型的特性

##### 2.1 泛型只在编译阶段有效

![image](http://bloghello.oursnail.cn/javabasic10-2.png)

输出结果：类型相同

> 通过上面的例子可以证明，在编译之后程序会采取去泛型化的措施。<font color=#ff0000>**也就是说Java中的泛型，只在编译阶段有效**。在编译过程中，正确检验泛型结果后，会将泛型的相关信息擦除，并且在对象进入和离开方法的边界处添加类型检查和类型转换的方法。也就是说，泛型信息不会进入到运行时阶段。</font>

**对此总结成一句话：泛型类型在逻辑上看以看成是多个不同的类型，实际上都是相同的基本类型。**


##### 2.2 泛型的兼容性

Java编译器是向后兼容的，也就是低版本的源代码可以用高版本编译器进行编译。下面来看看那些兼容性代码。

> 1. 引用和实例化都不包含泛型信息。

![image](http://bloghello.oursnail.cn/javabasic10-3.png)

上面的这段代码是可以通过编译的，这是JDK1.4之前的写法，所以可以验证JDK1.5之后的编译器是可以兼容JDK1.4之前的源代码的。不过，笔者在JDK1.8.x版本的编译器进行编译时，会抛出如下所示的警告信息。很显然，如果类被定义成泛型类，但是在实际使用时不使用泛型特性，这是不推荐的做法！


```
注: Compatibility.java使用了未经检查或不安全的操作。
注: 有关详细信息, 请使用 -Xlint:unchecked 重新编译。
```

> 2. 引用使用泛型，实例化不使用泛型。

![image](http://bloghello.oursnail.cn/javabasic10-4.png)

上面的代码编译不通过，由于对引用使用了泛型，其中的所能容纳的对象必须为String 类型。这种写法实际上跟完整写法的作用一致，不过Eclipse仍然会警告
```
上面的代码编译不通过，由于对引用使用了泛型，其中的所能容纳的对象必须为String 类型。这种写法实际上跟完整写法的作用一致，不过Eclipse仍然会警告。
```

> 3. 引用不使用泛型，实例化使用泛型。

![image](http://bloghello.oursnail.cn/javabasic10-5.png)

上面的这段代码可以编译通过，其效果与1（不使用泛型）完全一致。结合2、3可以知道，编译时只能做引用的类型检查，而无法检查引用所指向对象的实际类型。

## 3、泛型与继承

在使用泛型时，引用的参数类型与实际对象的参数类型要保持一致（通配符除外），就算两个参数类型是继承关系也是不允许的。看看下面的2行代码，它们均不能通过编译。

```java
ArrayList<String> arrayList1 = new ArrayList<Object>(); //编译错误  
ArrayList<Object> arrayList1 = new ArrayList<String>(); //编译错误
```

下面来探讨一下为什么不能这么做。

- 第1种情况，如果这种代码可以通过编译，那么调用`get()`方法返回的对象应该是`String`，但它实际上可以存放任意`Object`类型的对象，这样在调用类型转换指令时会抛出`ClassCastException`。
- 第2种情况。虽然`String`类型的对象转换为`Object`不会有任何问题，但是这有什么意义呢？我们原本想要用`String`对象的方法，但最终将其赋予了一个`Object`类型的引用。如果需要使用`String`中的某些方法，必须将`Object`强制转换为`String`。这样不会抛出异常，但是却违背了泛型设计的初衷。

## 4、泛型与多态

下面来考虑一下泛型中多态问题。普通类型的多态是通过继承并重写父类的方法来实现的，泛型也不例外，下面是一个泛型多态示例。


![image](http://bloghello.oursnail.cn/javabasic10-6.png)

![image](http://bloghello.oursnail.cn/javabasic10-7.png)

上面定义了一个泛型父类和一个实际参数为`String`类型的子类，并“重写”了`set(T)`和`get()`方法。`Son`类中的`@Override`注解也清楚地显示这是一个重写方法，最终执行的结果如下，与想象中的结果完全一致。

```
I am father, t=hello world
I am son.
```

真的这么简单么？虽然表面上（源代码层面）来看，泛型多态与普通类的多态并无二样，但是其内部的实时原理却大相径庭。

泛型类`Father`在编译后会擦除泛型信息，所有的泛型参数都会用`Object`类替代。实际上，`Father`编译后的字节码与下面的代码完全一致。

![image](http://bloghello.oursnail.cn/javabasic10-8.png)

`Son`类的与最终会变为：

![image](http://bloghello.oursnail.cn/javabasic10-9.png)

`Father`和`Son`类的`set()`方法的参数类型不一样，所以，这并不是方法重写，而是方法重载！但是，如果是重载，那么`Son`类就应该会继承`Father`类的`set(Object)`方法，也就是`Son`会同时包含`set(String)`和`set(Object)`，下面来测试一下。


```java
Son son = new Son();
son.set("test");
son.set(new Object()); // 编译错误
```

当`set`一个`Object`对象时，编译无法通过。这就很奇怪了，感觉跟之前学到的知识是相悖的。我们原本想通过重写方法来实现多态，但由于泛型的类型擦除，却最终变成了重载，所以类型擦除与多态有了矛盾。那么Java是怎么解决这个问题的呢？还是从字节码中找答案吧。`Son`类最终的编译结果如下：


```java
public void set(java.lang.String);         // 我们重写的方法
public java.lang.String get();              // 我们重写的方法
public java.lang.Object get();              // 编译器生成的方法
public void set(java.lang.Object);          // 编译器生成的方法
    ...
    2: checkcast     #39                 // class java/lang/String
    ...
```


<font color=#ff0000>⭐这里面多了一个`Object get()`方法和`set(Object)`方法，这两个方法在`Son`类源代码里面并不存在，这是编译器为了解决泛型的多态问题而自动生成的方法，称为“桥方法”。这两个方法的签名与`Father`类中的两个方法的签名完全一致，这才是真正的方法重写。也就是说，子类真正重写的我们看不到的桥方法，啊，多么痛的领悟！！！`@Override`注解只是假象，让人误以为他们真的是重写方法。</font>

再看看`set(Object)`桥方法的实现细节，先将`Object`对象强制转换为`String`对象，然后调用`Son`中的`set(String)`方法。饶了一个圈，最终才回到我们“重写”的方法。`main`方法中原本调用父类的`set(Object)`方法，由于子类通过桥方法重写了这个方法，所以最终的调用顺序是：`set(Object)` -> `set(String)`。

与`set(Object)`桥方法的意义不同，`Object get()`并不仅仅解决泛型与重写的冲突，而更具有一般性。看看下面的代码，这是一个普通类的继承:

```java
public class GeneralFather {
	public Object get() {
		return null;
	}
}
```


```java
public class GeneralSon extends GeneralFather {
	@Override
	public String get() {
		return "";
	}
}
```

子类的返回类型是父类的返回类型的子类，这是允许的，这种特性叫做Java返回值的协变性。而协变性的实现方法就是上面所述的桥方法。

这里还会有疑惑，`set`方法可以通过参数类型来确定调用的方法。但是，参数一样而返回值不一样是不能重载的。如果我们在源代码中通过编写`String get()`和`Object get()`方法是无法通过编译的。虽然，编译器无法通过编译，但是JVM是可以编写这两种方法的，它调用方法时，将返回值也作为方法签名的一部分。有种只许州官放火，不许百姓点灯的感觉。可以看到，JVM做了不少我们认为不合法的事情，所以如果不深入研究底层原理，有些问题根本解释不了。



## 5、泛型的使用

泛型有三种使用方式，分别为：泛型类、泛型接口、泛型方法.

##### 5.1 泛型类

泛型类型用于类的定义中，被称为泛型类。通过泛型可以完成对一组类的操作对外开放相同的接口。最典型的就是各种容器类，如：`List`、`Set`、`Map`。

![image](http://bloghello.oursnail.cn/javabasic10-10.png)

下面进行实例化：

![image](http://bloghello.oursnail.cn/javabasic10-11.png)

结果为：
```
12-27 09:20:04.432 13063-13063/? D/泛型测试: key is 123456
12-27 09:20:04.432 13063-13063/? D/泛型测试: key is key_vlaue
```

定义的泛型类，就一定要传入泛型类型实参么？并不是这样，在使用泛型的时候如果传入泛型实参，则会根据传入的泛型实参做相应的限制，此时泛型才会起到本应起到的限制作用。如果不传入泛型类型实参的话，在泛型类中使用泛型的方法或成员变量定义的类型可以为任何的类型。


![image](http://bloghello.oursnail.cn/javabasic10-12.png)


```
D/泛型测试: key is 111111
D/泛型测试: key is 4444
D/泛型测试: key is 55.55
D/泛型测试: key is false
```

##### 5.2 泛型接口

![image](http://bloghello.oursnail.cn/javabasic10-13.png)

当实现泛型接口的类，未传入泛型实参时：

![image](http://bloghello.oursnail.cn/javabasic10-14.png)

当实现泛型接口的类，传入泛型实参时：

![image](http://bloghello.oursnail.cn/javabasic10-15.png)

##### 5.3 泛型通配符

<font color=#ff0000>我们知道`Ingeter`是`Number`的一个子类，同时我们也验证过`Generic<Ingeter>`与`Generic<Number>`实际上是相同的一种基本类型。那么问题来了，在使用`Generic<Number>`作为形参的方法中，能否使用`Generic<Ingeter>`的实例传入呢？在逻辑上类似于`Generic<Number>`和`Generic<Ingeter>`是否可以看成具有父子关系的泛型类型呢？</font>

为了弄清楚这个问题，我们使用`Generic<T>`这个泛型类继续看下面的例子：


![image](http://bloghello.oursnail.cn/javabasic10-16.png)


![image](http://bloghello.oursnail.cn/javabasic10-17.png)

通过提示信息我们可以看到`Generic<Integer>`不能被看作为`Generic<Number>`的子类。由此可以看出:<font color=#ff0000>**同一种泛型可以对应多个版本（因为参数类型是不确定的），不同版本的泛型类实例是不兼容的**</font>。

回到上面的例子，如何解决上面的问题？总不能为了定义一个新的方法来处理`Generic<Integer>`类型的类，这显然与java中的多态理念相违背。因此我们需要一个在逻辑上可以表示同时是`Generic<Integer>`和`Generic<Number>`父类的引用类型。由此类型通配符应运而生。

我们可以将上面的方法改一下：

![image](http://bloghello.oursnail.cn/javabasic10-18.png)

类型通配符一般是使用`'?'`代替具体的类型实参，注意，<font color=#ff0000>**此处'?'是类型实参，而不是类型形参**</font> 。重要说三遍！此处`'?'`是类型实参，而不是类型形参 ！ 此处`'?'`是类型实参，而不是类型形参 ！再直白点的意思就是，此处的`'?'`和`Number`、`String`、`Integer`一样都是一种实际的类型，可以把`'?'`看成所有类型的父类。是一种真实的类型。

可以解决当具体类型不确定的时候，这个通配符就是`'?'`；当操作类型时，不需要使用类型的具体功能时，只使用`Object`类中的功能。那么可以用`'?'`通配符来表示未知类型。

##### 5.4 泛型方法


泛型类，是在实例化类的时候指明泛型的具体类型；泛型方法，是在调用方法的时候指明泛型的具体类型 。


![image](http://bloghello.oursnail.cn/javabasic10-19.png)

```java
Object obj = genericMethod(Class.forName("com.test.test"));
```

再对泛型方法进行一个比较，加深理解：

```java
public class GenericTest {
   //这个类是个泛型类，在上面已经介绍过
   public class Generic<T>{     
        private T key;

        public Generic(T key) {
            this.key = key;
        }

        //我想说的其实是这个，虽然在方法中使用了泛型，但是这并不是一个泛型方法。
        //这只是类中一个普通的成员方法，只不过他的返回值是在声明泛型类已经声明过的泛型。
        //所以在这个方法中才可以继续使用 T 这个泛型。
        public T getKey(){
            return key;
        }

        /**
         * 这个方法显然是有问题的，在编译器会给我们提示这样的错误信息"cannot reslove symbol E"
         * 因为在类的声明中并未声明泛型E，所以在使用E做形参和返回值类型时，编译器会无法识别。
        public E setKey(E key){
             this.key = key；
        }
        */
        
        //必须要声明E才行
        public <E> E setKey(E key){
            this.key = (T)key;
            return key;
        }
        
    }

    /** 
     * 这才是一个真正的泛型方法。
     * 首先在public与返回值之间的<T>必不可少，这表明这是一个泛型方法，并且声明了一个泛型T
     * 这个T可以出现在这个泛型方法的任意位置.
     * 泛型的数量也可以为任意多个 
     *    如：public <T,K> K showKeyName(Generic<T> container){
     *        ...
     *        }
     */
    public <T> T showKeyName(Generic<T> container){
        System.out.println("container key :" + container.getKey());
        //当然这个例子举的不太合适，只是为了说明泛型方法的特性。
        T test = container.getKey();
        return test;
    }

    //这也不是一个泛型方法，这就是一个普通的方法，只是使用了Generic<Number>这个泛型类做形参而已。
    public void showKeyValue1(Generic<Number> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

    //这也不是一个泛型方法，这也是一个普通的方法，只不过使用了泛型通配符?
    //同时这也印证了泛型通配符章节所描述的，?是一种类型实参，可以看做为Number等所有类的父类
    public void showKeyValue2(Generic<?> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

     /**
     * 这个方法是有问题的，编译器会为我们提示错误信息："UnKnown class 'E' "
     * 虽然我们声明了<T>,也表明了这是一个可以处理泛型的类型的泛型方法。
     * 但是只声明了泛型类型T，并未声明泛型类型E，因此编译器并不知道该如何处理E这个类型。
    public <T> T showKeyName(Generic<E> container){
        ...
    }  
    */

    /**
     * 这个方法也是有问题的，编译器会为我们提示错误信息："UnKnown class 'T' "
     * 对于编译器来说T这个类型并未项目中声明过，因此编译也不知道该如何编译这个类。
     * 所以这也不是一个正确的泛型方法声明。
    public void showkey(T genericObj){

    }
    */
    
    
    //在泛型类中声明了一个泛型方法，使用泛型E，这种泛型E可以为任意类型。可以类型与T相同，也可以不同。
    //由于泛型方法在声明的时候会声明泛型<E>，因此即使在泛型类中并未声明泛型，编译器也能够正确识别泛型方法中识别的泛型。
    public <E> void show_3(E t){
        System.out.println(t.toString());
    }

    //在泛型类中声明了一个泛型方法，使用泛型T，注意这个T是一种全新的类型，可以与泛型类中声明的T不是同一种类型。
    public <T> void show_2(T t){
        System.out.println(t.toString());
    }

    public static void main(String[] args) {


    }
}
```


##### 5.5 泛型方法与可变参数


![image](http://bloghello.oursnail.cn/javabasic10-20.png)

```java
printMsg("111",222,"aaaa","2323.4",55.55);
```

##### 5.6 静态方法与泛型

**如果静态方法要使用泛型的话，必须将静态方法也定义成泛型方法 。**

![image](http://bloghello.oursnail.cn/javabasic10-21.png)











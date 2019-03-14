title: java基础之克隆
tag: java基础
---

面试的时候可能会问到克隆相关的深拷贝和浅拷贝，至少我是被问过的，所以对它们的了解是必要的，本篇文章探讨Java克隆方面的知识。
<!-- more -->

## 1. Java中对象创建的两种方式


`clone`顾名思义就是复制， 在`Java`语言中， `clone`方法被对象调用，所以会复制对象。所谓的复制对象，首先要分配一个和源对象同样大小的空间，在这个空间中创建一个新的对象。那么在`java`语言中，有几种方式可以创建对象呢？

- 使用new操作符创建一个对象
- 使用clone方法复制一个对象

那么这两种方式有什么相同和不同呢？ `new`操作符的本意是分配内存。程序执行到`new`操作符时， 首先去看`new`操作符后面的类型，因为知道了类型，才能知道要分配多大的内存空间。分配完内存之后，再调用构造函数，填充对象的各个域，这一步叫做对象的初始化，构造方法返回后，一个对象创建完毕，可以把他的引用（地址）发布到外部，在外部就可以使用这个引用操纵这个对象。

而`clone`在第一步是和`new`相似的， 都是分配内存，调用`clone`方法时，分配的内存和源对象（即调用`clone`方法的对象）相同，然后再使用原对象中对应的各个域，填充新对象的域， 填充完成之后，`clone`方法返回，一个新的相同的对象被创建，同样可以把这个新对象的引用发布到外部。

## 2. 复制对象 or 复制引用

在Java中，以下类似的代码非常常见：

![image](http://bloghello.oursnail.cn/javabasic9-1.png)

当`Person p1 = p;`执行之后， 是创建了一个新的对象吗？ 首先看打印结果：

```
com.pansoft.zhangjg.testclone.Person@2f9ee1ac
com.pansoft.zhangjg.testclone.Person@2f9ee1ac
```


可已看出，打印的地址值是相同的，既然地址都是相同的，那么肯定是同一个对象。p和p1只是引用而已，他们都指向了一个相同的对象`Person(23, "zhang")` 。 可以把这种现象叫做引用的复制。

![image](http://xiaozhao.oursnail.cn/%E5%BC%95%E7%94%A8%E5%A4%8D%E5%88%B6.png)

而下面的代码是真真正正的克隆了一个对象。

![image](http://bloghello.oursnail.cn/javabasic9-2.png)

从打印结果可以看出，两个对象的地址是不同的，也就是说创建了新的对象， 而不是把原对象的地址赋给了一个新的引用变量：

```
com.pansoft.zhangjg.testclone.Person@2f9ee1ac
com.pansoft.zhangjg.testclone.Person@67f1fba0
```

以上代码执行完成后， 内存中的情景如下图所示：

![image](http://xiaozhao.oursnail.cn/%E5%85%8B%E9%9A%86%E5%AF%B9%E8%B1%A1.png)





## 3. 深拷贝 or 浅拷贝

![image](http://bloghello.oursnail.cn/javabasic9-3.png)

- `age`是基本数据类型，那么对它的拷贝没有什么疑议，直接将一个4字节的整数值拷贝过来就行。

- `name`是`String`类型的， 它只是一个引用， 指向一个真正的`String`对象，那么对它的拷贝有两种方式：  
    + 直接将源对象中的`name`的引用值拷贝给新对象的`name`字段
    + 或者是根据原`Person`对象中的`name`指向的字符串对象创建一个新的相同的字符串对象，将这个新字符串对象的引用赋给新拷贝的`Person`对象的`name`字段。
- 这两种拷贝方式分别叫做浅拷贝和深拷贝。深拷贝和浅拷贝的原理如下图所示：

![image](http://xiaozhao.oursnail.cn/%E6%B7%B1%E6%8B%B7%E8%B4%9D%E4%B8%8E%E6%B5%85%E6%8B%B7%E8%B4%9D.png)

下面通过代码进行验证。如果两个`Person`对象的`name`的地址值相同， 说明两个对象的`name`都指向同一个`String`对象， 也就是浅拷贝， 而如果两个对象的`name`的地址值不同， 那么就说明指向不同的`String`对象， 也就是在拷贝`Person`对象的时候， 同时拷贝了`name`引用的`String`对象， 也就是深拷贝。验证代码如下：

![image](http://bloghello.oursnail.cn/javabasic9-4.png)

覆盖`Object`中的`clone`方法， 实现深拷贝.假设`body`类里面组合了`head`类。

![image](http://bloghello.oursnail.cn/javabasic9-5.png)

`Body`中组合了`Head`，重写了`Body`的`clone`方法，那么显然第一个输出为`false`；但是没有对`Head`进行重写`clone`方法，那么他们指向的是同一个内存空间。即，没有重写`clone的Head`类只是浅拷贝。


```java
body == body1 : false
body.head == body1.head : true
```

如果要使`Body`对象在`clone`时进行深拷贝， 那么就要在`Body`的`clone`方法中，将源对象引用的`Head`对象也`clone`一份。

![image](http://bloghello.oursnail.cn/javabasic9-6.png)

打印结果：

```
body == body1 : false
body.head == body1.head : false
```

由此，我们得到一个结论：如果想要深拷贝一个对象， 这个对象必须要实现`Cloneable`接口，实现`clone`方法，并且在`clone`方法内部，把该对象引用的其他对象也要`clone`一份 ， 这就要求这个被引用的对象必须也要实现`Cloneable`接口并且实现`clone`方法。

那么，按照上面的结论， `Body`类组合了`Head`类， 而`Head`类组合了`Face`类，要想深拷贝`Body`类，必须在`Body`类的`clone`方法中将`Head`类也要拷贝一份，但是在拷贝`Head`类时，默认执行的是浅拷贝，也就是说`Head`中组合的`Face`对象并不会被拷贝。验证代码如下：

![image](http://bloghello.oursnail.cn/javabasic9-7.png)

输出结果符合预期：

```
body == body1 : false
body.head == body1.head : false
body.head.face == body1.head.face : true
```
内存结构图如下图所示：

![image](http://xiaozhao.oursnail.cn/%E5%85%8B%E9%9A%86%E5%AF%B9%E8%B1%A12.png)

那么此时`Head`中组合的`Face`又是一个浅拷贝。那么到底如何实现彻底的深拷贝呢？

对于上面的例子来说，怎样才能保证两个`Body`对象完全独立呢？只要在拷贝`Head`对象的时候，也将`Face`对象拷贝一份就可以了。这需要让`Face`类也实现`Cloneable`接口，实现`clone`方法，并且在在`Head`对象的`clone`方法中，拷贝它所引用的`Face`对象。修改的部分代码如下：


![image](http://bloghello.oursnail.cn/javabasic9-8.png)

再次运行上面的示例，得到的运行结果如下：

```
body == body1 : false
body.head == body1.head : false
body.head.face == body1.head.face : false
```

这说名两个`Body`已经完全独立了，他们间接引用的`face`对象已经被拷贝，也就是引用了独立的`Face`对象。内存结构图如下：

![image](http://xiaozhao.oursnail.cn/%E5%85%8B%E9%9A%86%E5%AF%B9%E8%B1%A13.png)


显然，对于复杂的对象而言，用这种方式实现深拷贝是十分困难的。这时我们可以用序列化的方式来实现对象的深克隆。

## 4. 序列化解决多层克隆问题

首先由一个外部类`Outer`：

![image](http://bloghello.oursnail.cn/javabasic9-9.png)

再来一个被序列化的类`Inner`:

![image](http://bloghello.oursnail.cn/javabasic9-10.png)

再对克隆的对象进行测试：

![image](http://bloghello.oursnail.cn/javabasic9-11.png)


运行结果：

```
false
false
outer的name值为：outer
Inner的name值为：inner
```




参考：
- [Java提高篇——对象克隆（复制）](https://www.cnblogs.com/Qian123/p/5710533.html)
- [详解Java中的clone方法 -- 原型模式](https://blog.csdn.net/zhangjg_blog/article/details/18369201)
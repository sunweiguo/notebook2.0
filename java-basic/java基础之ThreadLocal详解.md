title: java基础之ThreadLocal详解
tag: java基础
---
ThreadLocal是面试重灾区，但是好像我没遇到过有人问，尴尬脸，不过我们不能做砧板上的鱼肉静静等待宰割，分为两篇来讲解其中的用法和原理。这是第一篇。
<!-- more -->


## 一、ThreadLocal简介

`ThreadLocal`类用来提供线程内部的局部变量。这些变量在多线程环境下访问(通过`get`或`set`方法访问)时能保证各个线程里的变量相对独立于其他线程内的变量，`ThreadLocal`实例通常来说都是`private static`类型。

`ThreadLocal`类提供了四个对外开放的接口方法，这也是用户操作`ThreadLocal`类的基本方法：

- `void set(Object value)`设置当前线程的线程局部变量的值。
- `public Object get()`该方法返回当前线程所对应的线程局部变量。
- `public void remove()`将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是JDK 5.0新增的方法。需要指出的是，当线程结束后，对应该线程的局部变量将自动被垃圾回收，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它可以加快内存回收的速度。
- `protected Object initialValue()`返回该线程局部变量的初始值，该方法是一个`protected`的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第1次调用`get()`或`set(Object)`时才执行，并且仅执行1次，`ThreadLocal`中的缺省实现直接返回一个`null`。



> 一个简单的小例子来感受ThreadLocal到底是什么以及怎么用：

![image](http://bloghello.oursnail.cn/javabasic14-1.png)

> 运行结果：

```
Thread-0
张三
李四
王五
Thread-1
Chinese
English
```
> 分析

可以，看出虽然多个线程对同一个变量进行访问，但是由于`threadLocal`变量由`ThreadLocal` 修饰，则不同的线程访问的就是该线程设置的值，这里也就体现出来`ThreadLocal`的作用。

当使用`ThreadLocal`维护变量时，`ThreadLocal`为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。


## 二、扒开JDK threadlocal神秘面纱

threadlocal的原理图为：

![image](http://xiaozhao.oursnail.cn/iKjk3GS.png)


那`ThreadLocal`内部是如何为每一个线程维护变量副本的呢？到底是什么原理呢？

先来看一下`ThreadLocal`的`set()`方法的源码是如何实现的：

![image](http://bloghello.oursnail.cn/javabasic14-2.png)


我们看到，首先通过`getMap(Thread t)`方法获取一个和当前线程相关的`ThreadLocalMap`，然后将变量的值设置到这个`ThreadLocalMap`对象中，当然如果获取到的`ThreadLocalMap`对象为空，就通过`createMap`方法创建。

我们再往下面去一点，比如`map.set`方法到底是怎么实现的？

![image](http://bloghello.oursnail.cn/javabasic14-3.png)


结合上面的图，其实我们可以发现，数据并不是放在所谓的`Map`集合中，而是放进了一个`Entry`数组中，这个`entry`索引是上面计算好的，`entry`的`key`是指向`threadLocal`的一个软引用，`value`是指向真实数据的一个强引用，以后再获取的时候，再以同样的方式计算得到索引下标即可。

> 上面代码出现的 ThreadLocalMap 是什么？

**`ThreadLocalMap`是`ThreadLocal`类的一个静态内部类，它实现了键值对的设置和获取（对比Map对象来理解），每个线程中都有一个独立的`ThreadLocalMap`副本，它所存储的值，只能被当前线程读取和修改。**

我们深入看一下`getMap`和`createMap`的实现

`getMap`:

![image](http://bloghello.oursnail.cn/javabasic14-4.png)

`createMap`:

![image](http://bloghello.oursnail.cn/javabasic14-5.png)

代码非常直白，就是获取和设置`Thread`内的一个叫`threadLocals`的变量，而这个变量的类型就是`ThreadLocalMap`，这样进一步验证了上文中的观点：**每个线程都有自己独立的`ThreadLocalMap`对象**。

`Thread`源码中的`threadLocals`：

![image](http://bloghello.oursnail.cn/javabasic14-6.png)

我们接着看`ThreadLocal`中的`get`方法如下


![image](http://bloghello.oursnail.cn/javabasic14-7.png)

- 第一步 先获通过`Thread.currentThread（）`取当前线程
- 第二步 然后获取当前线程的`threadLocals`属性
- 第三步 在`threadLocals`属性里获取`Entry`实例
- 第四部 从`Entry`实例的`value`属性里获取到最后所要的`Object`对象

接下来讨论一下上面出现的`ThreadLocalMap`类以及`Entry`类，直接贴源码


![image](http://bloghello.oursnail.cn/javabasic14-8.png)

`Entry`是`ThreadLocalMap`的内部类，而且`ThreadLocalMap`里拥有一个类型为`Entry[]`的`table`属性，而且每个线程实例有自己的`ThreadLocalMap`。到这里结论已经很明显了：**负责保存`ThreadLocal`的`key`和`value`根本就不是一个`Map`类型，而是一个`Entry`数组!**

`Entry`继承`WeakReference`，因此继承拥有一个弱引用`referent`，而且自身也有一个`value`属性。`Entry`利用`referent`来保存`threadLocal`实例的弱引用，利用`value`保存`Object`的强引用。至于为什么一个是强引用，一个是弱引用，我们在下一篇中来探讨。

最后的问题是怎样在`Entry`数组里定位我们需要的`Entry`呢?其实上面在set的时候已经大概知道了，现在再来看看代码吧：


![image](http://bloghello.oursnail.cn/javabasic14-9.png)

留意`key.threadLocalHashCode`这个属性，`Entry`在保存进`Entry[]`数组之前，会利用`ThreadLocal`的引用计算出一个`hash`值，然后利用这个`hash`值作为下标定位到`Entry[]`数组的某个位置；

![image](http://xiaozhao.oursnail.cn/thread_specific_storage_custom.png)

原理总结：`ThreadLocal`类并没有一个`Map`来保存数据，数据都是保存在线程实例上的；客户端访问`ThreadLocal`实例的`get`方法，`get`方法通过`Thread.getCurrentThread`获得当前线程的实例，从而获得当前线程的`ThreadLocalMap`对象，而`ThreadLocalMap`里包含了一个`Entry`数组，里面的每个`Entry`保存了`ThreadLocal`引用以及`Object`引用，`Entry`的`referent`保存`ThreadLocal`的弱引用，`Entry`的`value`保存`Object`的强引用。



## 三、threadLoca应用

> `threadlocal`实现的可复用的耗时统计工具`Profiler`

![image](http://bloghello.oursnail.cn/javabasic14-10.png)

运行结果：

```
Thread-0耗时： 1000
Thread-1耗时： 1999
```

> `threadLocal`实现数据库连接线程隔离

![image](http://bloghello.oursnail.cn/javabasic14-11.png)

通过调用`ConnectionManager.getConnection()`方法，每个线程获取到的，都是和当前线程绑定的那个`Connection`对象，第一次获取时，是通过`initialValue()`方法的返回值来设置值的。通过`ConnectionManager.setConnection(Connection conn)`方法设置的`Connection`对象，也只会和当前线程绑定。这样就实现了`Connection`对象在多个线程中的完全隔离。

在`Spring`容器中管理多线程环境下的`Connection`对象时，采用的思路和以上代码非常相似。

## 四、threadLocal缺陷
`ThreadLocal`变量的这种隔离策略，也不是任何情况下都能使用的。

如果多个线程并发访问的对象实例只能创建那么一个，那就没有别的办法了，老老实实的使用同步机制吧。

下一篇探讨`ThreadLocal` 内存泄漏问题。

参考：
- [深入理解ThreadLocal](http://vence.github.io/2016/05/28/threadlocal-info)
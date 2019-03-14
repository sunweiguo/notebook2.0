title: java基础之JDK动态代理
tag: java基础
---
代理模式可以说是经常面试被问的一个东西，因为spring aop的实现原理就是基于它，关于它，只要记住，它是运行时动态生成的一个代理类。在这个基础上，再去看看它底层源码，其实JDK已经帮我们最大程度上封装成简单的函数了，我们只需要传入几个参数就可以生成对应的代理对象。
<!-- more -->
## 代理模式是什么

定义：给某个对象提供一个代理对象，并由代理对象控制对于原对象的访问，即客户不直接操控原对象，而是通过代理对象间接地操控原对象。

* `RealSubject` 是原对象（本文把原对象称为"委托对象"），`Proxy` 是代理对象。
* `Subject` 是委托对象和代理对象都共同实现的接口。
* `Request()` 是委托对象和代理对象共同拥有的方法。

## 结合生活理解代理模式

要理解代理模式很简单，其实生活当中就存在代理模式：

> 我们购买火车票可以去火车站买，但是也可以去火车票代售处买，此处的火车票代售处就是火车站购票的代理，即我们在代售点发出买票请求，代售点会把请求发给火车站，火车站把购买成功响应发给代售点，代售点再告诉你。
>
> 但是代售点只能买票，不能退票，而火车站能买票也能退票，因此代理对象支持的操作可能和委托对象的操作有所不同。

## Java实现静态代理示例

![image](http://bloghello.oursnail.cn/javabasic8-1.png)

代理的实现分为：

* 静态代理

代理类是在编译时就实现好的。也就是说 Java 编译完成后代理类是一个实际的 `class` 文件。
* 动态代理

代理类是在运行时生成的。也就是说 Java 编译完之后并没有实际的 `class` 文件，而是在运行时动态生成的类字节码，并加载到JVM中。

## Java 实现动态代理

##### 几个重要名词:

* 委托类和委托对象：委托类是一个类，委托对象是委托类的实例，即原类。
* 代理类和代理对象：代理类是一个类，代理对象是代理类的实例。

##### Java实现动态代理的大致步骤如下:

1. 定义一个委托类和公共接口。
2. 自己定义一个类（调用处理器类，即实现 `InvocationHandler` 接口），这个类的目的是指定运行时将生成的代理类需要完成的具体任务（包括`Preprocess`和`Postprocess`），即代理类调用任何方法都会经过这个调用处理器类（在本文最后一节对此进行解释）。
3. 生成代理对象（当然也会生成代理类），需要为他指定(1)类加载器对象(2)实现的一系列接口(3)调用处理器类的实例。因此可以看出一个代理对象对应一个委托对象，对应一个调用处理器实例。

##### Java 实现动态代理主要涉及以下几个类:

* `java.lang.reflect.Proxy`: 这是生成代理类的主类，通过 `Proxy` 类生成的代理类都继承了 `Proxy` 类，即 `DynamicProxyClass extends Proxy`。
* `java.lang.reflect.InvocationHandler`: 这里称他为"调用处理器"，他是一个接口，我们动态生成的代理类需要完成的具体内容需要自己定义一个类，而这个类必须实现 `InvocationHandler` 接口。

##### Proxy 类主要方法为：

![image](http://bloghello.oursnail.cn/javabasic8-2.png)

- 这个静态函数的第一个参数是类加载器对象（即哪个类加载器来加载这个代理类到 JVM 的方法区）
- 第二个参数是接口（表明你这个代理类需要实现哪些接口）
- 第三个参数是调用处理器类实例（指定代理类中具体要干什么）。
- 这个函数是 JDK 为了程序员方便创建代理对象而封装的一个函数，因此你调用`newProxyInstance()`时直接创建了代理对象（略去了创建代理类的代码）。其实他主要完成了以下几个工作：

![image](http://bloghello.oursnail.cn/javabasic8-3.png)

`Proxy` 类还有一些静态方法，比如：

* `InvocationHandler getInvocationHandler(Object proxy)`: 获得代理对象对应的调用处理器对象。
* `Class getProxyClass(ClassLoader loader, Class[] interfaces)`: 根据类加载器和实现的接口获得代理类。

`Proxy` 类中有一个映射表，映射关系为：(`<ClassLoader>`,(`<Interfaces>`,`<ProxyClass>`) )，可以看出一级key为类加载器，根据这个一级key获得二级映射表，二级key为接口数组，因此可以看出：一个类加载器对象和一个接口数组确定了一个代理类。

我们写一个简单的例子来阐述 Java 实现动态代理的整个过程：

![image](http://bloghello.oursnail.cn/javabasic8-4.png)

`InvocationHandler` 接口中有方法：`invoke(Object proxy, Method method, Object[] args)`

这个函数是在代理对象调用任何一个方法时都会调用的，方法不同会导致第二个参数method不同，**第一个参数是代理对象**（表示哪个代理对象调用了method方法，传递来的是），**第二个参数是 Method 对象**（表示哪个方法被调用了），**第三个参数是指定调用方法的参数**。

动态生成的代理类具有几个特点：

* 继承 `Proxy` 类，并实现了在`Proxy.newProxyInstance()`中提供的接口数组。
* `public final`。
* 命名方式为 `$ProxyN`，其中N会慢慢增加，一开始是 `$Proxy1`，接下来是`$Proxy2`...
* 有一个参数为 `InvocationHandler` 的构造函数。这个从 `Proxy.newProxyInstance()` 函数内部的`clazz.getConstructor(new Class[] { InvocationHandler.class })` 可以看出。

Java 实现动态代理的缺点：因为 Java 的单继承特性（每个代理类都继承了 Proxy 类），只能针对接口创建代理类，不能针对类创建代理类。

> 不难发现，代理类的实现是有很多共性的（重复代码），动态代理的好处在于避免了这些重复代码，只需要关注操作。

## 小栗子

假设模拟一个场景，买衣服，正常情况所有人买这件衣服要100块钱。

定义一个销售接口：

![image](http://bloghello.oursnail.cn/javabasic8-5.png)

一个具体的实现类：

![image](http://bloghello.oursnail.cn/javabasic8-6.png)

那么正常情况大家都要花100才能买这件衣服。但是现在对会员做活动，会员打5折。怎么做呢？正常思维是：增加一个接口，甚至更糟的想法是修改一下这个实现类，都是不好的，那么我们是否想过这样的方案：新建一个新的类，让这个代理类去做相应的逻辑呢？既不用修改原来的代码，而且还很简单就能实现。

现在写一个代理类：

![image](http://bloghello.oursnail.cn/javabasic8-7.png)

那么调用的时候，一个是会员，一个是普通用户，根据身份调不同的方法即可：

![image](http://bloghello.oursnail.cn/javabasic8-8.png)

## Java 动态代理的内部实现

现在我们就会有一个问题： Java 是怎么保证代理对象调用的任何方法都会调用 `InvocationHandler` 的 `invoke()` 方法的？

这就涉及到动态代理的内部实现。假设有一个接口 `Subject`，且里面有 `int request(int i)` 方法，则生成的代理类大致如下：

![image](http://bloghello.oursnail.cn/javabasic8-9.png)

通过上面的方法就成功调用了 `invoke()` 方法，所以这是代理类中已经注定要去执行 `invoke()` 方法了。

有一篇文章比较生动地阐述了动态代理的含义：[Java帝国之动态代理](https://mp.weixin.qq.com/s?__biz=MzAxOTc0NzExNg==&mid=2665513926&idx=1&sn=1c43c5557ba18fed34f3d68bfed6b8bd&chksm=80d67b85b7a1f2930ede2803d6b08925474090f4127eefbb267e647dff11793d380e09f222a8#rd)
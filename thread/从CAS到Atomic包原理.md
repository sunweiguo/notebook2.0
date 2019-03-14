title: 从CAS到Atomic包原理
tag: java多线程
---
我们知道，volatile保证了可见性，但是不能保证原子性，在面对线程安全问题时，就显地力不从心，那么除了synchronized关键字外，还有什么方式可以实现线程安全更新呢？本文首先介绍CAS是什么，引出JUC下一个重要的包：Atomic包。
<!-- more -->

## 一、CAS简介

`CAS`（`Compare and Swap`），即比较并替换，实现并发算法时常用到的一种技术，`Doug lea`大神在java同步器中大量使用了`CAS`技术，鬼斧神工的实现了多线程执行的安全性。

`CAS`的思想很简单：三个参数，一个当前内存值V、旧的预期值A、即将更新的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回`true`，否则什么都不做，并返回`false`。

## 二、n++问题


![image](http://bloghello.oursnail.cn/thread9-1.jpg)

通过`javap -verbose Case`看看`add`方法的字节码指令：

![image](http://bloghello.oursnail.cn/thread9-2.jpg)

我们可以看到，`n++`被拆分成了下面几个指令：

- 执行`getfield`拿到原始`n`；
- 执行`iadd`进行加1操作；
- 执行`putfield`写把累加后的值写回`n`；


通过`volatile`修饰的变量可以保证线程之间的可见性，但并不能保证这3个指令的原子执行，在多线程并发执行下，无法做到线程安全，得到正确的结果，那么应该如何解决呢？

这里顺便提一下线程安全三个特性

- 原子性：提供了互斥访问，同一时刻只能有一个线程来对它进行操作。
- 可见性：一个线程对主内存的修改可以及时地被其他线程观察到。
- 有序性：一个线程观察其他线程中的指令的执行顺序，由于指令重排序的存在，该观察结果一般杂乱无序。

可以看到原子性是线程安全的一大特性。

## 三、解决方案一

在`add`方法加上`synchronized`修饰解决。

![image](http://bloghello.oursnail.cn/thread9-3.jpg)


这个方案当然可行，但是性能上差了点，还有其它方案么？

## 四、解决方案二

我们可不可以用一下乐观锁的思想呢？即不加锁，等真正要赋值的时候比较一下。

![image](http://bloghello.oursnail.cn/thread9-4.jpg)

当然了，这段代码如果真的在并发下执行，肯定出问题，只有把这整个过程变成一个原子操作才行，即同一时刻只有一个线程才能修改变量`a`。

如何实现呢？

我们注意到JUC下有个好东西，以`Atomic`打头的一些类。就可以很好地帮助我们实现对一个数加一减一的原子性操作。比如我们要安全地对`n`加一，可以这样做：

![image](http://bloghello.oursnail.cn/thread9-5.jpg)

下面就以`AtomicInteger`的实现为例，分析一下`CAS`是如何实现的。

![image](http://bloghello.oursnail.cn/thread9-6.jpg)

- `Unsafe`，是`CAS`的核心类，由于Java方法无法直接访问底层系统，需要通过本地（`native`）方法来访问，`Unsafe`相当于一个后门，基于该类可以直接操作特定内存的数据。
- 变量`valueOffset`，表示该变量值在内存中的偏移地址，因为`Unsafe`就是根据内存偏移地址获取数据的。
- 变量`value`用`volatile`修饰，保证了多线程之间的内存可见性。


看看`AtomicInteger`如何实现并发下的累加操作：

![image](http://bloghello.oursnail.cn/thread9-7.jpg)

假设线程`A`和线程`B`同时执行`getAndIncrement`操作（分别跑在不同CPU上）：

- 假设`AtomicInteger`里面的`value`原始值为0，即主内存中`AtomicInteger`的`value`为0，根据Java内存模型，线程A和线程B各自持有一份`value`的副本，值为0。
- 线程A通过`getIntVolatile(var1, var2)`拿到`value`值0，这时线程A被挂起。
- 线程B也通过`getIntVolatile(var1, var2)`方法获取到`value`值0，运气好，线程B没有被挂起，并执行`compareAndSwapInt`方法比较内存值也为0，成功修改内存值为1。
- 这时线程A恢复，执行`compareAndSwapInt`方法比较，发现自己手里的值(0)和内存的值(1)不一致，说明该值已经被其它线程提前修改过了，那只能重新来一遍了。
- 重新获取`value`值，因为变量`value`被`volatile`修饰，所以其它线程对它的修改，线程A总是能够看到，线程A继续执行`compareAndSwapInt`进行比较替换，直到成功。


整个过程中，利用`CAS`保证了对于`value`的修改的并发安全，继续深入看看`Unsafe`类中的`compareAndSwapInt`方法实现。

![image](http://bloghello.oursnail.cn/thread9-8.jpg)

我们看到是一个本地方法，并且在每个操作系统的具体实现都是不大一样的，这里我们就不再深究了。只要知道它的比较和替换是一个原子操作即可。


## 五、其他重要的Atomic类


##### 5.1 LongAdder

上面提到了`AtomicInteger`，那么必然也存在``AtomicLong`。用法和原理是一样的。

既然用`LongAddr`也可以，但是为什么不使用`AtomicLong`呢？换句话说，为什么`AtomicLong`可以实现，还要有`LongAddr`这个类呢？？？


`LongAddr`优点：我们从`AtomicInteger`这个类的实现看到，他是在一个死循环内不停地尝试修改目标值，直到修改成功。如果竞争不激烈的时候，修改成功的几率很高。否则修改失败的概率就会很高。在大量修改失败的时候，多次尝试，性能会受到一定的影响。

对于普通类型的`Long`和`Double`变量，JVM允许将64位的读操作和写操作拆成两个32位的操作。

> 我们知道`JUC`下面提供的原子类都是基于`Unsafe`类实现的，并由`Unsafe`来提供`CAS`的能力。`CAS` (`compare-and-swap`)本质上是由现代`CPU`在硬件级实现的原子指令，允许进行无阻塞，多线程的数据操作同时兼顾了安全性以及效率。`getAndAddLong`方法会以`volatile`的语义去读需要自增的域的最新值，然后通过`CAS`去尝试更新，正常情况下会直接成功后返回，但是在高并发下可能会同时有很多线程同时尝试这个过程，也就是说线程A读到的最新值可能实际已经过期了，因此需要在`while`循环中不断的重试，造成很多不必要的开销。

将`AtomicLong`核心数据`value`分离成一个数组，每个线程访问时，通过`hash`等算法，映射到其中一个数字进行计数。最终的计数结果则为这个数组的求和累加。其中热点数据`value`会被分离成多个单元的`cell`，每个`cell`独自维护内部的值，当前对象的实际值由`cell`累计合成。这样，热点就得到有效的分离并提高了并行度。 `LongAddr`在`AtomicLong`基础上将单点的更新压力分散到各个节点上。低并发时通过对`base`直接更新，得到与`AtomicLong`一样的性能。

<div class="tip">
缺陷：统计的时候，如果有并发更新，会有统计的误差，例如获取一个全局唯一的ID还是采用`AtomicLong`更好一点。
</div>


##### 5.2 AtomicReference

这个其实很简单，用法如下：

![image](http://bloghello.oursnail.cn/thread9-9.jpg)

其实这个方法实现的是对一个共享对象的原子性操作，保证对象更新的原子性。

##### 5.3 AtomicIntegerFieldUpdater

假设现在有这样的一个场景： 一百个线程同时对一个int对象进行修改，要求只能有一个线程可以修改。

可能有的同学会这么写：

![image](http://bloghello.oursnail.cn/thread9-10.jpg)

我们来分析一下，对于`volatile`变量，写的时候会将线程本地内存的数据刷新到主内存上，读的时候会将主内存的数据加载到本地内存里，所以可以保证可见行和单个读/写操作的原子性。

但是上例中先 

- 先判断:`!ischanged` 
- 再执行赋值操作：`ischanged=true`  

该组合操作就不能保证原子性了，也就是说线程A A1->A2 , 线程B B1->B2 (第一个操作为`volatile`读或者第二个操作为`volatile`写的时候，编译器不会对两个语句重排序，所以最后的执行顺序满足顺序一致性模型的)，但是最后的执行结果可能是A1->B1->A2->B2。不满足需求.

这种情况下，`AtomicIntegerFieldUpdater`就可以派上用场了。

![image](http://bloghello.oursnail.cn/thread9-11.jpg)

对于这个代码的理解可以用下面这个代码来：

![image](http://bloghello.oursnail.cn/thread9-12.jpg)

运行结果：

```
update success 1:200
update fail
```

用`AtomicIntegerFieldUpdater.newUpdater`指定类里面的属性。这里我们要更新`Test`类里面的`A`字段（必须是`volatile`且不是`static`对象）。`update.compareAndSet()`方法使用`cas`机制，每次提交的时候都比较下`test.a`是不是100，如果是，则更新。

注意，不能使用`final`变量，因为语义冲突。对于`AtomicIntegerFieldUpdater`和`AtomicLongFieldUpdater`只能修改`int`/`long`类型的字段，不能修改其包装类型（`Integer`/`Long`）。如果要修改包装类型就需要使用`AtomicReferenceFieldUpdater`。

##### 5.4 AtomicStampedReference

对于上面说的`AtomicInteger`等存在一个问题就是ABA问题。

ABA问题：其他线程将A改为B，又重新改为了A，本线程用期望值A与之进行比较，发现是相等的，则进行下面的操作。因为这个值已经被改变过，这就是ABA问题。


解决：用个版本号来控制，来防止ABA问题。


##### 5.5 AtomicBoolean

场景：若干个线程进来，但是这个方法只能执行一次。

![image](http://bloghello.oursnail.cn/thread9-13.jpg)

好了，其实`Atomic`包最核心的思想就是用无阻塞的`CAS`来代替锁实现高性能操作，是实现线程安全的一种可行方法，理解了`CAS`原理和他们的基本用法和场景使用，基本就可以了。
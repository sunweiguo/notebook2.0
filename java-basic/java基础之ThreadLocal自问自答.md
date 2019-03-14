title: java基础之ThreadLocal自问自答
tag: java基础
---

以问答的形式加深对threadlocal的理解，做到面试的镇定自若。

<!-- more -->

### 问题1：ThreadLocal了解吗？您能给我说说他的主要用途吗？

官方定位：`ThreadLocal`类用来**提供线程内部的局部变量**。这种变量在多线程环境下访问（通过`get`和`set`方法访问）时能保证各个线程的变量相对独立于其他线程内的变量。

简单归纳就是：

- `ThreadLocal`的作用是提供线程内的局部变量
- 这种变量在线程的生命周期内起作用
- 不同的线程之间不会相互干扰

### 问题2：ThreadLocal实现原理是什么，它是怎么样做到局部变量不同的线程之间不会相互干扰的？

通常，如果我不去看源代码的话，我猜`ThreadLocal`是这样子设计的：每个`ThreadLocal`类都创建一个`Map`，然后用线程的ID `threadID`作为`Map`的`key`，要存储的局部变量作为`Map`的`value`，这样就能达到各个线程的值隔离的效果。这是最简单的设计方法，JDK最早期的`ThreadLocal`就是这样设计的。

但是，JDK后面优化了设计方案，现时JDK8 `ThreadLocal`的设计是：每个`Thread`维护一个`ThreadLocalMap`哈希表，这个哈希表的`key`是`ThreadLocal`实例本身，`value`才是真正要存储的值`Object`。

![image](http://xiaozhao.oursnail.cn/iKjk3GS.png)

**那么为什么不用上面个设计呢？多简单啊！**

如果用Map来做的话，只能是用`thread`+`threadlocal`计算出来作为`key`，毕竟我存的不一定只有一个变量。那么不用他的时候，如何清理呢？只能是手动`remove`掉，但是一方面很麻烦，另一方面代码很丑陋，最后一方面是在`remove`的时候突然出现问题，那么就可能导致内存泄漏。

**新的设计的好处：**

当`Thread`销毁之后，对应的`ThreadLocalMap`也会随之销毁，能减少内存的使用。

假设当前`thread`一直活着（比如赖在线程池中），有些无用的`threadlocal`对象怎么清理呢？

`key`是一个软引用指向`ThreadLocal`实例，特性是下一次gc的时候就会被回收掉了，`ThreadLocalMap`中就会出现`key`为`null`的`Entry`.

`key`回收掉了，`value`值还在啊，这个怎么回收！！！

`ThreadLocal`的`get`和`set`方法每次调用时，如果发现当前的`entry`的`key`为`null`（也就是被回收掉了），最终会调用`expungeStaleEntry(int staleSlot)`方法，该方法会把哈希表当前位置的无用数据清理掉（当然还有别的操作）。

但是最佳实践还是每次使用完`ThreadLocal`，都调用它的`remove()`方法，清除数据,确保不会出现内存泄漏问题。

### 问题3：您能说说ThreadLocal常用操作的底层实现原理吗？如存储set(T value)，获取get()，删除remove()等操作。

具体的代码就不贴了，核心代码都已经看过了。这里简单总结一下。

- 调用`get()`
    + 获取当前线程`Thread`对象，进而获取此线程对象中维护的`ThreadLocalMap`对象。
    + 判断当前的`ThreadLocalMap`是否存在,如果存在，则以当前的`ThreadLocal` 为 `key`，调用`ThreadLocalMap`中的`getEntry`方法获取对应的存储实体 `e`。找到对应的存储实体 `e`，获取存储实体 `e` 对应的 `value`值，即为我们想要的当前线程对应此`ThreadLocal`的值，返回结果值。
    + 如果不存在，则证明此线程没有维护的`ThreadLocalMap`对象，调用`setInitialValue`方法进行初始化。返回`setInitialValue`初始化的值。
- 调用`set(T value)`
    + 获取当前线程`Thread`对象，进而获取此线程对象中维护的`ThreadLocalMap`对象。
    + 判断当前的`ThreadLocalMap`是否存在：
    + 如果存在，则调用`map.set`设置此实体`entry`。
    + 如果不存在，则调用`createMap`进行`ThreadLocalMap`对象的初始化，并将此实体`entry`作为第一个值存放至`ThreadLocalMap`中。
- 调用`remove()`
    + 获取当前线程`Thread`对象，进而获取此线程对象中维护的`ThreadLocalMap`对象。
    + 判断当前的`ThreadLocalMap`是否存在， 如果存在，则调用`map.remove`，以当前`ThreadLocal`为`key`删除对应的实体`entry`。
    
### 问题4：对ThreadLocal的常用操作实际是对线程Thread中的ThreadLocalMap进行操作，核心是ThreadLocalMap这个哈希表，你能谈谈ThreadLocalMap的内部底层实现吗?

`ThreadLocalMap`的底层实现是一个定制的自定义`HashMap`哈希表，核心组成元素有：

- `Entry[] table`：底层哈希表 `table`,必要时需要进行扩容，底层哈希表 `table.length` 长度必须是2的n次方。
- `int size`：实际存储键值对元素个数 `entries`
- `int threshold`：下一次扩容时的阈值，阈值 `threshold = len(table) * 2 / 3`。当 `size >= threshold` 时，遍历`table`并删除`key`为`null`的元素，如果删除后`size >= threshold*3/4`时，需要对`table`进行扩容

其中`Entry[] table`哈希表存储的核心元素是`Entry`，`Entry`包含：
- `ThreadLocal<?> k`：当前存储的`ThreadLocal`实例对象
- `Object value`：当前 `ThreadLocal` 对应储存的值`value`

需要注意的是，此`Entry`继承了弱引用 `WeakReference`，所以在使用`ThreadLocalMap`时，发现`key == null`，则意味着此`key ThreadLocal`不在被引用，需要将其从`ThreadLocalMap`哈希表中移除。

### 问题5：ThreadLocalMap中的存储实体Entry使用ThreadLocal作为key，但这个Entry是继承弱引用WeakReference的，为什么要这样设计，使用了弱引用WeakReference会造成内存泄露问题吗？

参考上一篇[文章](http://fourcolor.oursnail.cn/2019/02/20/java-basic/java%E5%9F%BA%E7%A1%80%E4%B9%8BThreadLocal%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E9%97%AE%E9%A2%98/)。

### 问题6：ThreadLocal和synchronized的区别?

`ThreadLocal`和`synchronized`关键字都用于处理多线程并发访问变量的问题，只是二者处理问题的角度和思路不同。

- `ThreadLocal`是一个`Java`类,通过对当前线程中的局部变量的操作来解决不同线程的变量访问的冲突问题。所以，`ThreadLocal`提供了线程安全的共享对象机制，每个线程都拥有其副本。
- `Java`中的`synchronized`是一个保留字，它依靠JVM的锁机制来实现临界区的函数或者变量的访问中的原子性。在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。此时，被用作“锁机制”的变量时多个线程共享的。
- 同步机制(`synchronized`关键字)采用了以“时间换空间”的方式，提供一份变量，让不同的线程排队访问。而`ThreadLocal`采用了“以空间换时间”的方式，为每一个线程都提供一份变量的副本，从而实现同时访问而互不影响.

### 问题7：ThreadLocal在现时有什么应用场景？

总的来说`ThreadLocal`主要是解决2种类型的问题：

- 解决并发问题：使用`ThreadLocal`代替`synchronized`来保证线程安全。同步机制采用了“以时间换空间”的方式，而`ThreadLocal`采用了“以空间换时间”的方式。前者仅提供一份变量，让不同的线程排队访问，而后者为每一个线程都提供了一份变量，因此可以同时访问而互不影响。
- 解决数据存储问题：`ThreadLocal`为变量在每个线程中都创建了一个副本，所以每个线程可以访问自己内部的副本变量，不同线程之间不会互相干扰。如一个`Parameter`对象的数据需要在多个模块中使用，如果采用参数传递的方式，显然会增加模块之间的耦合性。此时我们可以使用`ThreadLocal`解决。

一般的`Web`应用划分为展现层、服务层和持久层三个层次，在不同的层中编写对应的逻辑，下层通过接口向上层开放功能调用。在一般情况下，从接收请求到返回响应所经过的所有程序调用都同属于一个线程`ThreadLocal`是解决线程安全问题一个很好的思路，它通过为每个线程提供一个独立的变量副本解决了变量并发访问的冲突问题。在很多情况下，`ThreadLocal`比直接使用`synchronized`同步机制解决线程安全问题更简单，更方便，且结果程序拥有更高的并发性。


### 总结

- `ThreadLocal`提供线程内部的局部变量，在本线程内随时随地可取，隔离其他线程。
- `ThreadLocal`的设计是：每个`Thread`维护一个`ThreadLocalMap`哈希表，这个哈希表的`key`是`ThreadLocal`实例本身，`value`才是真正要存储的值`Object`。
- 对`ThreadLocal`的常用操作实际是对线程`Thread`中的`ThreadLocalMap`进行操作。
- `ThreadLocalMap`的底层实现是一个定制的自定义哈希表，`ThreadLocalMap`的阈值`threshold` = `底层哈希表table的长度 len * 2 / 3`，当实际存储元素个数`size` 大于或等于 阈值`threshold的 3/4` 时`size >= threshold*3/4`，则对底层哈希表数组`table`进行扩容操作。
- `ThreadLocalMap`中的哈希表`Entry[] table`存储的核心元素是`Entry`，存储的`key`是`ThreadLocal`实例对象，`value`是`ThreadLocal` 对应储存的值`value`。需要注意的是，此`Entry`继承了弱引用 `WeakReference`，所以在使用`ThreadLocalMap`时，发现`key == null`，则意味着此`key  ThreadLocal`不在被引用，需要将其从`ThreadLocalMap`哈希表中移除。
- `ThreadLocalMap`使用`ThreadLocal`的弱引用作为`key`，如果一个`ThreadLocal`没有外部强引用来引用它，那么系统 GC 的时候，这个`ThreadLocal`势必会被回收。所以，在`ThreadLocal`的`get()`,`set()`,`remove()`的时候都会清除线程`ThreadLocalMap`里所有`key`为`null`的`value`。如果我们不主动调用上述操作，则会导致内存泄露。
- 为了安全地使用`ThreadLocal`，必须要像每次使用完锁就解锁一样，在每次使用完`ThreadLocal`后都要调用`remove()`来清理无用的`Entry`。这在操作在使用线程池时尤为重要。
- `ThreadLocal`和`synchronized`的区别：同步机制(`synchronized`关键字)采用了以“时间换空间”的方式，提供一份变量，让不同的线程排队访问。而`ThreadLocal`采用了“以空间换时间”的方式，为每一个线程都提供一份变量的副本，从而实现同时访问而互不影响。
- `ThreadLocal`主要是解决2种类型的问题：A. 解决并发问题：使用`ThreadLocal`代替同步机制解决并发问题。B. 解决数据存储问题：如一个`Parameter`对象的数据需要在多个模块中使用，如果采用参数传递的方式，显然会增加模块之间的耦合性。此时我们可以使用`ThreadLocal`解决。


title: java基础之NIO
tag: java基础
---

为了限制篇幅，关于IO这一块的内容，已经从本笔记中移除，具体还是另外看笔记，这里主要还是介绍NIO。
<!-- more -->

## 一、NIO

非阻塞的输入/输出 (`NIO`) 库是在 `JDK 1.4` 中引入的。`NIO` 弥补了原来的 `I/O` 的不足，提供了高速的、面向块的 `I/O`。


## 1.1 阻塞I/O通信模型 

假如现在你对阻塞I/O已有了一定了解，我们知道阻塞I/O在调用`InputStream.read()`方法时是阻塞的，它会一直等到数据到来时（或超时）才会返回；同样，在调用`ServerSocket.accept()`方法时，也会一直阻塞到有客户端连接才会返回，每个客户端连接过来后，服务端都会启动一个线程去处理该客户端的请求。阻塞I/O的通信模型示意图如下：

![image](http://xiaozhao.oursnail.cn/%E9%98%BB%E5%A1%9EIO%E9%80%9A%E4%BF%A1%E6%A8%A1%E5%9E%8B.jpg)

缺点：

1. 当客户端多时，会创建大量的处理线程。且每个线程都要占用栈空间和一些CPU时间
2. 阻塞可能带来频繁的上下文切换，且大部分上下文切换可能是无意义的。

## 1.2 java NIO原理及通信模型 

下面是`java NIO`的工作原理：

1. 由一个专门的线程来处理所有的 IO 事件，并负责分发。 
2. 事件驱动机制：事件到的时候触发，而不是同步的去监视事件。 
3. 线程通讯：线程之间通过 `wait`,`notify` 等方式通讯。保证每次上下文切换都是有意义的。减少无谓的线程切换。 

![image](http://xiaozhao.oursnail.cn/NIO%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.jpg)

`Java NIO`的服务端只需启动一个专门的线程来处理所有的 `IO` 事件，这种通信模型是怎么实现的呢？`java NIO`采用了双向通道（`channel`）进行数据传输，而不是单向的流（`stream`），在通道上可以注册我们感兴趣的事件。一共有以下四种事件：


事件名 | 对应值
---|---
服务端接收客户端连接事件 | SelectionKey.OP_ACCEPT(16)
客户端连接服务端事件|SelectionKey.OP_CONNECT(8)
读事件|SelectionKey.OP_READ(1)
写事件|SelectionKey.OP_WRITE(4)

服务端和客户端各自维护一个管理通道的对象，我们称之为`selector`，该对象能检测一个或多个通道 (`channel`) 上的事件。我们以服务端为例，如果服务端的`selector`上注册了读事件，某时刻客户端给服务端发送了一些数据，阻塞`I/O`这时会调用`read()`方法阻塞地读取数据，而`NIO`的服务端会在`selector`中添加一个读事件。服务端的处理线程会轮询地访问`selector`，如果访问`selector`时发现有感兴趣的事件到达，则处理这些事件，如果没有感兴趣的事件到达，则处理线程会一直阻塞直到感兴趣的事件到达为止。下面是我理解的`java NIO`的通信模型示意图：

![image](http://xiaozhao.oursnail.cn/NIO%E9%80%9A%E4%BF%A1%E6%A8%A1%E5%9E%8B.jpg)


## 二、关于阻塞与非阻塞，同步与非同步的理解

我们都知道常见的IO有四种方式，同步阻塞，同步非阻塞，异步阻塞，异步非阻塞。然而对于同步和阻塞的理解却一直很模糊。

#### 2.1 同步与异步

- 所谓同步就是一个任务的完成需要依赖另外一个任务时，只有等待被依赖的任务完成后，依赖的任务才能算完成，这是一种可靠的任务序列。要么成功都成功，失败都失败，两个任务的状态可以保持一致。
- 而异步是不需要等待被依赖的任务完成，只是通知被依赖的任务要完成什么工作，依赖的任务也立即执行，只要自己完成了整个任务就算完成了。至于被依赖的任务最终是否真正完成，依赖它的任务无法确定，所以它是不可靠的任务序列。
- 我们可以用打电话（同步）和发短信（异步）来很好的比喻同步与异步操作。


#### 2.2 阻塞和非阻塞

- 阻塞就是 CPU 停下来等待一个慢的操作完成 CPU 才接着完成其它的事。
- 非阻塞就是在这个慢的操作在执行时 CPU 去干其它别的事，等这个慢的操作完成时，CPU 再接着完成后续的操作。
- 虽然表面上看非阻塞的方式可以明显的提高 CPU 的利用率，但是也带了另外一种后果就是系统的线程切换增加。


#### 2.3 什么是阻塞IO？什么是非阻塞IO？

在了解阻塞IO和非阻塞IO之前，先看下一个具体的IO操作过程是怎么进行的。

通常来说，IO操作包括：对硬盘的读写、对socket的读写以及外设的读写。

当用户线程发起一个IO请求操作（本文以读请求操作为例），内核会去查看要读取的数据是否就绪，对于阻塞IO来说，如果数据没有就绪，则会一直在那等待，直到数据就绪；对于非阻塞IO来说，如果数据没有就绪，则会返回一个标志信息告知用户线程当前要读的数据没有就绪。当数据就绪之后，便将数据拷贝到用户线程，这样才完成了一个完整的IO读请求操作，也就是说一个完整的IO读请求操作包括两个阶段：

- 查看数据是否就绪；
- 进行数据拷贝（内核将数据拷贝到用户线程）。

那么阻塞（blocking IO）和非阻塞（non-blocking IO）的区别就在于<font color=#ff0000>第一个阶段</font>，如果数据没有就绪，在查看数据是否就绪的过程中是一直等待，还是直接返回一个标志信息。

<font color=#ff0000>Java中传统的IO都是阻塞IO，比如通过socket来读数据，调用read()方法之后，如果数据没有就绪，当前线程就会一直阻塞在read方法调用那里，直到有数据才返回；而如果是非阻塞IO的话，当数据没有就绪，read()方法应该返回一个标志信息，告知当前线程数据没有就绪，而不是一直在那里等待。</font>


#### 2.4 什么是同步IO？什么是异步IO？

我们知道了，阻塞和非阻塞是判断数据是否就绪时如何处理，即IO操作的第一阶段。

那么什么是同步IO和异步IO呢？

我们知道，同步是打电话，异步是发短信，打电话需要等到电话通了才能进行下一步，发短信就不用操心那么多了，我发出去就行了，至于什么时候发送、如何发送以及如何保证我这个短信一定能发出去，我是不管的。

同步IO即 如果一个线程请求进行IO操作，在IO操作完成之前，该线程会被阻塞；而异步IO为 如果一个线程请求进行IO操作，IO操作不会导致请求线程被阻塞。

> 描述的是用户线程与内核的交互方式：

- 同步是指用户线程发起 I/O 请求后需要等待或者轮询内核 I/O操作完成后才能继续执行；
- 异步是指用户线程发起I/O请求后仍继续执行，当内核I/O操作完成后会通知用户线程，或者调用用户线程注册的回调函数。

## 三、Channel（通道）
通道，顾名思义，就是通向什么的道路，为某个提供了渠道。在传统IO中，我们要读取一个文件中的内容，通常是像下面这样读取的：

![image](http://bloghello.oursnail.cn/javabbasic13-1.png)


这里的`InputStream`实际上就是为读取文件提供一个通道的。
因此可以将`NIO` 中的`Channel`同传统IO中的`Stream`来类比，但是要注意，传统IO中，`Stream`是单向的，比如`InputStream`只能进行读取操作，`OutputStream`只能进行写操作。而`Channel`是双向的，既可用来进行读操作，又可用来进行写操作。

通道包括以下类型：

- `FileChannel`：从文件中读写数据；
- `DatagramChannel`：通过 UDP 读写网络中数据；
- `SocketChannel`：通过 TCP 读写网络中数据；
- `ServerSocketChannel`：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 `SocketChannel`


下面给出通过`FileChannel`来向文件中写入数据的一个例子：

![image](http://bloghello.oursnail.cn/javabasic13-2.png)


## 四、Buffer（缓冲区）

`Buffer`（缓冲区），是`NIO`中非常重要的一个东西，在`NIO`中所有数据的读和写都离不开`Buffer`。比如上面的一段代码中，读取的数据时放在`byte`数组当中，而在`NIO`中，读取的数据只能放在`Buffer`中。同样地，写入数据也是先写入到`Buffer`中。

![image](http://xiaozhao.oursnail.cn/NIOBuffer.jpg)

上面的图描述了从一个客户端向服务端发送数据，然后服务端接收数据的过程。客户端发送数据时，必须先将数据存入`Buffer`中，然后将`Buffer`中的内容写入通道。服务端这边接收数据必须通过`Channel`将数据读入到`Buffer`中，然后再从`Buffer`中取出数据来处理。

缓冲区实质上是一个数组，但它不仅仅是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。

缓冲区包括以下类型：

- ByteBuffer
- CharBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer

如果是对于文件读写，上面几种`Buffer`都可能会用到。但是对于网络读写来说，用的最多的是`ByteBuffer`。

## 五、缓冲区状态变量

- capacity：最大容量；
- position：当前已经读写的字节数；
- limit：还可以读写的字节数。

状态变量的改变过程举例：

① 新建一个大小为 8 个字节的缓冲区，此时 `position` 为 0，而 `limit = capacity = 8`。`capacity` 变量不会改变，下面的讨论会忽略它。

![image](http://xiaozhao.oursnail.cn/1bea398f-17a7-4f67-a90b-9e2d243eaa9a.png)

② 从输入通道中读取 5 个字节数据写入缓冲区中，此时 `position` 移动设置为 5，`limit` 保持不变。

![image](http://xiaozhao.oursnail.cn/80804f52-8815-4096-b506-48eef3eed5c6.png)

③ 在将缓冲区的数据写到输出通道之前，需要先调用 `flip()` 方法，这个方法将 `limit` 设置为当前 `position`，并将 `position` 设置为 0。

> buffer中的flip方法涉及到bufer中的Capacity,Position和Limit三个概念。其中Capacity在读写模式下都是固定的，就是我们分配的缓冲大小,Position类似于读写指针，表示当前读(写)到什么位置,Limit在写模式下表示最多能写入多少数据，此时和Capacity相同，在读模式下表示最多能读多少数据，此时和缓存中的实际数据大小相同。在写模式下调用flip方法，那么limit就设置为了position当前的值(即当前写了多少数据),postion会被置为0，以表示读操作从缓存的头开始读。也就是说调用flip之后，读写指针指到缓存头部，并且设置了最多只能读出之前写入的数据长度(而不是整个缓存的容量大小)。

![image](http://xiaozhao.oursnail.cn/952e06bd-5a65-4cab-82e4-dd1536462f38.png)

④ 从缓冲区中取 4 个字节到输出缓冲中，此时 `position` 设为 4。

![image](http://xiaozhao.oursnail.cn/b5bdcbe2-b958-4aef-9151-6ad963cb28b4.png)

⑤ 最后需要调用 `clear()` 方法来清空缓冲区，此时 `position` 和 `limit` 都被设置为最初位置。

![image](http://xiaozhao.oursnail.cn/67bf5487-c45d-49b6-b9c0-a058d8c68902.png)

## 六、文件 NIO 实例

以下展示了使用 NIO 快速复制文件的实例：

![image](http://bloghello.oursnail.cn/javabasic13-3.png)

## 七、Selector（选择器）

可以说它是`NIO`中最关键的一个部分，`Selector`的作用就是用来轮询每个注册的`Channel`，一旦发现`Channel`有注册的事件发生，便获取事件然后进行处理。

![image](http://xiaozhao.oursnail.cn/4d930e22-f493-49ae-8dff-ea21cd6895dc.png)

用单线程处理一个`Selector`，然后通过`Selector.select()`方法来获取到达事件，在获取了到达事件之后，就可以逐个地对这些事件进行响应处理。

因为创建和切换线程的开销很大，因此使用一个线程来处理多个事件而不是一个线程处理一个事件具有更好的性能。


下面从编程的角度具体来看看选择器是如何实现的。

### 7.1 创建选择器

```java
Selector selector = Selector.open();
```

### 7.2 将通道注册到选择器上

```java
ServerSocketChannel ssChannel = ServerSocketChannel.open();
ssChannel.configureBlocking(false);
ssChannel.register(selector, SelectionKey.OP_ACCEPT);
```

通道必须配置为非阻塞模式，否则使用选择器就没有任何意义了，因为如果通道在某个事件上被阻塞，那么服务器就不能响应其它事件，必须等待这个事件处理完毕才能去处理其它事件，显然这和选择器的作用背道而驰。

在将通道注册到选择器上时，还需要指定要注册的具体事件，主要有以下几类：

- `SelectionKey.OP_CONNECT`
- `SelectionKey.OP_ACCEPT`
- `SelectionKey.OP_READ`
- `SelectionKey.OP_WRITE`

它们在 `SelectionKey` 的定义如下：

```java
public static final int OP_READ = 1 << 0;
public static final int OP_WRITE = 1 << 2;
public static final int OP_CONNECT = 1 << 3;
public static final int OP_ACCEPT = 1 << 4;
```

可以看出每个事件可以被当成一个位域，从而组成事件集整数。例如：

```java
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

### 7.3 监听事件

```java
int num = selector.select();
```

使用 `select()` 来监听事件到达，它会一直阻塞直到有至少一个事件到达。

### 7.4 获取到达的事件

```java
Set<SelectionKey> keys = selector.selectedKeys();
Iterator<SelectionKey> keyIterator = keys.iterator();
while (keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if (key.isAcceptable()) {
        // ...
    } else if (key.isReadable()) {
        // ...
    }
    keyIterator.remove();
}
```

### 7.5 事件循环

因为一次 `select()` 调用不能处理完所有的事件，并且服务器端有可能需要一直监听事件，因此服务器端处理事件的代码一般会放在一个死循环内。

```java
while (true) {
    int num = selector.select();
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = keys.iterator();
    while (keyIterator.hasNext()) {
        SelectionKey key = keyIterator.next();
        if (key.isAcceptable()) {
            // ...
        } else if (key.isReadable()) {
            // ...
        }
        keyIterator.remove();
    }
}
```


## 八、流与块

`I/O` 与 `NIO` 最重要的区别是数据打包和传输的方式，`I/O` 以流的方式处理数据，而 `NIO` 以块的方式处理数据。

面向流的 `I/O` 一次处理一个字节数据，一个输入流产生一个字节数据，一个输出流消费一个字节数据。为流式数据创建过滤器非常容易，链接几个过滤器，以便每个过滤器只负责复杂处理机制的一部分。不利的一面是，面向流的 `I/O` 通常相当慢。

面向块的 `I/O` 一次处理一个数据块，按块处理数据比按流处理数据要快得多。但是面向块的 `I/O` 缺少一些面向流的 `I/O` 所具有的优雅性和简单性。

`I/O` 包和 `NIO` 已经很好地集成了，`java.io.*` 已经以 `NIO` 为基础重新实现了，所以现在它可以利用 `NIO` 的一些特性。例如，`java.io.*` 包中的一些类包含以块的形式读写数据的方法，这使得即使在面向流的系统中，处理速度也会更快。

## 九、一个完整 NIO 实例

```java
public class NIOServer {

    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();

        ServerSocketChannel ssChannel = ServerSocketChannel.open();
        ssChannel.configureBlocking(false);
        ssChannel.register(selector, SelectionKey.OP_ACCEPT);

        ServerSocket serverSocket = ssChannel.socket();
        InetSocketAddress address = new InetSocketAddress("127.0.0.1", 8888);
        serverSocket.bind(address);

        while (true) {
            selector.select();
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = keys.iterator();
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                if (key.isAcceptable()) {
                    ServerSocketChannel ssChannel1 = (ServerSocketChannel) key.channel();
                    // 服务器会为每个新连接创建一个 SocketChannel
                    SocketChannel sChannel = ssChannel1.accept();
                    sChannel.configureBlocking(false);
                    // 这个新连接主要用于从客户端读取数据
                    sChannel.register(selector, SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    SocketChannel sChannel = (SocketChannel) key.channel();
                    System.out.println(readDataFromSocketChannel(sChannel));
                    sChannel.close();
                }
                keyIterator.remove();
            }
        }
    }

    private static String readDataFromSocketChannel(SocketChannel sChannel) throws IOException {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        StringBuffer data = new StringBuffer();
        while (true) {
            buffer.clear();
            int n = sChannel.read(buffer);
            if (n == -1)
                break;
            buffer.flip();
            int limit = buffer.limit();
            char[] dst = new char[limit];
            for (int i = 0; i < limit; i++)
                dst[i] = (char) buffer.get(i);
            data.append(dst);
            buffer.clear();
        }
        return data.toString();
    }
}
```

```java
public class NIOClient {

    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1", 8888);
        OutputStream out = socket.getOutputStream();
        String s = "hello world";
        out.write(s.getBytes());
        out.close();
    }
}
```



## 十、NIO和IO的主要区别

IO|NIO
--|--
面向流|	面向缓冲
阻塞IO|	非阻塞IO
无|	选择器

- 面向流与面向缓冲

 > Java IO和NIO之间第一个最大的区别是，IO是面向流的，NIO是面向缓冲区的。 Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方。此外，它不能前后移动流中的数据。如果需要前后移动从流中读取的数据，需要先将它缓存到一个缓冲区。 Java NIO的缓冲导向方法略有不同。数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动。这就增加了处理过程中的灵活性。但是，还需要检查是否该缓冲区中包含所有您需要处理的数据。而且，需确保当更多的数据读入缓冲区时，不要覆盖缓冲区里尚未处理的数据。

- 阻塞与非阻塞IO

 > Java IO的各种流是阻塞的。这意味着，当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了。Java NIO的非阻塞模式，使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。 非阻塞写也是如此。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。 线程通常将非阻塞IO的空闲时间用于在其它通道上执行IO操作，所以一个单独的线程现在可以管理多个输入和输出通道（channel）。

- 选择器（Selectors）

 > Java NIO的选择器允许一个单独的线程来监视多个输入通道，你可以注册多个通道使用一个选择器，然后使用一个单独的线程来“选择”通道：这些通道里已经有可以处理的输入，或者选择已准备写入的通道。这种选择机制，使得一个单独的线程很容易来管理多个通道。
 

## 十一、总结

- `NIO`其实实现的是一个IO的多路复用，用`select`来同时监听多个`channel`，本质上还是同步阻塞的，需要`select`不断监听端口。但是对于IO各个通道来说就是可以看做是异步。
- 基本可以认为 “NIO = I/O多路复用 + 非阻塞式I/O”，大部分情况下是单线程，但也有超过一个线程实现NIO的情况
- 我们可以用打电话（同步）和发短信（异步）来很好的比喻同步与异步操作
- 阻塞就是 CPU 停下来等待一个慢的操作完成 CPU 才接着完成其它的事。
- 非阻塞就是在这个慢的操作在执行时 CPU 去干其它别的事，等这个慢的操作完成时，CPU 再接着完成后续的操作。两种方式各有优劣。
- 传统IO中，`Stream`是单向的，比如`InputStream`只能进行读取操作，`OutputStream`只能进行写操作。而`Channel`是双向的，既可用来进行读操作，又可用来进行写操作。
- 在`NIO`中，读取的数据只能放在`Buffer`中。同样地，写入数据也是先写入到`Buffer`中。缓冲区有三个状态变量：`capacity`：最大容量；`position`：当前已经读写的字节数；`limit`：还可以读写的字节数。
- Selector的作用就是用来轮询每个注册的Channel，一旦发现Channel有注册的事件发生，便获取事件然后进行处理.
- NIO和IO的主要区别。
- NIO适用场景

> 服务器需要支持超大量的长时间连接。比如10000个连接以上，并且每个客户端并不会频繁地发送太多数据。例如总公司的一个中心服务器需要收集全国便利店各个收银机的交易信息，只需要少量线程按需处理维护的大量长期连接。

- BIO适用场景

> 适用于连接数目比较小，并且一次发送大量数据的场景，这种方式对服务器资源要求比较高，并发局限于应用中。




参考：

- [IO - 同步，异步，阻塞，非阻塞](https://blog.csdn.net/historyasamirror/article/details/5778378)
- [Java NIO：浅析I/O模型](https://www.cnblogs.com/dolphin0520/p/3916526.html)
- [NIO与传统IO的区别](https://blog.csdn.net/shimiso/article/details/24990499)
- [Java NIO：NIO概述](https://troywu0.gitbooks.io/spark/content/java-io%E6%B5%81.html)
- [Java I/O](https://cyc2018.github.io/CS-Notes/#/notes/Java%20IO)
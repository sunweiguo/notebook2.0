title: java基础之异常
tag: java基础
---

在开发中，异常处理是一个不可绕开的话题，我们对于异常的处理已经非常熟练了，对于异常本身的概念、用法等不再赘述了，直接结合面试问题来加深对异常的理解吧。
<!-- more -->

`Throwable` 可以用来表示任何可以作为异常抛出的类，分为两种： `Error` 和 `Exception`。

![image](http://xiaozhao.oursnail.cn/%E5%BC%82%E5%B8%B8%E7%BB%A7%E6%89%BF%E6%A0%91.png)


## 1. 什么是Java异常

异常是发生在程序执行过程中阻碍程序正常执行的错误事件。比如：用户输入错误数据、硬件故障、网络阻塞等都会导致出现异常。

只要在Java语句执行中产生了异常，**一个异常对象就会被创建**，JRE就会试图寻找异常处理程序来处理异常。如果有合适的异常处理程序，异常对象就会被异常处理程序接管，否则，将引发运行环境异常，JRE终止程序执行。

Java异常处理框架只能处理运行时错误，编译错误不在其考虑范围之内。

## 2. Error和Exception的区别

Error：是程序无法处理的错误，表示运行应用程序中较严重问题。大多数错误与代码编写者执行的操作无关，而表示代码运行时 JVM出现的问题。

例如，Java虚拟机运行错误，当 JVM 不再有继续执行操作所需的内存资源时，将出现`OutOfMemoryError`。这些异常发生时，Java虚拟机一般会选择线程终止。

## 3. Java异常处理中有哪些关键字？

- `throw`:有时我们需要显式地创建并抛出异常对象来终止程序的正常执行。`throw`关键字用来抛出并处理运行时异常。
- `throws`:当我们抛出任何“被检查的异常(`checked exception`)”并不处理时，需要在方法签名中使用关键字`throws`来告知调用程序此方法可能会抛出的异常。调用方法可能会处理这些异常，或者同样用`throws`来将异常传给上一级调用方法。`throws`关键字后可接多个潜在异常，甚至是在`main()`中也可以使用`throws`。
- `try-catch`:我们在代码中用`try-catch`块处理异常。当然，一个`try`块之后可以有多个`catch`子句，`try-catch`块也能嵌套。每个`catch`块必须接受一个（且仅有一个）代表异常类型的参数。
- `finally`:`finally`块是可选的，并且只能配合`try-catch`一起使用。虽然异常终止了程序的执行，但是还有一些打开的资源没有被关闭，因此，我们能使用`finally`进行关闭。不管异常有没有出现，`finally`块总会被执行。


## 4. 描述一下异常的层级

- `Throwable`是所有异常的父类，它有两个直接子对象`Error`,`Exception`，其中`Exception`又被继续划分为“被检查的异常(`checked exception`)”和”运行时的异常（`runtime exception`,即不受检查的异常）”。 `Error`表示编译时和系统错误，通常不能预期和恢复，比如硬件故障、JVM崩溃、内存不足等。
- 被检查的异常（`Checked exception`）在程序中能预期，并要尝试修复，如`FileNotFoundException`。我们必须捕获此类异常，并为用户提供有用信息和合适日志来进行调试。`Exception`是所有被检查的异常的父类。
- 运行时异常（`Runtime Exception`）又称为不受检查异常，源于糟糕的编程。比如我们检索数组元素之前必须确认数组的长度，否则就可能会抛出`ArrayIndexOutOfBoundException`运行时异常。`RuntimeException`是所有运行时异常的父类。


## 5. 描述Java 7 ARM(Automatic Resource Management，自动资源管理)特征和多个catch块的使用

如果一个`try`块中有多个异常要被捕获，`catch`块中的代码会变丑陋的同时还要用多余的代码来记录异常。有鉴于此，Java 7的一个新特征是：一个`catch`子句中可以捕获多个异常。示例代码如下：


```java
catch(IOException | SQLException | Exception ex){
     logger.error(ex);
     throw new MyException(ex.getMessage());
}
```
大多数情况下，当忘记关闭资源或因资源耗尽出现运行时异常时，我们只是用`finally`子句来关闭资源。这些异常很难调试，我们需要深入到资源使用的每一步来确定是否已关闭。因此，Java 7用`try-with-resources`进行了改进：在`try`子句中能创建一个资源对象，当程序的执行完`try-catch`之后，运行环境自动关闭资源。

> 利用Try-Catch-Finally管理资源（旧的代码风格）：
```java
private static void printFile() throws IOException {
    InputStream input = null;

    try {
        input = new FileInputStream("file.txt");//可能发生异常1

        int data = input.read();//可能发生异常2
        while(data != -1){
            System.out.print((char) data);
            data = input.read();
        }
    } finally {
        if(input != null){
            input.close();//可能发生异常3
        }
    }
}
```

假设`try`语句块抛出一个异常，然后`finally`语句块被执行。同样假设`finally`语句块也抛出了一个异常。那么哪个异常会根据调用栈往外传播？

<font color=#ff0000>即使`try`语句块中抛出的异常与异常传播更相关，最终还是`finally`语句块中抛出的异常会根据调用栈向外传播。</font>

> 在java7中，对于上面的例子可以用try-with-resource 结构这样写：

```java
private static void printFileJava7() throws IOException {

    try(FileInputStream input = new FileInputStream("file.txt")) {

        int data = input.read();
        while(data != -1){
            System.out.print((char) data);
            data = input.read();
        }
    }
}
```
<font color=#ff0000>当`try`语句块运行结束时，`FileInputStream` 会被自动关闭。这是因为`FileInputStream` 实现了java中的`java.lang.AutoCloseable`接口。所有实现了这个接口的类都可以在`try-with-resources`结构中使用。</font>

**当`try-with-resources`结构中抛出一个异常，同时`FileInputStream`被关闭时（调用了其`close`方法）也抛出一个异常，`try-with-resources`结构中抛出的异常会向外传播，而`FileInputStream`被关闭时抛出的异常被抑制了**。

> 你可以在块中使用多个资源而且这些资源都能被自动地关闭。下面是例子：


```java
private static void printFileJava7() throws IOException {

    try(  FileInputStream     input         = new FileInputStream("file.txt");
          BufferedInputStream bufferedInput = new BufferedInputStream(input)
    ) {

        int data = bufferedInput.read();
        while(data != -1){
            System.out.print((char) data);
    data = bufferedInput.read();
        }
    }
}
```
这些资源将按照他们被创建顺序的**逆序**来关闭。首先`BufferedInputStream` 会被关闭，然后`FileInputStream`会被关闭。

这个`try-with-resources`结构里不仅能够操作java内置的类。你也可以在自己的类中实现`java.lang.AutoCloseable`接口，然后在`try-with-resources`结构里使用这个类。

`AutoClosable` 接口仅仅有一个方法，接口定义如下：


```java
public interface AutoClosable {

    public void close() throws Exception;
}
```
任何实现了这个接口的方法都可以在`try-with-resources`结构中使用。下面是一个简单的例子：

```java
public class MyAutoClosable implements AutoCloseable {

    public void doIt() {
        System.out.println("MyAutoClosable doing it!");
    }

    @Override
    public void close() throws Exception {
        System.out.println("MyAutoClosable closed!");
    }
}
```
`doIt()`是方法不是`AutoClosable` 接口中的一部分，之所以实现这个方法是因为我们想要这个类除了关闭方法外还能做点其他事。

下面是`MyAutoClosable` 在`try-with-resources`结构中使用的例子：


```java
private static void myAutoClosable() throws Exception {

    try(MyAutoClosable myAutoClosable = new MyAutoClosable()){
        myAutoClosable.doIt();
    }
}
```

运行结果：

```
MyAutoClosable doing it!
MyAutoClosable closed!
```
通过上面这些你可以看到，不论`try-catch`中使用的资源是自己创造的还是java内置的类型，`try-with-resources`都是一个能够确保资源能被正确地关闭的强大方法。


## 6. 在Java中throw与throws关键字之间的区别？

throws用于在方法签名中声明此方法可能抛出的异常，而throw关键字则是中断程序的执行并移交异常对象到运行时进行处理。

## 7. 在Java中怎么写自定义的异常？

我们能继承Exception类或其任何子类来实现自己的自定义异常类。这自定义异常类可以有自己变量和方法来传递错误代码或其它异常相关信息来处理异常。


```java
@Data
public class HappyBikeException extends RuntimeException{
    private Integer code = ResponseEnum.ERROR.getCode();

    public HappyBikeException(Integer code,String msg){
        super(msg);
        this.code = code;
    }

    public HappyBikeException(String msg){
        super(msg);
    }

}
```

## 8. Java中final,finally,finalize的区别？

这是一个垃圾问题，很想删除掉，但是考虑到新手，还是保留一下吧，至少从单词上有那么一点点像。

`final`和`finally`在Java中是关键字，而finalize则是一个方法。

`final`关键字使得类变量不可变，避免类被其它类继承或方法被重写。`finally`跟`try-catch`块一起使用，即使是出现了异常，其子句总会被执行，通常，`finally`子句用来关闭相关资源。`finally`方法中的对象被销毁之前会被垃圾回收。

## 9. 在main方法抛出异常时发生了什么？

答：当`main`方法抛出异常时，Java运行时间终止并在控制台打印异常信息和栈轨迹。

## 10. catch子句能为空吗？

`catch`后面括号里面不能为空。

答：可以有空的`catch`子句，但那是最糟糕的编程，因为那样的话，异常即使被捕获，我们也得不到任何的有用信息，对于调试来说会是个噩梦，因此，编程时永远不要有空的`catch`子句。`Catch`子句中至少要包含一个日志语句输出到控制台或保存到日志文件中。

## 11. 提供一些Java异常处理的最佳实践。

- 使用具体的异常方便调试
- 程序中早点抛出异常
- 捕获异常后让调用者处理异常
- 使用Java 7 ARM功能确保资源关闭或者用finally子句正确地关闭它们
- 为了调试需要总是记录异常信息
- 用多个catch子句实现更完全的关闭
- 你自己的应用API中用自定义的异常来抛出单种类型异常
- 遵循命名规定，以异常结束
- 在Javadoc中用@throws来标注方法抛出的异常
- **处理异常是有花销的，因此只有在必要时才抛出**。否则，你会扑空或毫无收获。

## 12. try、catch、finally三个语句块应注意的问题

- try、catch、finally三个语句块均不能单独使用，三者可以组成 try...catch...finally、try...catch、try...finally三种结构，catch语句可以有一个或多个，finally语句最多一个。
- try、catch、finally三个代码块中变量的作用域为代码块内部，分别独立而不能相互访问。如果要在三个块中都可以访问，则需要将变量定义到这些块的外面。
- 多个catch块时候，只会匹配其中一个异常类并执行catch块代码，而不会再执行别的catch块，并且匹配catch语句的顺序是由上到下。
- 无论程序是否有异常，并且无论之间try-catch是否顺利执行完毕，都会执行finally语句。在以下特殊情况下，finally块不会执行：在finally语句块中发生异常；在前面代码中使用了System.exit()退出程序；程序所在线程死亡；关闭cpu。
- **⭐当程序执行try块，catch块时遇到return语句或者throw语句，这两个语句都会导致该方法立即结束，所以系统并不会立即执行这两个语句，而是去寻找该异常处理流程中的finally块，如果没有finally块，程序立即执行return语句或者throw语句，方法终止。如果有finally块，系统立即开始执行finally块，只有当finally块执行完成后，系统才会再次跳回来执行try块、catch块里的return或throw语句，如果finally块里也使用了return或throw等导致方法终止的语句，则finally块已经终止了方法，不用再跳回去执行try块、catch块里的任何代码了。**

## 13. 解释Java中的异常处理流程

![image](http://xiaozhao.oursnail.cn/%E5%BC%82%E5%B8%B8%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png)

## 异常处理完成以后，Exception对象会发生什么变化？

Exception对象会在下一个垃圾回收过程中被回收掉。

## 请写出 5 种常见到的runtime exception。

- `NullPointerException`：当操作一个空引用时会出现此错误。
- `NumberFormatException`：数据格式转换出现问题时出现此异常。
- `ClassCastException`：强制类型转换类型不匹配时出现此异常。
- `ArrayIndexOutOfBoundsException`：数组下标越界，当使用一个不存在的数组下标时出现此异常。
- `ArithmeticException`：数学运行错误时出现此异常


参考：
- http://www.importnew.com/7383.html
- http://www.importnew.com/7541.html
- http://www.importnew.com/7820.html
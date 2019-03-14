title: java基础之泛型下
tag: java基础
---

本节继续讨论泛型相关的知识。

<!-- more -->


## 6、泛型上下边界

我们再来讨论讨论通配符。

通配符有2种：

- 无边界通配符，用`<?>`表示。
- 有边界通配符，用`<? extends Object>`或者`<? super Object>`来表示。（`Object`仅仅是一个示例）

##### 6.1 无边界


```java
List<?> list = new ArrayList<String>(); // 合法
List<?> list = new ArrayList<?>(); // 不合法
List<String> list = new ArrayList<?>(); // 不合法
```

对于带有通配符的引用变量，是不能调用具有与泛型参数有关的方法的。

```
List<?> list = new ArrayList<String>();
list.add(1); // 编译不通过
list.get(0); // 编译通过
int size = list.size(); // 由于size()方法中不含泛型参数，所以可以在通配符变量中调用
```
总结起来，无边界通配符主要用做引用，可以调用与泛型参数无关的方法，不能调用参数中包含泛型参数的方法。

##### 6.2 有边界

在使用泛型的时候，我们还可以为传入的泛型类型实参进行上下边界的限制，如：类型实参只准传入某种类型的父类或某种类型的子类。

- 上边界通配，用<? extends 类型>表示。其语法为：


```java
List<? extends 类型1> x = new ArrayList<类型2>();
```

其中，类型2就只能是类型1或者是类型1的子类。下面代码验证合法性。

```java
List<? extends Number> x = new ArrayList<Integer>(); //由于Integer是Number的子类，这是合法的
List<? extends Number> x = new ArrayList<String>();  //由于String不是Number的子类，这是不合法的
```

- 下边界通配，用<? super 类型>表示。其语法为：

```java
List<? super 类型1> x = new ArrayList<类型2>();
```

其中，类型2就只能是类型1或者是类型1的超类。下面代码有验证合法性。

```java
List<? super Integer> x = new ArrayList<Number>(); //由于Number是Integer的超类，这是合法的
List<? super Integer> x = new ArrayList<String>();  //由于String不是Integer的超类，这是不合法的
```

那么到底什么时候使用下边界通配，什么时候使用上边界通配呢？首先考虑一下怎样才能保证不会发生运行时异常，这是泛型要解决的首要问题，通过前面的内容可以看到，任何可能导致类型转换异常的操作都无法编译通过。

- ⭐上边界通配：可以保证存放的实际对象至多是上边界指定的类型，那么在读取对象时，我们总是可以放心地将对象赋予上边界类型的引用。

```java
List<Integer> list1 = new ArrayList<Integer>();
list1.add(1);
List<? extends Number> list2 = list1;
Number a = list2.get(0); // 编译通过
```

- ⭐下边界通配：可以保证存放的实际对象至少是下边界指定的类型，那么在存入对象时，我们总是可以放心地将下边界类型的对象存入泛型对象中。


```java
List<? super Integer> list3 = new ArrayList<Number>();
list3.add(1);
list3.add(2);
```

> 总结：
- 如果你想从一个数据类型里获取数据，使用 ? extends 通配符。
- 如果你想把对象写入一个数据结构里，使用 ? super 通配符。
- 如果你既想存，又想取，那就别用通配符。

对于泛型方法添加上下边界：

```java
//在泛型方法中添加上下边界限制的时候，必须在权限声明与返回值之间的<T>上添加上下边界，即在泛型声明的时候添加
//public <T> T showKeyName(Generic<T extends Number> container)，编译器会报错："Unexpected bound"
public <T extends Number> T showKeyName(Generic<T> container){
    System.out.println("container key :" + container.getKey());
    T test = container.getKey();
    return test;
}
```


## 7、泛型的原理

##### 7.1 类型擦除

<font color=#ff0000>Java中的泛型是通过类型擦除来实现的</font>。**所谓类型擦除，是指通过类型参数合并，将泛型类型实例关联到同一份字节码上。编译器只为泛型类型生成一份字节码，并将其实例关联到这份字节码上。类型擦除的关键在于从泛型类型中清除类型参数的相关信息，并且再必要的时候添加类型检查和类型转换的方法。**

下面通过两个例子来证明在编译时确实发生了类型擦除。

例1分别创建实际类型为`String`和`Integer`的`ArrayList`对象，通过`getClass()`方法获取两个实例的类，最后判断这个实例的类是相等的，证明两个实例共享同一个类。

![image](http://bloghello.oursnail.cn/javabasic11-1.png)

例2创建一个只能存储`Integer`的`ArrayList`对象，在`add`一个整型数值后，利用反射调用`add(Object o)` `add`一个`asd`字符串，此时运行代码不会报错，运行结果会打印出1和asd两个值。这时再里利用反射调用`add(Integer o)`方法，运行会抛出`codeNoSuchMethodException`异常。这充分证明了在编译后，擦除了`Integer`这个泛型信息，只保留了原始类型。

![image](http://bloghello.oursnail.cn/javabasic11-2.png)

##### 7.2 自动类型转换

上一节上说到了类型擦除，Java编译器会擦除掉泛型信息。那么调用`ArrayList`的`get()`最终返回的必然会是一个`Object`对象，但是我们在源代码并没有写过`Object`转成`Integer`的代码，为什么就能“直接”将取出来的对象赋予一个`Integer`类型的变量呢（如下面的代码第12行）？

![image](http://bloghello.oursnail.cn/javabasic11-3.png)

<font color=#ff0000>实际上，Java的泛型除了类型擦除之外，还会自动生成`checkcast`指令进行强制类型转换</font>。上面的代码中的main方法编译后所对应的字节码如下。


![image](http://bloghello.oursnail.cn/javabasic11-4.png)

看到第26行代码就是将`Object`类型的对象强制转换为`Integer`的指令。我们完全可以将上面的代码转换为下面的代码，它所实现的效果跟上面的泛型是一模一样的。既然泛型也需要进行强制转换，所以泛型并不会提供运行时效率，不过可以大大降低编程时的出错概率。


![image](http://bloghello.oursnail.cn/javabasic11-5.png)


## 8、简单总结

### 8.1 类型擦除(Type Erasure)
- Java 的泛型是在编译器层次实现的。
- 在编译生成的字节码中不包含泛型中的类型参数，类型参数会在编译时去掉。
- 例如：`List<String>` 和 `List<Integer>` 在编译后都变成 `List`。
- 类型擦除的基本过程：将代码中的类型参数替换为具体的类，同时去掉 `<>` 的内容。


### 8.2 泛型的优势

- 编译时更强大的类型检测。

例如如下代码：方法传入一个`String`对象，传出一个`String` 对象，并强制转换为`Integer`对象。这段代码编译可以通过，因为都是`Object`的子类，但是运行时会产生`ClassCastException`。


![image](http://bloghello.oursnail.cn/javabasic11-6.png)

而如果通过泛型来实现，则会在编译时进行类型的检测。例如如下代码：会产生编译错误。

![image](http://bloghello.oursnail.cn/javabasic11-7.png)

- 提供自动和隐式的类型转换

![image](http://bloghello.oursnail.cn/javabasic11-8.png)

### 8.3 `<T>` VS `<?>`

不同点：

- `<T>`用于泛型的定义，例如`class MyGeneric<T> {...}`
- `<?>`用于泛型的声明，即泛型的使用，例如`MyGeneric<?> g = new MyGeneric<>()`;

相同点：都可以指定上界和下界:

![image](http://bloghello.oursnail.cn/javabasic11-9.png)

### 8.4 `<?>`不同于`<Object>`

- 指定未知类型，如`List<?>`。`List<?>`不等于`List<Object>`

![image](http://bloghello.oursnail.cn/javabasic11-10.png)

`String`是`Object`的子类，但是`List<String>`不是`List<Object>`的子类。

![image](http://bloghello.oursnail.cn/javabasic11-11.png)

如果将`List<Object>`换成`List<?>`，则可以编译通过。

注意：

- 相同参数类型的泛型类的继承关系取决于泛型类自身的继承结构。
例如`List<String>`是`Collection<String>`的子类
- 当类型声明中使用通配符`?`时，其子类型可以在两个维度上扩展。

```java
例如 Collection<? extends Number>
在维度1上扩展：List<? extends Number>
在维度2上扩展：Collection<Integer>
```

## 9、Java泛型中`List`、`List<Object>`、`List<?>`的区别

- `List`：原生态类型
- `List<Object>`：参数化的类型，表明`List`中可以**容纳任意类型的对象**
- `List<?>`：无限定通配符类型，表示**只能包含某一种未知对象类型**

![image](http://bloghello.oursnail.cn/javabasic11-12.png)

我们创建了一个`List<String>`类型的对象`strings`，再把它赋给原生态类型`List`，这是可以的。但是第5行中尝试把它传递给`List<Object>`时，出现了一个类型不相容错误，注意，这是一个编译期错误。

这是因为泛型有子类型化的规则：

`List<String>`是原生态类型`List`的一个子类型。虽然`String`是`Object`的子类型，但是由于泛型是不可协变的，`List<String>`并不是`List<Object>`的子类型，所以这里的传递无法通过编译。

`List<Object>`唯一特殊的地方只是`Object`是所有类型的超类，由于泛型的不可协变性，它只能表示`List`中可以容纳所有类型的对象，却不能表示任何参数类型的`List<E>`。

![image](http://bloghello.oursnail.cn/javabasic11-13.png)


输出结果：

```
11
sss
```

总结：

- `List<Object>`:表示可用装载任意类型的对象，如上面最后一个例子，但是他不能接受`List<String>`的替换，因为不具有继承性，并且`List<Object>`如果可以被`List<String>`，就不符合原则了，因为`List<String>`只能接受String类型的对象。
- `List<?>`:解决上面表面有继承关系的List的赋值问题，还有就是，他是用作声明能接收一种未知对象类型，而不是大杂烩啥都能接收。
- `List`：原始类型，啥都没有限制。个人认为与`List<Object>`类似，但是又没有继承的限制。即啥类型都可以接收。


## 10、参考
- http://hinylover.space/2016/06/25/relearn-java-generic-1/
- [java 泛型详解](https://blog.csdn.net/s10461/article/details/53941091)
- https://www.cnblogs.com/rese-t/p/8158870.html
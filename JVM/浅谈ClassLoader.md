title: 浅谈ClassLoader
tag: JVM
---

本篇为学习JAVA虚拟机的第二篇文章，上一篇文章初步提到了class文件，以及一个最简单程序执行的指令含义，我们提到，是由JAVA虚拟机先加载这些编译好的class文件，然后再去根据解析出来的指令去转换为具体平台上的机器指令执行，但是加载这个class文件时如何加载的呢？其实就涉及比较重要的东西：ClassLoader
<!-- more -->

有一个基本认识，从编译到实例化对象的过程可以概括为以下三个阶段：

- 编译器将`xxx.java`源文件编译为`xxx.class`字节码文件
- `ClassLoader`将字节码转换为JVM种的`Class<xxx>`对象
- JVM利用`Class<xxx>`对象实例化为`xxx`对象

## 一、JVM系统结构

![image](http://bloghello.oursnail.cn/jvm2-1.png)

- `ClassLoader`：依据特定格式，加载class文件到内存
- `Execution Engine`：对命令进行解析
- `Native Interface`：融合不同开发语言的原生库为Java所用
- `Runtime Data Area`：JVM内存空间结构模型

首先通过`ClassLoader`加载符合条件的字节码文件到内存中，然后通过`Execution Engine`解析字节码指令，交由操作系统去执行。


## 二、什么是ClassLoader

`ClassLoader`在java中有着非常重要的作用，它主要工作在`Class`装载的加载阶段，其主要作用是从系统外部获得`Class`二进制数据流。他是JAVA的核心组件，所有的`Class`都是由`ClassLoader`进行加载的，`ClassLoader`负责通过将`Class`文件里的二进制数据流装载进系统，然后交给JAVA虚拟机进行连接、初始化等操作。

简而言之，就是加载字节码文件。

我们翻开`ClassLoader`源码看看：
```java
public abstract class ClassLoader {...}
```
它是一个抽象类，下面我们再来说具体的实现类。

里面比较重要的是`loadClass()`方法：


```java
public Class<?> loadClass(String name) throws ClassNotFoundException {
    return loadClass(name, false);
}
```

就是根据`name`来加载字节码文件，返回`Class`实例，加载不到则抛出`ClassNotFoundException`异常。

## 三、ClassLoader的种类

- 启动类加载器（`Bootstrap ClassLoader`）：由`C++`语言实现（针对`HotSpot`）,加载核心库`java.*`。


- 扩展类加载器（`Extension ClassLoader`）：Java编写，加载扩展库`javax.*`

它扫描的是哪个路径呢？

![image](http://bloghello.oursnail.cn/jvm2-2.png)

我们看到，它负责将 `<JAVA_HOME >/lib/ext`或者由系统变量`-Djava.ext.dir`指定位置中的类库 加载到内存中。

- 应用程序类加载器（`Application ClassLoader`）：Java编写，加载程序所在目录

![image](http://bloghello.oursnail.cn/jvm2-3.png)

它负责将 用户类路径(`java -classpath`或`-Djava.class.path`变量所指的目录，即当前类所在路径及其引用的第三方类库的路径，看截图的最后一行，显示的是当前项目路径。

- 自定义`ClassLoader`：自定义


## 四、如何自定义ClassLoader

要自己实现一个`ClassLoader`，其核心涉及两个方法：


```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}

protected final Class<?> defineClass(byte[] b, int off, int len)
    throws ClassFormatError
{
    return defineClass(null, b, off, len, null);
}
```


首先想一下为什么是这两个类？

其实答案在`loadClass()`这个方法里面。如果已经熟悉双亲委派模型的同学，都会知道加载`Class`对象是先委派给父亲，看父亲是否已经加载，如果没有加载过，则从最顶层父亲开始逐层往下进行加载，这一块详细在下一篇文章中解释，我们先走马观花看看这个的核心方法长啥样：


```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        //首先看看当前类加载器是否已经加载过，没有则委派给父亲查询
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    //注意，这里是各递归函数，如果由下至上查询都没有加载过，则从上至下尝试去加载
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
            //如果所有类加载器都没有加载过，则开始尝试从上而下逐级去加载
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                //去加载
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

如果我们不去重写`findClass(name)`方法，默认是直接抛出找不到的异常，所以我们要对这个方法进行重写。

由于字节码文件是一堆二进制流，所以需要一个方法来根据这个二进制流来定义成一个类，即`defineClass()`这个方法来实现这个功能。

说的比较抽象，下面来真正实践一把！


## 五、实践自定义ClassLoader


首先写一个类：Robot.java


```java
public class Robot {
    static {
        System.out.println("hello , i am a robot!");
    }
}
```

在对`Robot.java`用`javac`编译之后形成`Robot.class`文件，就要删除本项目下的这个`Robot.java`文件，要不然就会被`AppClassLoader`类加载先加载了，而无法再被我们的自定义类加载器再去加载。这个`Robot.class`文件我就直接放到桌面去了。路径为`C:/Users/swg/Desktop/`.

然后定义一个自定义的`ClassLoader`，按照上面的理论，只要重写`findClass`就可以指定到某个地方获取class字节码文件，此时获取的是二进制流文件，转换为字节数组，最后借用`defineClass`获取真正的`Class`对象。


```java
public class MyClassLoader extends ClassLoader{
    //执行加载的class文件的路径
    private String path;
    //自定义类加载器的名字
    private String classLoaderName;

    MyClassLoader(String path,String classLoaderName){
        this.path = path;
        this.classLoaderName = classLoaderName;
    }

    //用于寻找类文件
    @Override
    protected Class findClass(String name){
        byte[] b = loadClassData(name);
        return defineClass(name,b,0,b.length);
    }

    //用于加载类文件
    private byte[] loadClassData(String name) {
        name = path + name + ".class";
        InputStream in = null;
        ByteArrayOutputStream out = null;
        try{
            in  = new FileInputStream(new File(name));
            out = new ByteArrayOutputStream();
            int i=0;
            while ((i = in.read()) != -1){
                out.write(i);
            }
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            try {
                in.close();
                out.close();
            }catch (Exception e){
                e.printStackTrace();
            }
        }
        return out.toByteArray();
    }

}
```

最后测试一下能不能用自定义类加载器去加载到`Robot`对应的Class对象：
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

打印结果：


```
MyClassLoader@677327b6
hello , i am a robot!
```

好了，学习了关于`ClassLoader`的分类以及如何自定义`ClassLoader`，我们知道了类加载器的基本实现，上面谈到了一个重要方法是`loadClass`，这就涉及了类加载器的双亲委派模型。下一节从代码层面好好来说说这个，其实很简单。
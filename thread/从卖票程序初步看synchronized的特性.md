title: 从卖票程序初步看synchronized的特性
tag: java多线程
---

本文是关于JAVA多线程和并发的第五篇，在多线程学习和编程中，synchronized都是我们第一个要碰见的关键字，它很重要，因为它被认为还有优化的空间，并且它代表的是互斥锁的基本思想，JDK或者其他地方的源码随处可见，本文用一个卖票程序来切入synchronized的学习，从语法和使用上进行全面了解，并且对其引申出来的一些概念进行说明。
<!--more-->

## 1. 线程安全问题产生原因

- 存在共享数据
- 存在多条线程共同操作这些共享数据


## 2. 线程安全问题解决方法

上面的问题归根结底是由于两个线程访问相同的资源造成的。对于并发编程，需要采取措施防止两个线程来访问相同的资源。

一种措施是当资源被一个线程访问时，为其加锁。第一个访问资源的线程必须锁定该资源，是其他任务在资源被解锁前不能访问该资源。

基本上所有的并发模式在解决线程安全问题时，都采用“序列化访问临界资源”的方案。即在同一时刻，只能有一个线程访问临界资源，也称作同步互斥访问。通常来说，是在访问临界资源的代码前面加上一个锁，当访问完临界资源后释放锁，让其他线程继续访问。 

这里来好好谈谈`Synchronized`实现加锁的方式。


## 3. synchronized修饰符

`synchronized`：可以在任意对象及方法上加锁，而加锁的这段代码称为“互斥区”或“临界区”.

`synchronized`满足了以下重要特性：

- 互斥性：即在同一时间只允许一个线程持有某个对象锁，通过这种特性来实现多线程的协调机制，这样在同一时间只有一个线程对需要同步的代码块进行访问，互斥性也称为操作的原子性。
- 可见性：必须确保在锁被释放之前，对共享变量所做的修改，对于随后获得该锁的另一个线程是可见的，否则另一个线程可能是在本地缓存的某个副本上继续操作，从而引起不一致

⭐`synchronized`锁的不是代码，是对象！


##### 3.1 不使用synchronized会出现线程不安全问题


```java
public class SellTicket implements Runnable{
	private int count = 5;

	@Override
	public void run() {
		sellTicket();
	}
	
	private void sellTicket() {
        if(count>0){
            count--;
            System.out.println(Thread.currentThread().getName()+",还剩"+count);
        }else {
            System.out.println("票卖光了");
        }
    }
	
}
```


```java
public class Main {
	public static void main(String[] args) {
		SellTicket sellTicket = new SellTicket();
		//同时开启五个线程去卖票
		Thread t1 = new Thread(sellTicket, "thread1");
		Thread t2 = new Thread(sellTicket, "thread2");
		Thread t3 = new Thread(sellTicket, "thread3");
		Thread t4 = new Thread(sellTicket, "thread4");
		Thread t5 = new Thread(sellTicket, "thread5");
		
		t1.start();
		t2.start();
		t3.start();
		t4.start();
		t5.start();
	}
}
```

某一次运行的结果是:

```
thread2,还剩2
thread1,还剩2
thread3,还剩2
thread4,还剩0
thread5,还剩0
```

很显然，多个线程之间打架了，数据混乱了。这是因为，多个线程同时操作`run（）`方法，对`count`进行修改，进而造成错误。

##### 3.2 使用synchronized来加锁

对卖票的核心方法上加上`synchronized`：

```java
private synchronized void sellTicket() {
    if(count>0){
        count--;
        System.out.println(Thread.currentThread().getName()+",还剩"+count);
    }else {
        System.out.println("票卖光了");
    }
}
```

或者写成同步代码块的形式：

```java
private void sellTicket() {
	synchronized (this) {
		if(count>0){
            count--;
            System.out.println(Thread.currentThread().getName()+",还剩"+count);
        }else {
            System.out.println("票卖光了");
        }
	}
}
```


结果只有一个：

```
thread1 count:4
thread4 count:3
thread5 count:2
thread3 count:1
thread2 count:0
```
结果是正确的，可以看出代码A和代码B的区别就是在`sellTicket()`方法上加上了`synchronized`修饰。

**说明**：当多个线程访问`MyThread` 的`run`方法的时候，如果使用了`synchronized`修饰，那个多线程就会以排队的方式进行处理（这里排队是按照CPU分配的先后顺序而定的），一个线程想要执行`synchronized`修饰的方法里的代码，首先是尝试获得锁，如果拿到锁，执行`synchronized`代码体的内容，如果拿不到锁的话，这个线程就会不断的尝试获得这把锁，直到拿到为止，而且多个线程同时去竞争这把锁，也就是会出现锁竞争的问题。


##### 3.3 一个对象有一把锁！不同对象不同锁！

每次开启一个线程就new一个对象的话，即对每个不同的对象加锁，则互不干扰：

```java
public class Main {
	public static void main(String[] args) {
		Thread t1 = new Thread(new SellTicket(), "thread1");
		Thread t2 = new Thread(new SellTicket(), "thread2");
		Thread t3 = new Thread(new SellTicket(), "thread3");
		Thread t4 = new Thread(new SellTicket(), "thread4");
		Thread t5 = new Thread(new SellTicket(), "thread5");
		
		t1.start();
		t2.start();
		t3.start();
		t4.start();
		t5.start();
	}
}
```

线程任务`SellTicket()`无论给不给`sellTicket()`加锁，结果都是一样的：

```
thread1,还剩4
thread2,还剩4
thread3,还剩4
thread5,还剩4
thread4,还剩4
```
这是因为我这里是五个不同的对象，每个对象各自获取自己的锁，互不影响，所以都是4.

关键字`synchronized`取得的锁都是对象锁，而不是把一段代码或方法当做锁，所以上述实例代码C中哪个线程先执行`synchronized` 关键字的方法，那个线程就持有该方法所属对象的锁，五个对象，线程获得的就是两个不同对象的不同的锁，他们互不影响的。


那么，我们在正常的场景的时候，肯定是有一种情况的就是，一个类`new`出来的所有对象会对一个变量`count`进行操作，那么如何实现哪？很简单就是加`static`，我们知道，用`static`修改的方法或者变量，在该类的所有对象是具有相同的引用的，这样的话，无论实例化多少对象，调用的都是一个方法。

`Main`函数不变：

```java
public class Main {
	public static void main(String[] args) {
		Thread t1 = new Thread(new SellTicket(), "thread1");
		Thread t2 = new Thread(new SellTicket(), "thread2");
		Thread t3 = new Thread(new SellTicket(), "thread3");
		Thread t4 = new Thread(new SellTicket(), "thread4");
		Thread t5 = new Thread(new SellTicket(), "thread5");
		
		t1.start();
		t2.start();
		t3.start();
		t4.start();
		t5.start();
	}
}
```
`SellTicket`则在卖票方法上增加`static`关键字：

```java
public class SellTicket implements Runnable{
	private static int count = 5;

	@Override
	public void run() {
		sellTicket();
	}
	
	private synchronized static void sellTicket() {
        if(count>0){
            count--;
            System.out.println(Thread.currentThread().getName()+",还剩"+count);
        }else {
            System.out.println("票卖光了");
        }
    }
}
```

或者显示地锁住`Class`对象，即锁住类对象：

```java
private static void sellTicket() {
	synchronized (SellTicket.class) {
		if(count>0){
            count--;
            System.out.println(Thread.currentThread().getName()+",还剩"+count);
        }else {
            System.out.println("票卖光了");
        }
	}
}
```
结果为:


```
thread1,还剩4
thread2,还剩3
thread4,还剩2
thread3,还剩1
thread5,还剩0
```


仔细看，我们给`sellTicket`设定为`static`静态方法，那么这个方法就从之前的对象方法上升到类级别方法，这个类所有的对象都调用的同一个方法。实现资源的共享和加锁。

上面讲的时对象锁和类锁，前者锁定的是某个实例对象，后者锁定的是Class对象。下面总结一下：

- 有线程访问对象的同步代码块时，另外的线程可以访问该对象的非同步代码块
- 若锁住的时同一个对象，一个线程在访问对象的同步代码块(同步方法)时，另一个访问对象的同步代码块(同步方法)的线程会被阻塞
- 同一个类的不同对象的对象锁互不干扰
- 类锁由于也是一种特殊的对象锁，因此表现与上述一致，只是由于一个类只有一把类锁，所以同一个类的不同对象使用类锁是同步的
- 类锁和对象锁互不干扰

## 4. Synchronized锁重入

##### 4.1 什么是可重入锁

锁的概念就不用多解释了,当某个线程A已经持有了一个锁,当线程B尝试进入被这个锁保护的代码段的时候.就会被阻塞.

**⭐而锁的操作粒度是"线程”,而不是调用.同一个线程再次进入同步代码的时候.可以使用自己已经获取到的锁,这就是可重入锁。**

##### 4.2 可重入锁的小例子

```java
public class SyncDubbo {
	public synchronized void method1(){
		System.out.println("method1...");
		method2();
	}
	
	public synchronized void method2(){
		System.out.println("method2...");
		method3();
	}
	
	public synchronized void method3(){
		System.out.println("method3...");
	}
	
	public static void main(String[] args) {
		SyncDubbo syncDubbo = new SyncDubbo();
		new Thread(new Runnable() {
			
			@Override
			public void run() {
				syncDubbo.method1();
			}
		}).start();
	}
}
```

示例代码向我们演示了，如何在一个已经被`synchronized`关键字修饰过的方法再去调用对象中其他被`synchronized`修饰的方法。

##### 4.3 为什么要可重入

我们上一篇文章中介绍了“一个对象一把锁，多个对象多把锁”，可重入锁的概念就是：**自己可以获取自己的内部锁**。

**假如有1个线程T获得了对象A的锁，那么该线程T如果在未释放前再次请求该对象的锁时**，如果没有可重入锁的机制，是不会获取到锁的，这样的话就会出现死锁的情况。

就如代码A体现的那样，线程T在执行到`method1（）`内部的时候，由于该线程已经获取了该对象`syncDubbo` 的对象锁，当执行到调用`method2（）` 的时候，会再次请求该对象的对象锁，如果没有可重入锁机制的话，由于该线程T还未释放在刚进入`method1（）` 时获取的对象锁，当执行到调用`method2（）` 的时候，就会出现死锁。

##### 4.4 可重入锁到底有什么用哪？

正如上述代码A和（4.3）中解释那样，最大的作用是避免死锁。假如有一个场景：用户名和密码保存在本地txt文件中，则登录验证方法和更新密码方法都应该被加`synchronized`，那么当更新密码的时候需要验证密码的合法性，所以需要调用验证方法，此时是可以调用的。

##### 4.5  什么是死锁？

 线程A当前持有互斥所锁`lock1`，线程B当前持有互斥锁`lock2`。接下来，当线程A仍然持有`lock1`时，它试图获取`lock2`，因为线程B正持有`lock2`，因此线程A会阻塞等待线程B对`lock2`的释放。如果此时线程B在持有`lock2`的时候，也在试图获取`lock1`，因为线程A正持有`lock1`，因此线程B会阻塞等待A对`lock1`的释放。二者都在等待对方所持有锁的释放，而二者却又都没释放自己所持有的锁，这时二者便会一直阻塞下去。这种情形称为死锁。
 
 一个例子来说明：
 
 
```java
public class DeadLock {
	private static Object obj1 = new Object();
	private static Object obj2 = new Object();
	
	public static void a(){
		synchronized (obj1) {
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			synchronized (obj2) {
				System.out.println("a");
			}
		}
	}
	
	public static void b(){
		synchronized (obj2) {
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			synchronized (obj1) {
				System.out.println("b");
			}
		}
	}
	
	public static void main(String[] args) {
		DeadLock d = new DeadLock();
		
		new Thread(new Runnable() {
			
			@Override
			public void run() {
				d.a();
			}
		}).start();
		
		new Thread(new Runnable() {
			
			@Override
			public void run() {
				d.b();
			}
		}).start();
	}
}
```
**产生死锁的原因主要是：**

* （1） 因为系统资源不足。
* （2） 进程运行推进的顺序不合适。
* （3） 资源分配不当等。

**如何解决死锁：**

- 尽量一个线程只获取一个锁。
- 一个线程只占用一个资源。
- 尝试使用定时锁，至少能保证锁最终会被释放。

##### 4.6 可重入锁支持在父子类继承的环境中


```java
public class Father  
{  
    public synchronized void doSomething(){  
        ......  
    }  
}  
  
public class Child extends Father  
{  
    public synchronized void doSomething(){  
        ......  
        super.doSomething();  
    }  
}  
```
执行子类的方法的时候,先获取了一次`Widget`的锁,然后在执行`super`的时候,就要获取一次,如果不可重入,那么就跪了.

在这里，可能会产生疑问：

> 重入”代表一个线程可以再次获得同一个对象的锁。可是你给出的代码示例中，我理解的是一个线程调用`Child`的`doSomething`方法前或得了`Child`对象的锁，`super.doSomething`方法调用时，次线程获得了`Child`对象父对象的锁。两个锁属于不同的对象，这还算是重入吗？


解释：当`Child`实例对象调用`doSomething`方法时，此时持有的是`Child`实例对象的锁，之后调用`super.doSomething();`，这时仍然对于`Child`实例对象加锁，因为此时仍然使用的是`Child`实例对象内存空间的数据。

至于这句话的理解，就牵涉到继承的机制：

> 在一个子类被创建的时候，首先会在内存中创建一个父类对象，然后在父类对象外部放上子类独有的属性，两者合起来形成一个子类的对象。所以所谓的继承使子类拥有父类所有的属性和方法其实可以这样理解，子类对象确实拥有父类对象中所有的属性和方法，但是父类对象中的私有属性和方法，子类是无法访问到的，只是拥有，但不能使用。就像有些东西你可能拥有，但是你并不能使用。所以子类对象是绝对大于父类对象的，所谓的子类对象只能继承父类非私有的属性及方法的说法是错误的。可以继承，只是无法访问到而已。

之所以网上有很多说只继承`protected`或者`private`的，是因为从语言的角度出发的：

![image](https://uploadfiles.nowcoder.net/images/20171019/1829253_1508380548300_1372D420D0F13C1126C6FCB3DC35A515)

从内存的角度来看，的确是继承了的，可以写一个简单的继承类，debug看子类的属性是否存在父类的private属性，事实证明是有的。

针对这里有人说：不是创建一个父类对象，而只是创建一个父类空间并进行相应的初始化。对此，我一开始也是这么想的，不过当我看到这个答案的时候，又觉得很有道理：


> 会创建父类对象。《Java编程思想》（第四版）129页，当创建一个导出类对象时，该对象包含了一个基类的子对象，这子对象与你用基类直接创建的对象是一样的，二者区别在于后者来源于外部，而基类的子对象被包装在导出类对象内部。

## 5. 发生异常时会自动释放锁


```java
public class SyncException {

    private int i = 0;

    public synchronized void operation() {
        while (true) {
            i++;
            System.out.println(Thread.currentThread().getName() + " , i= " + i);
            if (i == 10) {
                Integer.parseInt("a");
            }
        }
    }

    public static void main(String[] args) {
        final SyncException se = new SyncException();
        new Thread(new Runnable() {
            public void run() {
                se.operation();
            }
        }, "t1").start();
    }
}
```
执行结果如下：


```
t1 , i= 2
t1 , i= 3
t1 , i= 4
t1 , i= 5
t1 , i= 6
t1 , i= 7
t1 , i= 8
t1 , i= 9
t1 , i= 10
java.lang.NumberFormatException: For input string: "a"
    at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
    //其他输出信息
```

关于`synchronized`的优化放到后文去讲。
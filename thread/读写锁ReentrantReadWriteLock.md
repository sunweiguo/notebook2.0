title: 读写锁ReentrantReadWriteLock
tag: java多线程
---

读写锁的出现是为了提高性能，思想是：读读不互斥，读写互斥，写写互斥。本文来了解一下读写锁的使用和锁降级的概念。

<!-- more -->

## 1. 锁的分类

* 排他锁：在同一时刻只允许一个线程进行访问，其他线程等待；
* 读写锁：在同一时刻允许多个读线程访问，但是当写线程访问，所有的写线程和读线程均被阻塞。读写锁维护了一个读锁加一个写锁，通过读写锁分离的模式来保证线程安全，性能高于一般的排他锁。

## 2. 读写锁

我们对数据的操作无非两种：“读”和“写”，试想一个这样的情景，当十个线程同时读取某个数据时，这个操作应不应该加同步。答案是没必要的。只有以下两种情况需要加同步：

* 这十个线程对这个公共数据既有读又有写
* 这十个线程对公共数据进行写操作
* 以上两点归结起来就一点就是有对数据进行改变的操作就需要同步

所以

**java5提供了读写锁这种锁支持多线程读操作不互斥，多线程读写互斥，多线程写互斥**。读操作不互斥这样有助于性能的提高，这点在java5以前没有。

## 3. java并发包提供的读写锁

java并发包提供了读写锁的具体实现`ReentrantReadWriteLock`，它主要提供了一下特性：

* 公平性选择：支持公平和非公平（默认）两种获取锁的方式，非公平锁的吞吐量优于公平锁；
* 可重入：支持可重入，读线程在获取读锁之后能够再次获取读锁，写线程在获取了写锁之后能够再次获取写锁，同时也可以获取读锁；
* 锁降级：线程获取锁的顺序遵循获取写锁，获取读锁，释放写锁，写锁可以降级成为读锁。

## 4. 先看个小例子

**读取数据和写入数据**
```java
import java.util.HashMap;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Demo {
    //定义一个map用来读取和存放数据
	private HashMap<String,String> map = new HashMap<String,String>();
	
	//实例化ReentrantReadWriteLock
	private ReadWriteLock rwl = new ReentrantReadWriteLock();
	
	//根据实例化对象分别获取读锁和写锁
	private Lock r = rwl.readLock();
	private Lock w = rwl.writeLock();
	
	//读取数据
	public void get(String key){
	    //上读锁
		r.lock();
		System.out.println(Thread.currentThread().getName()+" 读操作开始执行");
		try{
			try {
				Thread.sleep(3000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			//读取数据
			System.out.println(map.get(key));
		}finally {
		    //解读锁
			r.unlock();
			System.out.println(Thread.currentThread().getName()+" 读操作执行完毕");
		}
	}
	
	//存入数据，即写数据
	public void put(String key,String value){
	    //上写锁
		w.lock();
		System.out.println(Thread.currentThread().getName()+" 写操作开始执行");
		try{
			try {
				Thread.sleep(3000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			//写数据
			map.put(key, value);
		}finally{
		    //解写锁
			w.unlock();
			System.out.println(Thread.currentThread().getName()+" 写操作执行完毕");
		}
	}
	
}
```

**Main进行创建多线程测试：先来测试一下存在写的情况(只有写或者写读都有)**
```java
public class Main {
	public static void main(String[] args) {
		Demo demo = new Demo();
		
		//写
		new Thread(new Runnable() {
			@Override
			public void run() {
				demo.put("key1", "value1");
			}
		}).start();
		
		//读
		new Thread(new Runnable() {
			@Override
			public void run() {
				demo.get("key1");
			}
		}).start();
		
		//写
		new Thread(new Runnable() {
			@Override
			public void run() {
				demo.put("key2", "value2");
			}
		}).start();
		
		//写
		new Thread(new Runnable() {
			@Override
			public void run() {
				demo.put("key3", "value3");
			}
		}).start();
		
	}
}
```

**执行结果：**
```
Thread-0 写操作开始执行
Thread-0 写操作执行完毕
Thread-1 读操作开始执行
value1
Thread-1 读操作执行完毕
Thread-2 写操作开始执行
Thread-2 写操作执行完毕
Thread-3 写操作开始执行
Thread-3 写操作执行完毕
```

**分析：**

发现存在写的情况，那么就是一个同步等待的过程，即开始执行，然后等待3秒，执行完毕，符合第2个目录中提到的规则。

**对只有读操作的情形进行测试**


```java
public class Main {
	public static void main(String[] args) {
		Demo demo = new Demo();
		
		demo.put("key1", "value1");
		demo.put("key2", "value2");
		demo.put("key3", "value3");
		
		new Thread(new Runnable() {
			@Override
			public void run() {
				demo.get("key1");
			}
		}).start();
		
		new Thread(new Runnable() {
			@Override
			public void run() {
				demo.get("key2");
			}
		}).start();
		
		new Thread(new Runnable() {
			@Override
			public void run() {
				demo.get("key3");
			}
		}).start();
	}
}
```


**运行结果：**
```
Thread-0 读操作开始执行
Thread-1 读操作开始执行
Thread-2 读操作开始执行
value1
Thread-0 读操作执行完毕
value2
Thread-1 读操作执行完毕
value3
Thread-2 读操作执行完毕
```

**分析**

在主线程中先`put`进去几个数用于读的测试，下面开辟三个读线程，我们可以从执行结果中发现，其中一个线程进去之后，另外的线程能够立即再次进入，即这三把锁不是互斥的。


## 5. 锁降级

锁降级是指写锁将为读锁。

锁降级：从写锁变成读锁；锁升级：从读锁变成写锁。读锁是可以被多线程共享的，写锁是单线程独占的。也就是说写锁的并发限制比读锁高，这可能就是升级/降级名称的来源。

如下代码会产生死锁，因为同一个线程中，在没有释放读锁的情况下，就去申请写锁，这属于锁升级，`ReentrantReadWriteLock`是不支持的。


```java
ReadWriteLock rtLock = new ReentrantReadWriteLock();  
rtLock.readLock().lock();  //上读锁
System.out.println("get readLock.");  
rtLock.writeLock().lock();  //读锁还没有释放，不允许上死锁
System.out.println("blocking"); 
```
`ReentrantReadWriteLock`支持锁降级，如下代码不会产生死锁。


```java
ReadWriteLock rtLock = new ReentrantReadWriteLock();  
rtLock.writeLock().lock();  //上写锁
System.out.println("writeLock");  
  
rtLock.readLock().lock();  //可以在写锁没有释放的时候立即上读锁
System.out.println("get read lock"); 
```

利用这个机制：**同一个线程中，在没有释放读锁的情况下，就去申请写锁，这属于锁升级，`ReentrantReadWriteLock`是不支持的。**

在写锁没有释放的时候，先获取到读锁，然后再释放写锁，保证后面读到的数据的一致性。

![image](https://segmentfault.com/img/bVOGUM?w=1063&h=246)

```java
private volatile boolean isUpdate;

public void readWrite(){
	r.lock();//为了保证isUpdate能够拿到最新的值
	if(isUpdate){
		r.unlock();
		w.lock();
		map.put("xxx","xxx");
		r.lock();//写锁还没有释放，立即获取读锁，阻塞本线程，保证本线程下面读的一致性
		w.unlock();
	}
	String value = map.get("xxx"); //读到的数据是本线程自己更新的数据，不会被其他线程打扰
	System.out.println(value);
	r.unlock();
}
```

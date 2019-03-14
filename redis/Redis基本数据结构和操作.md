title: Redis基本数据结构和操作
tag: redis
---

本文为redis学习笔记的第二篇文章，本文主要介绍redis如何启动，以及基本的键命令和五种基本数据类型的操作。部分图片可能看不清楚，可以拖到新窗口打开。

<!-- more -->
## 一、启动方式

我的环境是`windows`，那么直接进入`redis`的解压目录中，分别执行`redis-server.exe`和`redis-cli.exe`两个可执行的程序。也可以通过`cmd`启动：

![image](http://bloghello.oursnail.cn/redis2-1.png)

不要直接用`crtl+C`关闭`server`，在`linux`下，直接停掉`server`的话，会导致数据的丢失。正确的做法是在客户端执行 `redis-cli.exe shutdown`

![image](http://bloghello.oursnail.cn/redis2-2.png)


还可以指定端口启动：`./redis-server.exe --port 6380`


![image](http://bloghello.oursnail.cn/redis2-3.png)

那么对应客户端连接也要指定相应 的端口才能连接。关闭服务端也要指定相应的端口才行：

![image](http://bloghello.oursnail.cn/redis2-4.png)


`-h`指定远程`redis`的`ip`


![image](http://bloghello.oursnail.cn/redis2-5.png)

通过配置文件启动,可以在下面这个文件中指定端口号：

![image](http://bloghello.oursnail.cn/redis2-6.png)

结合配置文件启动:

![image](http://bloghello.oursnail.cn/redis2-7.png)

还可以设置密码：

![image](http://bloghello.oursnail.cn/redis2-8.png)

那么客户端连接就必须要密码验证了：

![image](http://bloghello.oursnail.cn/redis2-9.png)

## 二、命令

###### 1、基础命令

`info`:查看系统信息

`select (0-15)`，redis一共有16个工作区间，一般默认从0开始，到15.

![image](http://bloghello.oursnail.cn/redis2-10.png)

- `flushdb`：清空当前选择的空间
- `flushall`：清空所有
- `dbsize`：当前空间里面key-value键值对的数目
- `save`：人工实现redis的持久化
- `quit`：退出

###### 2、键命令

`del key`成功返回1，失败返回0.

![image](http://bloghello.oursnail.cn/redis2-11.png)


`exits key`

![image](http://bloghello.oursnail.cn/redis2-12.png)

`ttl`和`expire`

![image](http://bloghello.oursnail.cn/redis2-13.png)

`type key` 查看key的类型

`randomkey`:

![image](http://bloghello.oursnail.cn/redis2-14.png)

`rename oldkey newkey`

![image](http://bloghello.oursnail.cn/redis2-15.png)


如果是重命名为已经存在的key呢？

![image](http://bloghello.oursnail.cn/redis2-16.png)

`renamenx`:

![image](http://bloghello.oursnail.cn/redis2-17.png)

## 三、redis数据结构
###### 1、String字符串

`setex`&`psetex`

![image](http://bloghello.oursnail.cn/redis2-18.png)

`getrange`&`getset`

![image](http://bloghello.oursnail.cn/redis2-19.png)

`mset`&`mget`&`strlen`


![image](http://bloghello.oursnail.cn/redis2-20.png)

`setnx`&`msetnx`

![image](http://bloghello.oursnail.cn/redis2-21.png)


数值操作

![image](http://bloghello.oursnail.cn/redis2-22.png)

##### 2、hash

![image](http://bloghello.oursnail.cn/redis2-23.png)

##### 3、list

![image](http://bloghello.oursnail.cn/redis2-24.png)

##### 4、set

![image](http://bloghello.oursnail.cn/redis2-25.png)

![image](http://bloghello.oursnail.cn/redis2-26.png)

![image](http://bloghello.oursnail.cn/redis2-27.png)

###### 5、sorted set

![image](http://bloghello.oursnail.cn/redis2-28.png)
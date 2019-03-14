title: SpringBoot通用知识深入--切面、异常、单元测试
tag: miscellany
---

对于小白来说，下面的知识都是满满的干货，值得好好学习，具体的视频是学习的廖师兄的[Spring Boot进阶之Web进阶](http://www.imooc.com/learn/810)，值得一看，下面是笔记总结。
<!-- more -->

## 一、面向切面编程

![image](http://bloghello.oursnail.cn/zaji16-1.png)

访问 `localhost:8080/dev/add` 日志打印结果是：

![image](http://bloghello.oursnail.cn/zaji16-2.png)

## 二、校验和返回结果封装

1、首先是在`student`类中对`username`增加注解：`@NotNull(message = "用户名不能为空")`

2、在增加一个学生的方法上进行参数的验证：

![image](http://bloghello.oursnail.cn/zaji16-3.png)

3、在http工具上输入

> http://localhost:8080/dev/add?username=hello&age=10

返回正确信息，即学生的json流：{"id":5,"username":"hello","age":10}

如果不传`username`，即必填的那一项不给，则发生异常：

> http://localhost:8080/dev/add?age=10

返回信息为：用户名不能为空

4、我们会发现，正确返回是新添加学生的json格式，错误返回就是一个字符串，这样对于前台来说是无法处理的，所以需要一个封装类来包装一下。

![image](http://bloghello.oursnail.cn/zaji16-4.png)

5、下面对controller层进行改造，返回统一的格式。

![image](http://bloghello.oursnail.cn/zaji16-5.png)

6、controller曾进行结果封装的时候，发现代码重复，进行优化。

新建一个工具类，用来封装结果。

![image](http://bloghello.oursnail.cn/zaji16-6.png)

继而改造controller:

![image](http://bloghello.oursnail.cn/zaji16-7.png)

结果与上面一致。

## 三、统一异常处理

如果service层业务逻辑是：

![image](http://bloghello.oursnail.cn/zaji16-8.png)

1、我们第一个想到的方案可能是给每一种情况加上一个标记，controller层根据标记的不同进行不同的返回处理：

service层方法：

![image](http://bloghello.oursnail.cn/zaji16-9.png)

相应的controller层为：

![image](http://bloghello.oursnail.cn/zaji16-10.png)

对于逻辑比较简单的情况下，是可以到达我们的预期效果，但是一旦业务量逻辑复杂度高一点，就会非常地混乱。解决方案是统一异常处理。

2、加上异常处理

service层处理为:

![image](http://bloghello.oursnail.cn/zaji16-11.png)

controller层处理为:

![image](http://bloghello.oursnail.cn/zaji16-12.png)

返回结果为：

![image](http://bloghello.oursnail.cn/zaji16-13.png)

显然格式都是不统一的，解决方案是对默认的exception返回信息再进行一次封装。

3、修改exception返回信息格式：

新建一个handle类：

![image](http://bloghello.oursnail.cn/zaji16-14.png)

但是对于这种方式，如果我想让上小学的学硕状态码为100,上初中的学生的状态码为101，就无法实现了。解决方案：自定义异常。

4、自定义异常

新建一个异常类：

![image](http://bloghello.oursnail.cn/zaji16-15.png)

在`service`层抛出异常为`throw new StudentException(100,"还在上小学")`。

在刚才写的默认异常处理类中进行判断处理：

![image](http://bloghello.oursnail.cn/zaji16-16.png)

这样，当属于小学生时，状态码为100,当属于初中生时，状态码为101，就区分开了。当不属于这个异常的异常，就会抛出未知错误。这样也不好，最好用日志将未知错误打印出来。


5、系统异常(默认异常处理):

![image](http://bloghello.oursnail.cn/zaji16-17.png)

到现在仍然存在一些问题：状态码与状态信息都是自己在程序中临时定义的，是不规范的行为，需要有一个地方统一管理，便于区分和防止混乱重复。

6、枚举定义异常状态

![image](http://bloghello.oursnail.cn/zaji16-18.png)

改造service层：

![image](http://bloghello.oursnail.cn/zaji16-19.png)

![image](http://bloghello.oursnail.cn/zaji16-20.png)

这样就完成了异常的统一管理。


## 四、springboot测试

1、对于service层的测试
创建：方法名右击---go to--test

![image](http://bloghello.oursnail.cn/zaji16-21.png)

2、对于controller层的api的测试

![image](http://bloghello.oursnail.cn/zaji16-22.png)
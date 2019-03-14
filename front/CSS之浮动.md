title: CSS之浮动
tag: 前端基础
---
本文来研究一下第二个重点：浮动。
<!--more-->

## 浮动

![image](http://bloghello.oursnail.cn/html7-1.png)

那么我们如何让他变成一行呢？之前已经说过了，可以用`inline-block`来实现：

![image](http://bloghello.oursnail.cn/html7-2.png)

但是我们发现，有该死的间隙存在。此时就需要用到我的浮动了。什么是浮动，顾名思义，就是浮在上面，下面来直观感受一下：

![image](http://bloghello.oursnail.cn/html7-3.png)

我们会发现，一个加了浮动，一个没有加的话，那么这两个在排版的时候是互相不冲突的，也就是说，浮动的元素根本就不占用标准元素的空间，所以这两者重叠在了一起。浮动就像是飞机，而不是浮动的就是汽车，汽车与汽车之间是互斥的，但是汽车与飞机之间不互斥，下面我们再来解决上面存在间隙的问题，我们只需要将这两个元素全部置为向左浮动即可。

![image](http://bloghello.oursnail.cn/html7-4.png)

那么既然已经是浮动的元素了，这个元素是属于谁的呢？也就是说，是属于整个body还是属于父元素的呢？如果是属于父元素的话，那么就非常好了，我可以调整好父元素的位置之后，里面的子元素采用浮动来置为一行。我们来测试一下。

![image](http://bloghello.oursnail.cn/html7-5.png)

这说明确实是属于父元素的。

## 一些问题

![image](http://bloghello.oursnail.cn/html7-6.png)

此时下面的`div`直接就将第一个`outer`给顶掉了，这不是我们想要的结果。结果方案有三种。

![image](http://bloghello.oursnail.cn/html7-7.png)

可能有的时候需要用第三种，因为第一种方法有的时候父元素会隐藏掉多出来的子元素。有的时候我们不想隐藏，就需要用到这个方法。

![image](http://bloghello.oursnail.cn/html7-8.png)


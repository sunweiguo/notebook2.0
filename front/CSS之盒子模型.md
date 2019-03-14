title: CSS之盒子模型
tag: 前端基础
---
盒子模型是比较基础但是比较重要的一点，本文来简单了解一下。
<!--more-->

## 盒子模型

![image](http://bloghello.oursnail.cn/html6-1.png)

CSS盒模型本质上是一个盒子，封装周围的HTML元素，它包括：边距，边框，填充，和实际内容。
盒模型允许我们在其它元素和周围元素边框之间的空间放置元素。
下面的图片说明了盒子模型(Box Model)：

![image](http://bloghello.oursnail.cn/html6-2.png)

由上面我知道了：最终元素的总宽度计算公式是这样的：

- 总元素的宽度=宽度+左填充+右填充+左边框+右边框+左边距+右边距

元素的总高度最终计算公式是这样的：

- 总元素的高度=高度+顶部填充+底部填充+上边框+下边框+上边距+下边距

简单了解了盒子模型之后，我们看到这个margin是在盒子的最外面，那么两个这样的盒子相邻的话可能会出一些问题。

比如两个盒子一上一下放，两个都设置margin，那么这两者之间上下间隔会是多少呢？是两者的margin之和吗？

⭐经过验证，间隔是以大的那个为准，而不是两者之和。

那对于内部嵌套的情况呢？

![image](http://bloghello.oursnail.cn/html6-5.png)

看起来好像没什么问题，都是符合我们一开始的期望的。但是去掉父盒子中的`padding`或者同时删除掉`padding`和`border`会是什么样子呢？

![image](http://bloghello.oursnail.cn/html6-6.png)

好像子中`margin`的设置没什么作用了，此时我增大一下`margin`：

![image](http://bloghello.oursnail.cn/html6-4.png)

（删除掉`border`效果也一样），效果就是两者同时下移了。这就很奇怪了，用比较专业的词汇来说，就是这两者的`margin`咋合并了呢？这里确实是合并了，`div`距离最上面的距离由其中较大的`margin`的值决定。

解决方法是什么呢？正如上面第一种显示的，要么加一个`border`要么加一个`padding`.

![image](http://bloghello.oursnail.cn/html6-7.png)


还有一种方式是在父元素上增加`overflow`，顾名思义就是溢出的意思。

![image](http://bloghello.oursnail.cn/html6-8.png)

这也可以解决边框合并问题。我们稍微来看一下这个溢出是啥意思。起始就是说，如果子元素不在这个父元素范围内了，就直接不显示了。我们也可以显示，调整为`auto`模式即可，就可以下拉看到子元素了。

![image](http://bloghello.oursnail.cn/html6-9.png)

我们来稍微总结一下，盒子模型的计算方法跟`margin`决定的外边距，`border`决定的边框宽度，`padding`决定的内边距以及实际内容相关。

然后就是存在外边距合并的问题，一个是上下关系，一个是嵌套关系。重点是嵌套关系，要解除外边距合并问题，第一个比较可行的方案是给父元素添加一个白色的边框，一个是给父元素增加`overflow`。那个`padding`是不大好用的，毕竟增加了`padding`之后整个父元素的宽度或者高度都变化了。

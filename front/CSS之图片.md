title: CSS之图片
tag: 前端基础
---
本文来说一说图片相关的一些处理以及精灵图。
<!--more-->

## 图片

![image](http://bloghello.oursnail.cn/html5-1.png)

这就是简单放一张图片，但是我们发现，当图片比我们设定的div要小的时候，它会自动复制取铺满整个div。如果我们仅仅显示原来的图片，不要它铺开呢？

![image](http://bloghello.oursnail.cn/html5-2.png)

这样就不会重复显示了，那我们能不能移动移动这个图片呢？

![image](http://bloghello.oursnail.cn/html5-3.png)

注意，正数是让它往左往右移动。这个有什么用呢？这就要说一说精灵图了。

还可以用`background-repeat: repeat-x;`表示横向自动填充。


## 精灵图

什么是精灵图呢？

我们去腾讯游戏的官网看一下：

![image](http://bloghello.oursnail.cn/html5-4.png)

我们注意到有很多的小图标，这些小图片特点是数量多，并且很小。为了减轻与服务器交互带来的不必要的开销，可以将这些小图标全部放在一张大图上，要用哪个图标通过上面说过的`background-position`来指定即可。

![image](http://bloghello.oursnail.cn/html5-5.png)

比如我想只显示那个奖杯。

![image](http://bloghello.oursnail.cn/html5-6.png)

慢慢移动到这个位置就可以了。
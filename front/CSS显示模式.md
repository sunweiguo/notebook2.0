title: CSS显示模式
tag: 前端基础
---

CSS显示模式一般分为两种，一种是独占一行的`block`一种是行内元素`inline`类型。他们之间可以互相转换。是比较常见的，比较重要。

<!--more-->

## 显示模式

我们知道`div`是独占一行或者说一块`block`，默认情况下同行是不能再放其他元素了。`span`是行内元素，即`inline`，一行可以有很多列这样的元素。

他们还有一个区别是，块级元素可以设置行高，但是行内元素是不能设置行高的，比如`span`给他一个高度也是没有用的，它是随着里面内容的变化而变化的，比如文字，里面的文字变大，那么这个`span`区域也就会随着变大。

`block`和`inline`就是两种显示模式。


## block转inline

![image](http://bloghello.oursnail.cn/html3-1.png)


我们可以验证上面的说法，就是`div`是个块级，一行默认独占一格，所以两个`inner`就分为了两行，那么有没有办法调整为`inline`元素呢？


![image](http://bloghello.oursnail.cn/html3-2.png)

这又验证了一下上面的说法，就是`inline`元素是否显示以及显示的大小与里面的内容有关。如果我这里不写一点字占坑的话，就直接没了。


此时我们发现虽然可以将这两个`div`放到同一行去，但是称为`inline`元素之后我们就不能随意设置它的宽高了，我又想把他们搞到一行，又想设置宽高，怎么实现呢？答案就是用`inline-block`

![image](http://bloghello.oursnail.cn/html3-3.png)

## inline转block

比如比较常见的`a`标签，往往不是点文字才有用，而是点一大块区域都可以，但是我们知道`a`标签是一格`inline`标签，不能直接给他设置宽高，这个时候需要将它转为`block`就好了，怎么搞呢？

![image](http://bloghello.oursnail.cn/html3-4.png)

通过`display: block;`之后就变成了`inline-block`，就可以设置宽高了，那一片区域都可以作为超链接点击。


下面再来一个例子，比如`ul`和`li`标签，默认情况下是每一个独占一行的，如果我想让它跟分页一样排成一行怎么弄呢？起始很简单就是将其置为`inline-block`即可。

![image](http://bloghello.oursnail.cn/html3-5.png)

下一节用之前的知识做一个小例子。
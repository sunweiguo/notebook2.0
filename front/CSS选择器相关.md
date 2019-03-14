title: CSS选择器相关
tag: 前端基础
---

CSS要想根据我们的需要对指定的东西进行美化，需要用到选择器。下面我们来看看基本的选择器是如何使用的。

<!--more-->

## 一、内联样式


```html
<!DOCTYPE html>
<html>
<head>
	<title></title>
</head>
<body>

	<div style="color: skyblue;border: 1px dashed red;">我是南邮吴镇宇！</div>

</body>
</html>
```


一般情况下不会这么写，所以会涉及选择器，就是css到底对谁起作用。


```
<link rel="stylesheet" type="text/css" href="">
```

在外部放一个css文件。

## 二、选择器

##### 2.1 ID选择器

就是给某个标签，比如`div`标签增加一个`id="div1"`，那么我就可以通过

```css
#div1{
    border: 1px dashed red;
}
```

完整如下：


```html
<!DOCTYPE html>
<html>
<head>
	<title></title>
	<style type="text/css">
		#div1{
		    border: 1px dashed red;
		    color: skyblue;
		}
	</style>
</head>
<body>

	<div id="div1">我是南邮吴镇宇！</div>

</body>
</html>
```
但是精准控制每个id是不现实的，要累死人的，下面介绍标签选择器。


##### 2.2 标签选择器



```css
<style type="text/css">
	div{
		border: 1px dashed red;
	    color: skyblue;
	}
</style>
```

标签为`div`的都起作用了。这种方式也不好，因为范围太大了。


##### 2.3 类选择器

形如：

```html
<!DOCTYPE html>
<html>
<head>
	<title></title>
	<style type="text/css">
		.div1{
			border: 1px dashed red;
		    color: skyblue;
		}
	</style>
</head>
<body>

	<div class="div1">我是南邮吴镇宇！</div>

	<div>我是南邮吴彦祖！</div>

</body>
</html>
```

这里的`class`可以写多个。这边可能会出现覆盖。但是注意类选择器的权重是小于ID选择器的，所以类选择器无法覆盖ID选择器的效果。

##### 2.4 后代选择器

比如我这里：


```html
<!DOCTYPE html>
<html>
<head>
	<title></title>
	<style type="text/css">
		.div1{
			border: 1px dashed red;
		    color: skyblue;
		}
	</style>
</head>
<body>

	<div class="div1">
		我是南邮吴镇宇！
	</div>

	<div class="div1">
		我是南邮吴彦祖！
	</div>

</body>
</html>
```

那么这两个`div`都会起作用，但是如果我只想让吴镇宇变化咋办呢？我们可以给他加个`span`标签。`span`是`.div`的儿子。也可以给这个`span`里面加一个`class`，写法一样的。这就是后代选择器。


```html
<!DOCTYPE html>
<html>
<head>
	<title></title>
	<style type="text/css">
		.div1 span{
			border: 1px dashed red;
		    color: skyblue;
		}
	</style>
</head>
<body>

	<div class="div1">
		<span>
			我是南邮吴镇宇！
		</span>
	</div>

	<div class="div1">
		我是南邮吴彦祖！
	</div>

</body>
</html>
```

比较简单，这些都是比较常用的选择器，当然还有一些比较特殊的选择器，到时候再说。

## 三、字体相关

```html
<!DOCTYPE html>
<html>
<head>
	<title></title>
	<style type="text/css">
		.div1{
			font-size: 16px;/*12px在谷歌中是最小的字体*/
			font-family: "宋体";/*字体样式，一般用默认即可*/
			font-style: italic;/*默认是normal,italic是斜体*/
			font-weight: 900;/*100-900的范围，默认是normal*/
			text-align: left;/*center置为中间*/
			text-indent: 2em;/*首行缩进*/
			line-height: 50px;/*调整一行的行高*/
		}

		a{
			text-decoration: none;/*去掉a标签的下划线*/
			color: green;/*设定默认链接是绿色*/
		}
		/*鼠标悬浮在a标签上之后就会变成红色并且出现下划线*/
		a:hover{
			color: red;
			text-decoration: underline;
		}
	</style>
</head>
<body>

	<div class="div1">
		我是南邮吴镇宇！
	</div>

	<a href="#">
		我是南邮吴彦祖！
	</div>

</body>
</html>
```

## 四、交集和并集


```html
<body>

	<p>我是1号</p>
	<p id="id">我是2号</p>
	<p class="para">我是3号</p>
	<p class="para">我是4号</p>
	<p>我是5号</p>

</body>
```

如果想选标签是`p`并且`class="para"`的行，这就是交集：


```css
<style type="text/css">
	p.para{
		color: red;
	}
</style>
```

如果我选择`class="para"`或者`id="id"`的行，这就是并集：


```css
<style type="text/css">
	#id,.para{
		color: red;
	}
</style>
```



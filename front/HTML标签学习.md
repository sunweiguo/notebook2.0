title: HTML标签学习
tag: 前端基础
---

学习前端的第一步是学习html的基本标签，比较重要的标签并不多，多用用就好了。下面一一过一下，复习一下即可。

<!--more-->

## 一、文本标签

- p标签

`<p>xxxx</p>`对文本可以进行段落划分。

- 斜体

`<i>xxx</i>`或者`<em>xxx</em>`

- 删除线

`<s>xxx</s>`或者`<del>xxx</del>`

- 加粗

`<strong>xxx</strong>`或者`<b>xxx</b>`

- 下划线

`<u>xxx</u>`或者`<ins>xxx</ins>`

## 二、图片

没什么好说的：`<img src="xxx" alt="图裂了" title="本图片提示信息">`


## 三、超链接

- 原地跳转新页面：`<a href="http://www.google.com" target="_self">超链接</a>`
- 新开页面跳转：`<a href="http://www.google.com" target="_blank">超链接</a>`


这里顺便说一下锚点定位。就是点一下页面移动到指定的地方开始显示。

`<a href="#hello">移动到hello处</a>`

`<h1 id="hello">我是hello</h1>`

点一下就移动到对应的地方了。


## 四、列表

- 无序列表：


```html
<ul>
    <li>1</li>
    <li>2</li>
    <li>3</li>
</ul>
```


- 有序列表：

```html
<ol>
    <li>1</li>
    <li>2</li>
    <li>3</li>
</ol>
```

所谓的顺序其实只是后者显示了123而已，并不是说对内容进行排序。

- 自定义列表

```html
<dl>
    <dt>标题1</dt>
    <dd>1</dd>
    <dd>2</dd>
    <dd>3</dd>
    <dt>标题2</dt>
    <dd>4</dd>
    <dd>5</dd>
    <dd>6</dd>
</dl>
```

⭐在`webstorm`中，我们这样一个一个打有点累，其实是有快捷键的。比如我写一个有5个`<li>`的`<ul>`，快捷写法是`ul>li*5`再按`tab`即可。

比如我要五组这样的呢？`(ul>li*5)*5`+`tab`即可。

## 五、表格

这个也没什么好说的

```html
<table border="1px">
    <tr>
        <td>111</td>
        <td>222</td>
        <td>333</td>
    </tr>
    <tr>
        <td>444</td>
        <td>555</td>
        <td>666</td>
    </tr>
</table>
```

也可以有表头用`<th>`即可。下面注意一下，如果我们想要`111`一下占两列咋办呢？就是跟`excel`中差不多的单元格合并功能。

- 占两列：`<td colspan="2">111</td>`
- 占两行：`<td rowspan="2">111</td>`


## 六、表单

- 输入框：`<input type="text">`

往往下面这个是成对出现的：效果是点击`label`的提示语鼠标的焦点就会进入输入框了。

```html
<label for="text">请输入：</label>
<input id="text" type="text">
```


- 按钮：`<input type="button" value="我是按钮">`


- 勾选框：`<input type="checkbox">`


- 单选按钮：`<input type="radio">`


比如男女是一组`radio`，如何只让用户选择一个呢？


```html
<input name="sex" type="radio" value="male">
<input name="sex" type="radio" value="female">
```

- 文本输入框：`<textarea name="" id="" cols="30" rows="10"></textarea>`

- 下拉框：


```html
<select name="" id="">
    <option value="">1</option>
    <option value="">2</option>
    <option value="">3</option>
</select>
```

## 七、name属性

表单提交，比如登陆功能，我们需要将输入的用户名和密码发送到服务器校验。那么后端如何接收这个参数呢？其实就是根据`name`来识别的，后端受到的数据形如`username:xxx`，那么后端就可以根据`username`这个`name`属性接受到xxx这个内容。

id只是标识这个标签，要唯一。两者不要混淆。


```html
<form action="服务器接口地址">
    <input type="text" name="username">
    <input type="password" name="password">
    <input type="submit">
    <input type="reset">
</form>
```

好了，关键的标签就全部说完了。其实猛然回首才发现，这不正是我们初学html的时候一个一个学习的标签吗?真是时光荏苒呐！下面就要进军CSS了。
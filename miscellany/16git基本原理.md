title: Git基本原理
tag: miscellany
---
小伙伴们对git并不陌生，全球最大的同性交友网站github是程序猿们的最爱，这里只有你想不到的，没有你找不到的，各种资源应有尽有，免去了从浏览器中找到若干垃圾的麻烦。因此熟悉git也成为程序猿标配，本文来简单说说git的基本操作和基本原理。
<!-- more -->

## 一、Git工作流程

![image](http://bloghello.oursnail.cn/zaji17-1.png)

这四个区域的名字如下:

- Workspace：工作区

> 程序员进行开发改动的地方，是你当前看到的，也是最新的。平常我们开发就是拷贝远程仓库中的一个分支，基于该分支进行开发。在开发过程中就是对工作区的操作。

- Index / Stage：暂存区

> `.git`目录下的`index`文件, 暂存区会记录`git add`添加文件的相关信息(文件名、大小、`timestamp`...)，不保存文件实体, 通过id指向每个文件实体。
>
> 可以使用`git status`查看暂存区的状态。暂存区标记了你当前工作区中，哪些内容是被git管理的。
当你完成某个需求或功能后需要提交到远程仓库，那么第一步就是通过`git add`先提交到暂存区，被git管理。

- Repository：仓库区（或本地仓库）

> `git commit`后同步`index`的目录树到本地仓库，方便从下一步通过`git push`同步本地仓库与远程仓库的同步。

- Remote：远程仓库

> 远程仓库的内容可能被分布在多个地点的处于协作关系的本地仓库修改，因此它可能与本地仓库同步，也可能不同步，但是它的内容是最旧的。


**总结一下：**

- 任何对象都是在工作区中诞生和被修改；
- 任何修改都是从进入`index`区才开始被版本控制；
- 只有把修改提交到本地仓库，该修改才能在仓库中留下痕迹；
- 与协作者分享本地的修改，可以把它们`push`到远程仓库来共享。

下面这幅图更加直接阐述了四个区域之间的关系，可能有些命令不太清楚，没关系，下部分会详细介绍。

![image](http://bloghello.oursnail.cn/zaji17-2.png)


## 二、常用git命令

![image](http://bloghello.oursnail.cn/zaji17-3.png)

看不清可以拖动图片到新的页面打开。我们从关键字入手git常用命令。

## 2.1 HEAD

`HEAD`，它始终指向当前所处分支的最新的提交(commit)点。你所处的分支变化了，或者产生了新的提交点，`HEAD`就会跟着改变。

![image](http://bloghello.oursnail.cn/zaji17-4.png)

## 2.2 add和commit

![image](http://bloghello.oursnail.cn/zaji17-5.png)

`add`相关命令很简单，主要实现将工作区修改的内容提交到暂存区，交由git管理。

命令 | 含义
---|---
git add . | 添加当前目录的所有文件到暂存区
git add [dir] | 添加指定目录到暂存区，包括子目录
git add [file] | 添加指定文件到暂存区


`commit`相关命令也很简单，主要实现将暂存区的内容提交到本地仓库，并使得当前分支的`HEAD`向后移动一个提交点。

命令 | 含义
---|---
git commit -m [message] | 提交暂存区到本地仓库，message代表说明信息
git commit [file] -m [message] | 提交暂存区的指定文件到本地仓库
git commit --amend -m [message] | 使用一次新的commit，替代上一次提交

## 2.3 branch


![image](http://bloghello.oursnail.cn/zaji17-8.png)

涉及到协作，自然会涉及到分支，关于分支，大概有展示分支，切换分支，创建分支，删除分支这四种操作。

命令 | 含义
---|---
git branch | 列出所有本地分支
git branch -r | 列出所有远程分支
git branch -a | 列出所有本地分支和远程分支
git branch [branch-name] | 新建一个分支，但依然停留在当前分支
git branch -b [branch-name] | 新建一个分支，并切换到该分支
git branch --track [branch] remote-branch[] | 新建一个分支，与指定的远程分支建立追踪关系
git checkout [branch-name] | 切换到指定分支，并更新工作区
git branch -d [branch-name] | 删除分支
git push origin --delete [branch-name] | 删除远程分支

## 2.4 merge

![image](http://bloghello.oursnail.cn/zaji17-10.png)

`merge`命令把不同的分支合并起来。如上图，在实际开放中，我们可能从`master`分支中切出一个分支，然后进行开发完成需求，中间经过R3,R4,R5的`commit`记录，最后开发完成需要合入`master`中，这便用到了`merge`。

命令 | 含义
---|---
git fetch [remote] | merge之前先拉一下远程仓库最新代码
git merge [branch] | 合并指定分支到当前分支

一般在`merge`之后，会出现`conflict`，需要针对冲突情况，手动解除冲突。主要是因为两个用户修改了同一文件的同一块区域。

就是说同一个代码两个人都进行了修改，那么必然需要通过人工的沟通协调最终选择一个统一的版本。

## 2.5 reset

![image](http://bloghello.oursnail.cn/zaji17-12.png)

reset命令把当前分支指向另一个位置，并且相应的变动工作区和暂存区。

命令 | 含义
---|---
git reset --soft [commit] | 只改变提交点，暂存区和工作区的内容都不改变
git reset --mixed [commit] | 改变提交点，同时改变暂存区的内容
git reset --hard [commit] | 暂存区和工作区的内容都会被修改到与提交点完全一致的状态
git reset --hard HEAD | 让工作区回到上次提交的状态

还有一个叫做`git revert`,与`git reset`的区别是：

![image](http://bloghello.oursnail.cn/zaji17-14.png)

`git reset` 是把`HEAD`向后移动了一下，而`git revert`是`HEAD`继续前进，只是新的`commit`的内容和要`revert`的内容正好相反，能够抵消要被`revert`的内容。

##### 2.6 push

上传本地仓库分支到远程仓库分支，实现同步。

命令 | 含义
---|---
git push [remote] [branch] | 上传本地指定分支到远程仓库
git push [remote] --force | 强行推送当前分支到远程仓库，即使有冲突
git push [remote] --all | 推送所有分支到远程仓库

## 2.7 其他命令

命令 | 含义
---|---
git status | 显示有变更的文件
git log | 显示当前分支的版本历史
git diff | 显示暂存区和工作区的差异
git diff HEAD | 显示工作区与当前分支最新commit之间的差异
git cherry-pick [commit] | 选择一个commit合并进当前分支


整理自：[一篇文章让你读懂Git](https://mp.weixin.qq.com/s?__biz=MzUwOTQ1NTAzNA==&mid=2247483714&idx=2&sn=a7893d7306025dc35ca0fb2678003795&chksm=f910be97ce673781f259bb353b3802818eb64e2cb9b06390290374c3d95d139f32d8acb710e8&mpshare=1&scene=1&srcid=1220thpAb37YI9AJFiDH0rMA#rd)
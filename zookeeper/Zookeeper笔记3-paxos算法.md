title: Zookeeper笔记3-paxos算法
tag: zookpeeper
---

有人说一千个人就有一千个paxos算法理解，算法本身晦涩难懂，如何快速生动理解paxos核心要点一直是一个老大难的问题，本文集结多篇文章精华，算是理顺了其中的门道。
<!-- more -->

## 一、paxos解决了什么问题

在上一章节中，我们着重提到了解决一致性问题的两种协议：2PC和3PC。但是在分布式环境中，这两种协议都无法真正实现一致性(分布式的一致性问题其实主要是指分布式系统中的数据一致性问题。所以，为了保证分布式系统的一致性，就要保证分布式系统中的数据是一致的。)

分布式系统中的节点通信存在两种模型：共享内存（Shared memory）和消息传递（Messages passing）。基于消息传递通信模型的分布式系统，不可避免的会发生以下错误：
- 进程可能会慢、被杀死或者重启
- 消息可能会延迟、丢失、重复

（在基础 Paxos 场景中，先不考虑可能出现消息篡改即拜占庭错误的情况）

**⭐Paxos 算法解决的问题是在一个可能发生上述异常的分布式系统中如何就某个值达成一致，保证不论发生以上任何异常，都不会破坏决议的一致性。**

那么我们可用形象地理解为：Paxos可以说是一个民主选举的算法——大多数节点的决定会成个整个集群的统一决定。任何一个点都可以提出要修改某个数据的提案，是否通过这个提案取决于这个集群中是否有超过半数的节点同意。取值一旦确定将不再更改，并且可以被获取到(不可变性，可读取性)。

## 二、典型场景

一个典型的场景是，在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点都执行相同的操作序列，那么他们最后能得到一个一致的状态。为保证每个节点执行相同的命令序列，需要在每一条指令上执行一个“一致性算法”以保证每个节点看到的指令一致。

所以，**paxos算法主要解决的问题就是如何保证分布式系统中各个节点都能执行一个相同的操作序列**。


![image](http://bloghello.oursnail.cn/18-12-2/77123681.jpg)


上图中，C1是一个客户端，N1、N2、N3是分布式部署的三个服务器，初始状态下N1、N2、N3三个服务器中某个数据的状态都是S0。当客户端要向服务器请求处理操作序列：op1--op2--op3时（op表示operation）

如果想保证在处理完客户端的请求之后，N1、N2、N3三个服务器中的数据状态都能从S0变成S1并且一致的话（或者没有执行成功，还是S0状态），就要保证N1、N2、N3在接收并处理操作序列op1--op2--op3时，严格按照规定的顺序正确执行opi，要么全部执行成功，要不就全部都不执行。


所以，针对上面的场景，paxos解决的问题就是如何依次确定不可变操作opi的取值，也就是确定第i个操作什么，在确定了opi的内容之后，就可以让各个副本执行opi操作。



## 三、paxos算法

### 1、算法涉及的主要角色：

- Proposer：提议者，提出议案(同时存在一个或者多个，他们各自发出提案)
- Acceptor：接受者，收到议案后选择是否接受
- Learner：最终决策学习者，只学习正确的决议
- Client：产生议题者，发起新的请求

![image](http://bloghello.oursnail.cn/18-12-3/34413307.jpg)

主要的角色就是“提议者”和“接受者”。先有提议，再来表决。（注：实际应用中，可以将一堆服务器任意指定角色，一部分做“提议者”、一部分做“接受者”，也可以指定特定的服务器做“提议者”，剩下的都是“接受者”。）

`Proposer`就像`Client`的代理人，由`Proposer`拿着`Client`的议题去向`Acceptor`提议，让`Acceptor`来做出决策。

这幅图表示的是角色之间的逻辑关系，每一种角色就代表了一种节点类型。在物理部署环节，可以把每一种角色都部署在一台物理机器上，也可以组合任何两种或者多种角色部署在一台物理机器上，甚至于，把这四种角色都部署在同一台物理机器上也是可以的。



### 2、关于这两阶段的理解

从一个故事说起。

从前，在国王Leslie Lamport的统治下，有个黑暗的希腊城邦叫paxos。城邦里有3类人，
- 决策者
- 提议者
- 群众


虽然这是一个黑暗的城邦但是很民主，按照议会民主制的政治模式制订法律，群众有什么建议和意见都可以写提案交给提议者，提议者会把提案交给决策者来决策，决策者有奇数个，为什么要奇数个？很简单因为决策的方式很无脑，少数服从多数。最后决策者把刚出炉的决策昭告天下，群众得知决策结果。

等一下，那哪里黑暗呢？问题就出在“提议者会把提案交给决策者来决策”，那么多提案决策者先决策谁的？谁给的钱多就决策谁的。

那这样会有几个问题，决策者那么多，怎么保证最后决策的是同一个提案，以及怎么保证拿到所有提议者中最高的报价。

聪明又贪婪的决策者想到了一个办法：分两阶段报价


**第一阶段**：

- 决策者接受所有比他当前持有报价高的报价，且不会通知之前报价的人
- 提议者给所有(一半以上即可)决策者报价，若有人比自己报价高就加价，有半数以上决策者接受自己报价就停止报价。

一旦某个提议者收到了所有决策者中一半以上的人同意的回复。就会进入第二阶段。

**第二阶段**：


提议者去找收过自己钱的大佬签合同，这里有3种情况：

- 很多大佬收了别人更高的价，达不到一半人数了，只好回去拿钱继续贿赂，回到第一阶段重新升级;
- 大佬收到的最高报价是自己的，美滋滋，半数以上成功签合同，提案成功;
- 提议者回去拿钱回来继续贿赂的时候发现合同已经被签了且半数以上都签了这个提案，不干了，赶快把自己的提案换成已经签了的提案，再去提给所有大佬，看看能不能分一杯羹遇见还没签的大佬。

最后一步就是让所有节点知道这个过半通过的提议是什么，从而达到最终的一致。

## 三、深化理解

假设有两个“提议者”和三个“接受者”。下面这一坨的内容一开始如果看不明白不要紧，立即转到下面的图示过程，看懂图示再回过头来就会理解了。

- 怎么明确意见领袖呢？通过编号。每个“提议者”在第一阶段先报个号，谁的号大，谁就是意见领袖。如果不好理解，可以想象为贿选。每个提议者先拿着钞票贿赂一圈“接受者”，谁给的钱多，第二阶段“接受者”就听谁的。**意见领袖理解为贿赂中胜出的“提议者”即可**。
- 有个跟选举常识不一样的地方，**就是每个“提议者”不会执着于让自己的提议通过，而是每个“提议者”会执着于让提议尽快达成一致意见**。所以，为了这个目标，如果“提议者”在贿选的时候，发现“接受者”已经接受过前面意见领袖的提议了，即便“提议者”贿选成功，也会默默的把自己的提议改为前面意见领袖的提议。所以一旦贿赂成功，胜出的“提议者”再提出提议，提议内容也是前面意见领袖的提议（这样，在谋求尽早形成多数派的路上，又前进了一步）。
- 钱的多少很重要，如果钱少了，无论在第一还是第二阶段“接受者”都不会鸟你，直接拒绝。
- 上面讲到，如果“提议者”在贿选时，发现前面已经有意见领袖的提议，那就将自己的提议默默改成前面意见领袖的提议。这里有一种情况，如果你是“提议者”，在贿赂的时候，“接受者1”跟你说“他见过的意见领袖的提议是方案1”，而“接受者2”跟你说“他见过的意见领袖提议是方案2”，你该怎么办？这时的原则也很简单，还是：钱的多少很重要！你判断一下是“接受者1”见过的意见领袖有钱，还是“接受者2”见过的意见领袖有钱？如何判断呢？因为“接受者”在被“提议者”贿赂的时候，自己会记下贿赂的金额。所以当你贿赂“接受者”时，一旦你给的贿赂多而胜出，“接受者”会告诉你两件事情：a.前任意见领袖的提议内容（如果有的话），b.前任意见领袖当时贿赂了多少钱。这样，再面对刚才的情景时，你只需要判断一下“接受者1”和“接受者2”告诉你的信息中，哪个意见领袖当时给的钱多，那你就默默的把自己的提议，改成那个意见领袖的提议。
- 在整个选举过程中，每个人谁先来谁后到，“接受者”什么时间能够接到“提议者”的信息，是完全不可控的。所以很可能一个意见领袖已经产生了，但是由于这个意见领袖的第二阶段刚刚开始，绝大部分“接受者”还没有收到这个意见领袖的提议。结果，这时突然冲进来了一个新的土豪“提议者”，那么这个土豪“提议者”也是有机会让自己的提议胜出的！这时就形成了一种博弈：a.上一个意见领袖要赶在土豪“提议者”贿赂到“接受者”前，赶到“接受者”面前让他接受自己的提议，否则会因为自己的之前贿赂的钱比土豪少而被拒绝。b.土豪“提议者”要赶在上一个意见领袖将提议传达给“接受者”前，贿赂到“接受者”，否则土豪“提议者”即便贿赂成功，也要默默的将自己的提议改为前任意见领袖的提议。这整个博弈的过程，最终就看这两个“提议者”谁的进展快了。但最终一定会有一个意见领袖，先得到多数“接受者”的认可，那他的提议就胜出了。这一块不理解就看下面的分解说明。

1）首先“提议者1”贿赂了3个“接受者”

![image](http://bloghello.oursnail.cn/zk3-1.png)


2）3个“接受者”记录下贿赂金额，因为目前只有一个“提议者”出价，因此$1就是最高的了，所以“接受者”们返回贿赂成功。此外，因为没有任何先前的意见领袖提出的提议，因此“接受者”们告诉“提议者1”没有之前接受过的提议（自然也就没有上一个意见领袖的贿赂金额了）。

假如此时由于贿赂的人数超过了一半，那么第一阶段成功，准备进入第二阶段，就是正式签合同，签超过半数的合同才真正表示本轮成功。

![image](http://bloghello.oursnail.cn/zk3-2.png)

3）“提议者1”发现有超过一半人接受了自己的贿赂，下面就要真正发起提议了，先向“接受者1”提出了自己的提议：1号提议，并告知自己之前已贿赂$1。

![image](http://bloghello.oursnail.cn/zk3-3.png)

4）“接受者1”检查了一下，目前记录的贿赂金额就是$1，于是接受了这一提议，并把1号提议记录在案。

![image](http://bloghello.oursnail.cn/zk3-4.png)

5）在“提议者1”向“接受者2”“接受者3”发起提议前，土豪“提议者2”出现，他开始用$2贿赂“接受者1”与“接受者2”。

![image](http://bloghello.oursnail.cn/zk3-5.png)


6）“接受者1”与“接受者2”立刻被收买，将贿赂金额改为$2。但是，不同的是：“接受者1”告诉“提议者2”,之前我已经接受过1号提议了，同时1号提议的“提议者”贿赂过$1；而，“接受者2”告诉“提议者2”，之前没有接受过其他意见领袖的提议，也没有上一个意见领袖的贿赂金额。

![image](http://bloghello.oursnail.cn/zk3-6.png)

7）这时，“提议者1”回过神来了，他向“接受者2”和“接受者3”发起1号提议，并带着信息“我前期已经贿赂过$1”。

![image](http://bloghello.oursnail.cn/zk3-7.png)

8）“接受者2”“接受者3”开始答复：“接受者2”检查了一下自己记录的贿赂金额，然后表示，已经有人出价到$2了，而你之前只出到$1，不接受你的提议，再见。但“接受者3”检查了一下自己记录的贿赂金额，目前记录的贿赂金额就是$1，于是接受了这一提议，并把1号提议记录在案。


![image](http://bloghello.oursnail.cn/zk3-8.png)

9）到这里，“提议者1”已经得到两个接受者的赞同，已经得到了多数“接受者”的赞同。于是“提议者1”确定1号提议最终通过。



10）此时“提议者2”发现1号提议已经被通过了，为了最快达成一致，那么他就默默地将自己的提议也改为与1号提议一致，然后开始向“接受者1”“接受者2”发起提议（提议内容仍然是1号提议），并带着信息：之前自己已贿赂过$2。

![image](http://bloghello.oursnail.cn/zk3-9.png)

11）这时“接受者1”“接受者2”收到“提议者2”的提议后，照例先比对一下贿赂金额，比对发现“提议者2”之前已贿赂$2，并且自己记录的贿赂金额也是$2，所以接受他的提议，也就是都接受1号提议。

![image](http://bloghello.oursnail.cn/zk3-10.png)

12）于是，“提议者2”也拿到了多数派的意见，最终通过的也是1号提议。

回到上面的第5）步，如果“提议者2”第一次先去贿赂“接受者2”“接受者3”会发生什么？那很可能1号提议就不会成为最终选出的提议。因为当“提议者2”先贿赂到了“接受者2”“接受者3”，那等“提议者1”带着议题再去找这两位的时候，就会因为之前贿赂的钱少（$1<$2）而被拒绝。所以，这也就是刚才讲到可能存在博弈的地方：a.“提议者1”要赶在“提议者2”贿赂到“接受者2”“接受者3”之前，让“接受者2”“接受者3”接受自己的意见，否则“提议者1”会因为钱少而被拒绝；b.“提议者2”要赶在“提议者1”之前贿赂到“接受者”，否则“提议者2”即便贿赂成功，也要默默的将自己的提议改为“提议者1”的提议。但你往后推演会发现，无论如何，总会有一个“提议者”的提议获得多数票而胜出。


## 五、总结

好啦，故事到这里基本讲述完了，咱们来总结一下，其实Paxos算法就下面这么几个原则：

- Paxos算法包括两个阶段：第一个阶段主要是贿选，还没有提出提议；第二个阶段主要根据第一阶段的结果，明确接受谁的提议，并明确提议的内容是什么（这个提议可能是贿选胜出“提议者”自己的提议，也可能是前任意见领袖的提议，具体是哪个提议，见下面第3点原则）。
- 编号（贿赂金额）很重要，无论在哪个阶段，编号（贿赂金额）小的，都会被鄙视（被拒绝）。
- 在第一阶段中，一旦“接受者”已经接受了之前意见领袖的提议，那后面再来找这个“接受者”的“提议者”，即便在贿赂中胜出，也要被洗脑，默默将自己的提议改为前任意见领袖的提议，然后他会在第二阶段提出该提议（也就是之前意见领袖的提议，以力争让大家的意见趋同）。如果“接受者”之前没有接受过任何提议，那贿选胜出的“提议者”就可以提出自己的提议了。



还有一个问题需要考量，假如`proposer A`发起ID为n的提议，在提议未完成前`proposer B`又发起ID为n+1的提议，在n+1提议未完成前`proposer C`又发起ID为n+2的提议…… 如此`acceptor`不能完成决议、形成活锁(`livelock`)，虽然这不影响一致性，但我们一般不想让这样的情况发生。解决的方法是从`proposer`中选出一个`leader`，提议统一由`leader`发起。


- [如何浅显易懂地解说 Paxos 的算法？](https://www.zhihu.com/question/19787937)
- [共识算法之：Paxos](https://zhuanlan.zhihu.com/p/44997221)
- [讲一个关于paxos的故事](http://blog.51cto.com/9587671/2286358)
- [浅显易懂地解读Paxos算法](https://zhuanlan.zhihu.com/p/21895686)
- [分布式系统理论进阶 - Paxos](http://www.cnblogs.com/bangerlee/p/5655754.html)
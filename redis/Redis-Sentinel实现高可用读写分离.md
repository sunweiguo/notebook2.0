title: Redis-Sentinel实现高可用读写分离
tag: redis
---
本文为redis学习笔记的第七篇文章。Redis Sentinel 是一个分布式系统，你可以在一个架构中运行多个 Sentinel 进程，这些进程使用流言协议（gossip protocols)来接收关于主服务器是否下线的信息，并使用投票协议（agreement protocols）来决定是否执行自动故障迁移，以及选择哪个从服务器作为新的主服务器。
<!-- more -->

虽然 `Redis Sentinel` 是一个单独的可执行文件 `redis-sentinel` ，但实际上它只是一个运行在特殊模式下的 `Redis` 服务器，你可以在启动一个普通 `Redis` 服务器时通过给定 `–sentinel` 选项来启动 `Redis Sentinel` 。

- 启动方式一：使用`sentinel`可执行文件 `redis-sentinel` 程序来启动 `Sentinel` 系统，命令如下：


```
redis-sentinel /path/to/sentinel.conf
```


- `sentinel`只是运行在特殊模式下的`redis`服务器，你可以用启动`redis`服务的命令来启动一个运行在 `Sentinel` 模式下的 `Redis` 服务器：


```
redis-server /path/to/sentinel.conf --sentinel
```


## 1. redis sentinel

首先来看看什么是 `redis sentinel`，中文翻译是redis哨兵。顾名思义，哨兵是站岗监督突发情况的，那么这里具体的功能上很类似：

- 监控：`Sentinel` 会不断地检查你的主服务器和从服务器是否运作正常。
- 提醒：当被监控的某个 `Redis` 服务器出现问题时，`Sentinel` 可以通过 API 向管理员或者其他应用程序发送通知。
- 自动故障迁移：当一个主服务器不能正常工作时，`Sentinel` 会开始一次自动故障迁移操作，它会将失效主服务器的其中一个从服务器升级为新的主服务器，并让失效主服务器的其他从服务器改为复制新的主服务器；当客户端试图连接失效的主服务器时，集群也会向客户端返回新主服务器的地址，使得集群可以使用新主服务器代替失效服务器。



![image](http://bloghello.oursnail.cn/redis_sentinel%E7%BB%93%E6%9E%84.png)

其中总结一下故障转移的基本原理：

- 多个`sentinel`发现并确认`master`有问题
- 选举出一个`sentinel`作为领导
- 选出一个可以成为新的`master`的`slave`
- 通知其他的`slave`称为新的`master`的`slave`
- 通知客户端主从变化
- 等待老的`master`复活称为新的`master`的`slave`


也支持多个`master-slave`结构：


![image](http://bloghello.oursnail.cn/%E5%A4%9A%E4%B8%AAmaster_slave.png)

## 2. 安装与配置

1. 配置开启主从节点
2. 配置开启`sentinel`监控主节点（`sentinel`是特殊的`redis`）
3. 实际应该多台机器，但是演示方便，只用一台机器来搭建
4. 详细配置节点


本地安装的结构图：

![image](http://bloghello.oursnail.cn/%E6%95%88%E6%9E%9C%E5%9B%BE.png)

对于`master:redis-7000.conf`配置：

![image](http://bloghello.oursnail.cn/redis-7000.conf.png)


```
port 7000
daemonize yes
pidfile /usr/local/redis/data/redis-7000.pid
logfile "7000.log"
dir "/usr/local/redis/data"
```


对于`slave:redis-7001`和`redis-7002`配置：

![image](http://bloghello.oursnail.cn/redis-slave.png)


```
port 7001
daemonize yes
pidfile /usr/local/redis/data/redis-7001.pid
logfile "7001.log"
dir "/usr/local/redis/data"
slaveof 127.0.0.1 7000
```
启动`redis`服务：

```
redis-server ../config/redis-7000.conf
```



访问7000端口的`master redis`:

```
redis-cli -p 7000 info replication
```

显示他有两个从节点：


```
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=7002,state=online,offset=99550,lag=1
slave1:ip=127.0.0.1,port=7001,state=online,offset=99816,lag=0
master_repl_offset:99816
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:99815
```


对于`sentinel`主要配置：

![image](http://bloghello.oursnail.cn/sentinel%E4%B8%BB%E8%A6%81%E9%85%8D%E7%BD%AE.png)


`master sentinel config`:
```
port 26379
daemonize yes
dir "/usr/local/redis/data"
logfile "26379.log"
sentinel monitor mymaster 127.0.0.1 7000 2
...
```

启动`redis sentinel`:

```
redis-sentinel ../config/redis-sentinel-26379.conf
```


访问26379 `redis sentinel master`:


```
redis-cli -p 26379 info sentinel
```

显示：


```
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
master0:name=mymaster,status=ok,address=127.0.0.1:7000,slaves=2,sentinels=3
```


```
查看这六个进程是否都起来了：ps -ef | grep redis
```

注意，如果上面是配置在虚拟机的话，需要将127.0.0.1改为虚拟机的ip，要不然找不着。


## 3. 故障转移演练

#### 3.1 java客户端程序

`JedisSentinelPool`只是一个配置中心，不需要具体连接某个`redis`，注意它不是代理。

```java
private Logger logger = LoggerFactory.getLogger(AppTest.class);

@Test
public void test4(){
    //哨兵配置，我们访问redis，就通过sentinel来访问
    String masername = "mymaster";
    Set<String> sentinels = new HashSet<>();
    sentinels.add("10.128.24.176:26379");
    sentinels.add("10.128.24.176:26380");
    sentinels.add("10.128.24.176:26381");

    JedisSentinelPool sentinelPool = new JedisSentinelPool(masername,sentinels);

    //一个while死循环，每隔一秒往master塞入一个值，并且日志打印
    while (true){
        Jedis jedis = null;
        try{
            jedis = sentinelPool.getResource();

            int index = new Random().nextInt(100000);
            String key = "k-" + index;
            String value = "v-" + index;
            jedis.set(key,value);
            logger.info("{}  value is {}",key,jedis.get(key));

            TimeUnit.MILLISECONDS.sleep(1000);
        }catch (Exception e){
            logger.error(e.getMessage(),e);
        }finally {
            if(jedis != null){
                jedis.close();
            }
        }
    }
}
```

`maven`依赖是：

```xml
<!--jedis-->
<dependency>
  <groupId>redis.clients</groupId>
  <artifactId>jedis</artifactId>
  <version>2.9.0</version>
</dependency>
<!--slf4j日志接口-->
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>1.7.6</version>
</dependency>
<!--logback日志实现-->
<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>1.1.1</version>
</dependency>
```

启动程序，发现是正常写入：

```
16:16:01.424 [main] INFO  com.njupt.swg.AppTest - k-54795  value is v-54795
16:16:02.426 [main] INFO  com.njupt.swg.AppTest - k-55630  value is v-55630
16:16:03.429 [main] INFO  com.njupt.swg.AppTest - k-70642  value is v-70642
16:16:04.430 [main] INFO  com.njupt.swg.AppTest - k-42978  value is v-42978
16:16:05.431 [main] INFO  com.njupt.swg.AppTest - k-96297  value is v-96297
16:16:06.433 [main] INFO  com.njupt.swg.AppTest - k-4220  value is v-4220
16:16:07.435 [main] INFO  com.njupt.swg.AppTest - k-34103  value is v-34103
16:16:08.436 [main] INFO  com.njupt.swg.AppTest - k-9177  value is v-9177
16:16:09.437 [main] INFO  com.njupt.swg.AppTest - k-24389  value is v-24389
16:16:10.439 [main] INFO  com.njupt.swg.AppTest - k-32325  value is v-32325
16:16:11.440 [main] INFO  com.njupt.swg.AppTest - k-68538  value is v-68538
16:16:12.441 [main] INFO  com.njupt.swg.AppTest - k-36233  value is v-36233
16:16:13.443 [main] INFO  com.njupt.swg.AppTest - k-305  value is v-305
16:16:14.444 [main] INFO  com.njupt.swg.AppTest - k-59279  value is v-59279
```

我们将现在的端口为7000的`redis master` 给`kill`掉

> kill -9 master的pid

我们会发现：客户端报异常，但是在大概十几秒之后，就继续正常塞值了。原因是服务端的哨兵机制的选举`matser`需要一定的时间。


## 4. 三个定时任务

#### 4.1 每10秒每个sentinel对master和slave执行Info

- 发现`slave`节点
- 确认主从关系


![image](http://bloghello.oursnail.cn/%E7%AC%AC%E4%B8%80%E4%B8%AA%E5%AE%9A%E6%97%B6.png)

#### 4.2 每2秒每个sentinel通过master节点的channel交换信息(pub/sub)

- 通过`__sentinel__`:hello进行频道交互
- 交互对节点的“看法”和自身信息

![image](http://bloghello.oursnail.cn/%E7%AC%AC%E4%BA%8C%E4%B8%AA%E5%AE%9A%E6%97%B6.png)

#### 4.3 每1秒每个`sentinel`对其他`sentinel`和`redis`执行`ping`

- 心跳监测，失败判定依据

![image](http://bloghello.oursnail.cn/%E7%AC%AC%E4%B8%89%E4%B8%AA%E5%AE%9A%E6%97%B6.png)


## 5. 主观下线和客观下线

对于之前的`Sentinel`配置文件中有两条配置：

监控`master redis`节点，这里是当超过两个`sentinel`认为`master`挂了，则认为`master`挂了。

> `sentinel monitor <masterName> <masterIp> <msterPort> <quorum>`
>
> `sentinel monitor mymaster 127.0.0.1 6379 2`


这里是每秒`sentinel`都会去`Ping`周围的`master redis`，超过30秒没有任何响应，说明其挂了。

> `sentinel down-after-milliseconds <masterName> <timeout>`
>
> `sentinel down-after-milliseconds mymaster 300000`


#### 5.1 主观下线

主观下线：每个`sentinel`节点对`Redis`节点失败的“偏见”

这是一种主观下线。因为在复杂的网络环境下，这个`sentinel`与这个`master`不通，但是`master`与其他的`sentinel`都是通的呢？所以是一种“偏见”

这是依靠的第三种定时：每秒去ping一下周围的`sentinel`和`redis`。对于`slave redis`,可以使用这个主观下线，因为他不需要进行故障转移。

#### 5.2 客观下线

客观下线：所有`sentinel`节点对`master Redis`节点失败“达成共识”（超过`quorum`个则统一）

这是依靠的第二种定时：每两秒，`sentinel`之间进行“商量”，传递的消息是:`sentinel is-master-down-by-addr`

对于`master redis`的下线，必须要达成共识才可以，因为涉及故障转移，仅仅依靠一个`sentinel`判断是不够的。

## 6. 领导者选举

原因：只有一个`sentinel`节点完成故障转移

选举：通过`sentinel is-master-down-by-addr`命令都希望成为领导者

- 每个做主观下线的`sentinel`节点向其他`sentinel`节点发送命令，要求将它设置为领导者
- 收到命令的`sentinel`节点如果还没有同意过其他`semtinel`节点发送的命令，那么将同意该请求，否则拒绝
- 如果该`sentinel`节点发现自己的票数已经超过`sentinel`集合半数并且超过`quorum`，那么它将成为领导者。
- 如果此过程中多个`sentinel`节点成为了领导者，那么将等待一段时间重新进行选举

## 7. 故障转移

- 从`slave`节点中选出一个“合适的”节点作为新的`master`节点
- 对上述的`slave`节点执行“`slaveof no one`”命令使其成为`master`节点
- 向剩余的`slave`节点发送命令，让它们成为新`master`节点的`slave`节点，复制规则和`parallel-syncs`参数一样
- 更新对原来的`master`节点配置为`slave`，并保持着对其“关注”，当恢复后命令他去复制新的`master`节点


那么，如何选择“合适”的`slave`节点呢？

- 选择`slave-priority`(`slave`节点优先级)最高的`slave`节点，如果存在则返回，不存在则继续。
- 选择复制偏移量最大的`slave`节点(复制得最完整)，如果存在则返回，不存在则继续
- 选择`run_id`最小的`slave`节点(最早的节点)
 

## 8. 节点下线

主节点下线：`sentinel failover <masterName>`

从节点下线要注意读写分离问题。

## 9. 总结与思考

> `redis sentinel`是`redis`高可用实现方案：故障发现、故障自动转移、配置中心、客户端通知。

> `redis sentinel`从`redis2.8`版本才正式生产可用，之前版本不可生产用。

> 尽可能在不同物理机上部署`redis sentinel`所有节点。

> `redis sentinel`中的`sentinel`节点个数应该大于等于3且最好是奇数。

> `redis sentinel`中的数据节点和普通数据节点没有区别。每个`sentinel`节点在本质上还是一个`redis`实例，只不过和`redis`数据节点不同的是，其主要作用是监控`redis`数据节点

> 客户端初始化时连接的是`sentinel`节点集合，不再是具体的`redis`节点，但`sentinel`只是配置中心不是代理。

> `redis sentinel`通过三个定时任务实现了`sentinel`节点对于主节点、从节点、其余`sentinel`节点的监控。

> `redis sentinel`在对节点做失败判定时分为主观下线和客观下线。

> 看懂`redis sentinel`故障转移日志对于`redis sentine`l以及问题排查非常有用。

> `redis sentinel`实现读写分离高可用可以依赖`sentinel`节点的消息通知，获取`redis`数据节点的状态变化。



`redis sentinel`可以实现高可用的读写分离，高可用体现在故障转移，那么实现高可用的基础就是要有从节点，主从节点还实现了读写分离，减少`master`的压力。但是如果是从节点下线了，`sentinel`是不会对其进行故障转移的，并且连接从节点的客户端也无法获取到新的可用从节点，而这些问题在`Cluster`中都得到了有效的解决。

对于性能提高、容量扩展的时候，这种方式是比较复杂的，比较推荐的是使用集群，就是下面讨论的`redis cluster`!

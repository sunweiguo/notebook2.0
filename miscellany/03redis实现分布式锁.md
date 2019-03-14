title: redis实现分布式锁
tag: miscellany
---
为了讲解redis分布式锁，我将引入一个场景：定时关单。因为往往订单服务是一个集群，那么定时器会同时触发这些集群去取消订单，显然是浪费机器资源的，所以目的是：只让其中一台机器去执行取消订单即可。这里可以用分布式锁来实现。
<!-- more -->

项目是从练手项目中截取出来的，框架是基于`SSM`的`XML`形式构成，所以下面还涉及一点`XMl`对于定时器`spring schedule`的配置内容。

## 1、引入目标

定时自动对超过两个小时还未支付的订单对其进行取消，并且重置库存。

## 2、配置

首先是`spring`配置文件引入`spring-schedule`

```xml
xmlns:task="http://www.springframework.org/schema/task"
...
http://www.springframework.org/schema/task
http://www.springframework.org/schema/task/spring-task.xsd
...
<task:annotation-driven/>
```

补充：针对`applicationContext-datasource.xml`中的`dataSource`读取配置文件的信息无法展现的问题，在`spring`的配置文件中增加一条配置：

```xml
<context:property-placeholder location="classpath:datasource.properties"/>
```

## 3、定时调度代码

此代码的主要功能是：定时调用取消订单服务。

```java
@Component
@Slf4j
public class CloseOrderTask {
    @Autowired
    private OrderService orderService;

    @Scheduled(cron = "0 */1 * * * ?")//每隔一分钟执行，一分钟的整数倍的时候执行
    public void closeOrderTaskV1(){
        log.info("关闭订单定时任务启动");
        int hour = Integer.parseInt(PropertiesUtil.getProperty("close.order.task.time.hour","2"));
        orderService.closeOrder(hour);
        log.info("关闭订单定时任务结束");
    }
}
```

> `@Component`一定要加，否则`spring`扫描不到。

> `close.order.task.time.hour` 也是配置在`snailmall.properties`中的，这里配置的是默认的2，即两个小时，下订单超过两个小时仍然不支付，就取消该订单。

对于`orderService`里面的具体方法：

这里是关单的具体逻辑，细节是行锁。这段代码只要知道他是具体关单的逻辑即可，不需要仔细了解代码。
```java
@Override
public void closeOrder(int hour) {
    Date closeDateTime = DateUtils.addHours(new Date(),-hour);
    //找到状态为未支付并且下单时间是早于当前检测时间的两个小时的时间,就将其置为取消
    //SELECT <include refid="Base_Column_List"/> from mmall_order WHERE  status = #{status} <![CDATA[ and create_time <= #{date} ]]> order by create_time desc
    List<Order> orderList = orderMapper.selectOrderStatusByCreateTime(Const.OrderStatusEnum.NO_PAY.getCode(),DateTimeUtil.dateToStr(closeDateTime));
    for(Order order:orderList){
        List<OrderItem> orderItemList = orderItemMapper.getByOrderNo(order.getOrderNo());
        for(OrderItem orderItem:orderItemList){
            //一定要用主键where条件，防止锁表。同时必须是支持MySQL的InnoDB.
            Integer stock = productMapper.selectStockByProductId(orderItem.getProductId());
            if(stock == null){
                continue;
            }
            //更新产品库存
            Product product = new Product();
            product.setId(orderItem.getProductId());
            product.setStock(stock+orderItem.getQuantity());
            productMapper.updateByPrimaryKeySelective(product);
        }
        //关闭order
        //UPDATE mmall_order set status = 0 where id = #{id}
        orderMapper.closeOrderByOrderId(order.getId());
        log.info("关闭订单OrderNo:{}",order.getOrderNo());
    }
}
```


这样，`debug`启动项目，一分钟后就会自动执行`closeOrderTaskV1`方法了。找一个未支付的订单，进行相应测试。

## 4、存在的问题

经过实验发现，同时部署两台`tomcat`服务器，执行定时任务的时候是两台都同时执行的，显然不符合我们集群的目标，我们只需要在同一时间只有一台服务器执行这个定时任务即可。那么解决方案就是引入`redis`分布式锁。

`redis`实现分布式锁，核心命令式`setnx`命令。所以阅读下面，您需要对`redis`分布式锁的基本实现原理必须要先有一定的认识才行。

## 5、第一种方案

![image](http://bloghello.oursnail.cn/redis%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81V1.png)

- 第一步：`setnx`进去，如果成功，说明塞入`redis`成功，抢占到锁

- 第二步：抢到锁之后，先设置一下过期时间，即后面如果执行不到`delete`，也会将这个锁自动释放掉，防止死锁

- 第三步：关闭订单，删除`redis`锁

- 存在的问题：如果因为`tomcat`关闭或`tomcat`进程在执行`closeOrder()`方法的时候，即还没来得及设置锁的过期时间的时候，这个时候会造成死锁。需要改进。

```java
//第一个版本，在突然关闭tomcat的时候有可能出现死锁
@Scheduled(cron = "0 */1 * * * ?")//每隔一分钟执行，一分钟的整数倍
public void closeOrderTaskV2(){
    log.info("关闭订单定时任务启动");
    //设置锁，value是用当前时间+timeout进行设置的
    long timeout = Long.parseLong(PropertiesUtil.getProperty("lock.timeout"));
    Long setnxResult = RedisShardPoolUtil.setnx(Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK,String.valueOf(System.currentTimeMillis()+timeout));
    if(setnxResult != null && setnxResult.intValue() ==1){
        //说明被当前的tomcat进程抢到锁，下面就可以关闭订单
        closeOrder(Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK);
    }else {
        log.info("没有获取分布式锁：{}",Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK);
    }
    log.info("关闭订单定时任务结束");
}

private void closeOrder(String lockName) {
    //给锁一个过期时间，如果因为某个原因导致下面的锁没有被删除，造成死锁
    RedisShardPoolUtil.expire(lockName,50);
    log.info("获取{}，ThreadName:{}",Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK,Thread.currentThread().getName());
    int hour = Integer.parseInt(PropertiesUtil.getProperty("close.order.task.time.hour","2"));
    orderService.closeOrder(hour);
    //关闭订单之后就立即删除这个锁
    RedisShardPoolUtil.del(lockName);
    log.info("释放{}，ThreadName:{}",Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK,Thread.currentThread().getName());
    System.out.println("=============================================");
}
```

## 6、改进

![image](http://bloghello.oursnail.cn/redis%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81V2.png)

图看不清，可以重新打开一个窗口看。具体的逻辑代码：
```java
@Scheduled(cron = "0 */1 * * * ?")//每隔一分钟执行，一分钟的整数倍
public void closeOrderTaskV3(){
    log.info("关闭订单定时任务启动");
    //设置锁，value是用当前时间+timeout进行设置的
    long timeout = Long.parseLong(PropertiesUtil.getProperty("lock.timeout"));
    Long setnxResult = RedisShardPoolUtil.setnx(Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK,String.valueOf(System.currentTimeMillis()+timeout));
    if(setnxResult != null && setnxResult.intValue() ==1){
        //说明被当前的tomcat进程抢到锁，下面就可以关闭订单
        closeOrder(Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK);
    }else {
        //在没有拿到锁的情况下，也要进行相应的判断，确保不死锁
        String lockValueStr = RedisShardPoolUtil.get(Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK);
        //如果判断锁是存在的并且现在已经超时了，那么我们这个进程就有机会去占有这把锁
        if(lockValueStr != null && System.currentTimeMillis() > Long.parseLong(lockValueStr)){
            //当前进程进行get set操作，拿到老的key，再塞进新的超时时间
            String getSetResult = RedisShardPoolUtil.getset(Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK,String.valueOf(System.currentTimeMillis()+timeout));
            //如果拿到的是空的，说明老的锁已经释放，那么当前进程有权占有这把锁进行操作；
            //如果拿到的不是空的，说明老的锁仍然占有，并且这次getset拿到的key与上面查询get得到的key一样的话，说明没有被其他进程刷新，那么本进程还是有权占有这把锁进行操作
            if(getSetResult == null || (getSetResult != null && StringUtils.equals(lockValueStr,getSetResult))){
                closeOrder(Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK);
            }else {
                log.info("没有获取分布式锁：{}",Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK);
            }            }else {
            log.info("没有获取分布式锁：{}",Const.REDIS_LOCK.CLOSE_ORDER_TASK_LOCK);
        }
    }
    log.info("关闭订单定时任务结束");
}
```

这样两次的防死锁措施，不仅可以防止死锁，还可以提高效率。

## 7、扩展

###### mysql四种事务隔离机制

1. `read uncommitted`:读取未提交内容

两个线程，其中一个线程执行了更新操作，但是没有提交，另一个线程在事务内就会读到该线程未提交的数据。

2. `read committed`:读取提交内容（不可重复读）

针对第一种情况，一个线程在一个事务内不会读取另一个线程未提交的数据了。但是，读到了另一个线程更新后提交的数据，也就是说重复读表的时候，数据会不一致。显然这种情况也是不合理的，所以叫不可重复读。

3. `repeatable read`:可重复读（默认）

可重复读，显然解决2中的问题，即一个线程在一个事务内不会再读取到另一个线程提交的数据，保证了该线程在这个事务内的数据的一致性。

对于某些情况，这种方案会出现幻影读，他对于更新操作是没有任何问题的了，但是对于插入操作，有可能在一个事务内读到新插入的数据（但是MySQL中用多版本并发控制机制解决了这个问题），所以默认使用的就是这个机制，没有任何问题。

4. `serializable`:序列化

略。

###### 存储引擎

`MySQL`默认使用的是`InnoDB`，支持事务。还有例如`MyISAM`,这种存储引擎不支持事务，只支持只读操作，在用到数据的修改的地方，一般都是用默认的`InnoDB`存储引擎。

###### 索引的一个注意点

一般类型为`normal`和`unique`，用`btree`实现，对于联合索引(字段1和字段2)，在执行查询的时候，例如
```sql
select * from xxx where 字段1="xxx" ... 
```
是可以利用到索引的高性能查询的，但是如果是 

```sql
select * from xxx where 字段2="xxx" ...
```
效率跟普通的查询时一样的，因为用索引进行查询，最左边的那个字段必须要有，否则无效。


扩展的内容知识顺便提一下，在数据库这一块，会详细介绍一下。



title: Curator
tag: miscellany
---

从技术角度出发，注册一个网站，再高并发的时候，有可能出现用户名重复这样的问题（虽然一般情况下不会出现这种问题），如何解决呢？

<!-- more -->

从数据库角度，对于单表，我可以用`select .. for update`悲观锁实现，或者用version这种乐观锁的思想。

更好的方法是将这个字段添加唯一索引，用数据库来保证不会重复。一旦插入重复，那么就会抛出异常，程序就可以捕获到。

但是，假如我们这里分表了，以上都是针对单表，第一种方案是锁表，不行，设置唯一索引是没有用。怎么办呢？

解决方案：用ZK做一个分布式锁。

首先准备一个ZK客户端，用的是`Curator`来连接我们的ZK：

```java
@Component
public class ZkClient {

    @Autowired
    private Parameters parameters;

    @Bean
    public CuratorFramework getZkClient(){

        CuratorFrameworkFactory.Builder builder= CuratorFrameworkFactory.builder()
                .connectString(parameters.getZkHost())
                .connectionTimeoutMs(3000)
                .retryPolicy(new RetryNTimes(5, 10));
        CuratorFramework framework = builder.build();
        framework.start();
        return framework;

    }

}
```

注册用一个分布式锁来控制：

```java
@Override
public void registerUser(User user) throws Exception {
    InterProcessLock lock = null;
    try{
        lock = new InterProcessMutex(zkClient, Constants.USER_REGISTER_DISTRIBUTE_LOCK_PATH);
        boolean retry = true;
        do{
            if (lock.acquire(3000, TimeUnit.MILLISECONDS)){
                //查询重复用户
                User repeatedUser = userMapper.selectByEmail(user.getEmail());
                if(repeatedUser!=null){
                    throw  new MamaBuyException("用户邮箱重复");
                }
                user.setPassword(passwordEncoder.encode(user.getPassword()));
                user.setNickname("码码购用户"+user.getEmail());
                userMapper.insertSelective(user);
                //跳出循环
                retry = false;
            }
            //可以适当休息一会...也可以设置重复次数，不要无限循环
        }while (retry);
    }catch (Exception e){
        log.error("用户注册异常",e);
        throw  e;
    }finally {
        if(lock != null){
            try {
                lock.release();
                log.info(user.getEmail()+Thread.currentThread().getName()+"释放锁");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

思路非常简单，就是先尝试上锁，即`acquire`，但是有可能失败，所以这里用一个超时时间，即`3000ms`之内上不了锁就失败，进入下一次循环。最后释放锁即可。

ok，这里要来说说ZK实现分布式锁了。这里用了开源客户端`Curator`，他对于实现分布式锁进行了封装，但是，我还是想了解一下它的实现原理：

每个客户端对某个方法加锁时，在zookeeper上的与该方法对应的指定节点的目录下，生成一个唯一的瞬时有序节点。 判断是否获取锁的方式很简单，只需要判断有序节点中序号最小的一个。 当释放锁的时候，只需将这个瞬时节点删除即可。同时，其可以避免服务宕机导致的锁无法释放，而产生的死锁问题。

也就是说，最小的那个节点就是Leader，进来判断是不是为那个节点，是的话就可以获取到锁，反之不行。

> 为什么不能通过大家一起创建节点，如果谁成功了就算获取到了锁。 多个client创建一个同名的节点，如果节点谁创建成功那么表示获取到了锁，创建失败表示没有获取到锁。

答：使用临时顺序节点可以保证获得锁的公平性，及谁先来谁就先得到锁，这种方式是随机获取锁，会造成无序和饥饿。


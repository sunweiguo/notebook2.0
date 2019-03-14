title: Spring Session
tag: miscellany
---

在单体应用中，我们经常用http session去管理用户信息，但是到了分布式环境下，显然是不行的，因为session对于不同的机器是隔离的，而http本身是无状态的，那么就无法判断出用户在哪一个服务器上登陆的。这个时候就需要有一个独立的地方存储用户session。spring session可以做到无代码侵入的方式实现分布式session存储。

<!-- more -->

在`spring boot`开发中，我们先注册相应`bean`并且打开相应的注解：

```java
@Configuration
@EnableRedisHttpSession //(maxInactiveIntervalInSeconds = 604800)//session超时
public class HttpSessionConfig {

    @Autowired
    private Parameters parameters;


    @Bean
    public HttpSessionStrategy httpSessionStrategy(){
        return  new HeaderHttpSessionStrategy();
    }

    @Bean
    public JedisConnectionFactory connectionFactory(){

        JedisConnectionFactory connectionFactory = new JedisConnectionFactory();

        String redisHost = parameters.getRedisNode().split(":")[0];
        int redisPort = Integer.valueOf(parameters.getRedisNode().split(":")[1]);

        connectionFactory.setTimeout(2000);
        connectionFactory.setHostName(redisHost);
        connectionFactory.setPort(redisPort);
//        connectionFactory.setPassword(parameters.getRedisAuth());

        return connectionFactory;
    }
}
```
ok，这样子其实就配置好了，一开始我也云里雾里的，这是啥玩意？

其实官网的文档中讲的是最准确的。所以还是官网看看吧！

ok，来spring session的官网(https://spring.io/projects/spring-session)来看看把，我们来看看1.3.4GA版本的文档(https://docs.spring.io/spring-session/docs/1.3.4.RELEASE/reference/html5/#httpsession-rest).


spring session可以存在很多介质中，比如我们的数据源，比如redis，甚至是mongodb等。但是我们常用的是存在redis中，结合redis的过期机制来做。

所以其实我们只要关心如何跟redis整合，以及restful接口。

我们可以看到一开始文档就告诉我们要配置一下`HttpSessionStrategy`和存储介质。从`HttpSessionStrategy`语义就能大致看出配置的是它的策略，是基于`header`的策略。这个是什么意思，下面会提到。

![image](http://bloghello.oursnail.cn/mama4-1.png)

那么我们就来看看文档吧！

![image](http://bloghello.oursnail.cn/mama4-2.png)

好了，我们知道了它的基本原理，下面来看看是如何在restful接口中实现用户session的管理的：

![image](http://bloghello.oursnail.cn/mama4-3.png)

也就是说要想在restful接口应用中用这种方式，直接告诉spring session:`return  new HeaderHttpSessionStrategy();`即可。进入源码我们就会知道，它默认给这个header里面放置的一条类似于token的名字是`private String headerName = "x-auth-token";`。

那么在用户登陆成功之后，到底存到是什么呢，先来看看响应数据的header里面是什么：

![image](http://bloghello.oursnail.cn/mama4-4.png)

这一串数字正好可以跟redis中对应上，我们可以先来redis中看看到底在里面存储了啥玩意：

![image](http://bloghello.oursnail.cn/mama4-5.png)


我们已经看到了想要看到的一串字符串，这里解释一下`redis`中存储的东西：

* `spring:session`是默认的`Redis HttpSession`前缀（`redis`中，我们常用’:’作为分割符）
* 每一个`session`都会有三个相关的`key`，第一个`key`(`spring:session:sessions:37...`)最为重要，它是一个`HASH`数据结构，将内存中的`session`信息序列化到了`redis`中。如本项目中用户信息,还有一些`meta`信息，如创建时间，最后访问时间等。
* 另外两个key，一个是`spring:session:expiration`，还有一个是`spring:session:sessions:expires`，前者是一个SET类型，后者是一个STRING类型，可能会有读者发出这样的疑问，redis自身就有过期时间的设置方式TTL，为什么要额外添加两个key来维持session过期的特性呢？redis清除过期key的行为是一个异步行为且是一个低优先级的行为，用文档中的原话来说便是，可能会导致session不被清除。于是引入了专门的expiresKey，来专门负责session的清除，包括我们自己在使用redis时也需要关注这一点。

这样子，就可以用独立的`redis`来存储用户的信息，通过前端传来的`header`里面的`token`，就可以到`redis`拿出当前登陆用户的信息了。

OK，在解决了`spring session`的问题之后，下面就可以来实现登陆啦：

controller:

```java
@RequestMapping("login")
public ApiResult login(@RequestBody @Valid User user, HttpSession session){
    ApiResult<UserElement> result = new ApiResult<>(Constants.RESP_STATUS_OK,"登录成功");

    UserElement ue= userService.login(user);
    if(ue != null){
        if(session.getAttribute(Constants.REQUEST_USER_SESSION) == null){
            session.setAttribute(Constants.REQUEST_USER_SESSION,ue);
        }
        result.setData(ue);
    }

    return result;
}
```

就跟以前一样，将session直接存进去就可以了。


```java
@Override
public UserElement login(User user) {
    UserElement ue = null;
    User userExist = userMapper.selectByEmail(user.getEmail());
    if(userExist != null){
        //对密码与数据库密码进行校验
        boolean result = passwordEncoder.matches(user.getPassword(),userExist.getPassword());
        if(!result){
            throw new MamaBuyException("密码错误");
        }else{
            //校验全部通过，登陆通过
            ue = new UserElement();
            ue.setUserId(userExist.getId());
            ue.setEmail(userExist.getEmail());
            ue.setNickname(userExist.getNickname());
            ue.setUuid(userExist.getUuid());
        }
    }else {
        throw new MamaBuyException("用户不存在");
    }
    return ue;
}
```

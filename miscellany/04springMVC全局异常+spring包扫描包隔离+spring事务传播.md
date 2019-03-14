title: springMVC全局异常+spring包扫描包隔离+spring事务传播
tag: miscellany
---
在开发中，springMVC全局异常+spring包扫描包隔离+spring事务传播这三个不可能不会遇到。下面来好好说说他们吧。
<!-- more -->

## 1、全局异常引入原因

假设在我们的`login.do`的`controller`方法中第一行增加一句：

```java
int i = 1/0;
```

重新启动服务器进行用户登录操作，那么就会抛出异常：


```java
java.lang.ArithmeticException: / by zero
    com.swg.controller.portal.UserController.login(UserController.java:37)
    ...其他的堆栈信息
```

这些信息会直接显示在网页上，如果是关于数据库的错误，同样，会详细地将数据库中的字段都显示在页面上，这对于我们的项目来说是存在很大的安全隐患的。这个时候，需要用全局异常来处理，如果发生异常，我们就对其进行拦截，并且在页面上显示我们给出的提示信息。

对于`SpringBoot`，一般全局异常是:

```java
@ControllerAdvice
@ResponseBody
@Slf4j
public class ExceptionHandlerAdvice {
    @ExceptionHandler(Exception.class)
    public ServerResponse handleException(Exception e){
        log.error(e.getMessage(),e);
        return ServerResponse.createByErrorCodeMessage(Constants.RESP_STATUS_INTERNAL_ERROR,"系统异常，请稍后再试");
    }

    @ExceptionHandler(SnailmallException.class)
    public ServerResponse handleException(SnailmallException e){
        log.error(e.getMessage(),e);
        return ServerResponse.createByErrorCodeMessage(e.getExceptionStatus(),e.getMessage());
    }

}
```


## 2、引入全局异常

```java
@Slf4j
@Component
public class ExceptionResolver implements HandlerExceptionResolver{
    @Override
    public ModelAndView resolveException(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) {
        log.error("exception:{}",httpServletRequest.getRequestURI(),e);
        ModelAndView mv = new ModelAndView(new MappingJacksonJsonView());
        mv.addObject("status",ResponseEnum.ERROR.getCode());
        mv.addObject("msg","接口异常，详情请查看日志中的异常信息");
        mv.addObject("data",e.toString());
        return mv;
    }
}
```

那么，再执行登陆操作之后，就不会在页面上直接显示异常信息了。有效地屏蔽了关键信息。

## 3、spring和springmvc配置文件的优化

##### 3.1 包隔离优化

在编写全局异常之前，先进行了包隔离和优化，一期中的扫描包的写法是：

```xml
<!--spring:-->
<context:component-scan base-package="com.swg" annotation-config="true"/>

<!--springmvc:-->
<context:component-scan base-package="com.swg" annotation-config="true"/>
```
即`spring`和`springmvc`扫描包下面的所有的`bean`和`controller`.优化后的代码配置：
```xml
#spring
<context:component-scan base-package="com.swg" annotation-config="true">
<!--将controller的扫描排除掉-->
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>

#springmvc
<context:component-scan base-package="com.swg.controller" annotation-config="true" use-default-filters="false">
<!--添加白名单，只扫描controller，总之要将service给排除掉即可-->
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```



这样做的原因是：`Spring`和`SpringMVC`是有父子容器关系的，而且正是因为这个才往往会出现包扫描的问题。

![image](http://bloghello.oursnail.cn/zaji4-1.png)


针对包扫描只要记住以下几点即可：

- `spring`是父容器，`springmvc`是子容器，子容器可以访问父容器的`bean`,父容器不能访问子容器的`bean`。 
- 只有顶级容器（`spring`）才有加强的事务能力，而`springmvc`容器的`service`是没有的。
- 如果`springmvc`不配置包扫描的话，页面404.

##### 3.2 事务的传播机制

针对事务，不得不展开说明spring事务的几种传播机制了。在 `spring` 的 `TransactionDefinition` 接口中一共定义了七种事务传播属性：

1. `PROPAGATION_REQUIRED` -- 支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择（默认）。 
2. `PROPAGATION_SUPPORTS` -- 支持当前事务，如果当前没有事务，就以非事务方式执行。 
3. `PROPAGATION_MANDATORY` -- 支持当前事务，如果当前没有事务，就抛出异常。 
4. `PROPAGATION_REQUIRES_NEW` -- 新建事务，如果当前存在事务，把当前事务挂起。 
5. `PROPAGATION_NOT_SUPPORTED` -- 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。 
6. `PROPAGATION_NEVER` -- 以非事务方式执行，如果当前存在事务，则抛出异常。 
7. `PROPAGATION_NESTED` -- 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与`PROPAGATION_REQUIRED`类似的操作。 

## 4、补充

`Spring`默认情况下，会对运行期例外(`RunTimeException`)，即`uncheck`异常，进行事务回滚。如果遇到`checked`异常就不回滚。如何改变默认规则：

- 让`checked`例外也回滚：在整个方法前加上 `@Transactional(rollbackFor=Exception.class)`
- 让`unchecked`例外不回滚： `@Transactional(notRollbackFor=RunTimeException.class)`
- 不需要事务管理的(只查询的)方法：`@Transactional(propagation=Propagation.NOT_SUPPORTED)`


## 5、那么什么是嵌套事务呢？

嵌套是子事务套在父事务中执行，子事务是父事务的一部分，在进入子事务之前，父事务建立一个回滚点，叫做`save point`，然后执行子事务，这个子事务的执行也算是父事务的一部分，然后子事务执行结束，父事务继续执行。重点在于那个`save point`，看以下几个问题：

**问题1：如果子事务回滚，会发生什么？**

父事务会回到进入子事务前建立的`save point`，然后尝试其他的事务或者其他的业务逻辑，父事务之前的操作不会受到影响，更不会自动回滚。

**问题2：如果父事务回滚，会发生什么？**

父事务回滚，子事务也会跟着回滚，为什么呢？因为父事务结束之前，子事务是不会提交的，我们说子事务是父事务的一部分，正是这个道理/

**问题3：父事务先提交，然后子事务再提交；还是子事务先提交，然后父事务再提交呢？**

答案是第二种情况，子事务是父事务的一部分，由父事务同意提交。


## 6、spring配置文件的一些理解：

> 容器

在`Spring`整体框架的核心概念中，容器是核心思想，就是用来管理`Bean`的整个生命周期的，而在一个项目中，容器不一定只有一个，`Spring`中可以包括多个容器，而且容器有上下层关系，目前最常见的一种场景就是在一个项目中引入`Spring`和`SpringMVC`这两个框架，那么它其实就是两个容器，`Spring`是父容器，`SpringMVC`是其子容器，并且在`Spring`父容器中注册的`Bean`对于`SpringMV`C容器中是可见的，而在`SpringMVC`容器中注册的`Bean`对于`Spring`父容器中是不可见的，也就是子容器可以看见父容器中的注册的`Bean`，反之就不行。

```xml
<context:component-scan base-package="com.springmvc.test" />
```

我们可以使用统一的如下注解配置来对`Bean`进行批量注册，而不需要再给每个`Bean`单独使用`xml`的方式进行配置。

从`Spring`提供的参考手册中我们得知该配置的功能是扫描配置的`base-package`包下的所有使用了`@Component`注解的类，并且将它们自动注册到容器中，同时也扫描`@Controller`，`@Service`，`@Respository`这三个注解，因为他们是继承自`@Component`。


```xml
<context:annotation-config/>
```

其实有了上面的配置，这个是可以省略掉的，因为上面的配置会默认打开以下配置。以下配置会默认声明了`@Required`、`@Autowired`、 `@PostConstruct`、`@PersistenceContext`、`@Resource`、`@PreDestroy`等注解。


```xml
<mvc:annotation-driven />
```

这个是`SpringMVC`必须要配置的，因为它声明了`@RequestMapping`、`@RequestBody`、`@ResponseBody`等。并且，该配置默认加载很多的参数绑定方法，比如`json`转换解析器等。



## 7、总结

在实际工程中会包括很多配置，我们按照官方推荐根据不同的业务模块来划分不同容器中注册不同类型的`Bean`：`Spring`父容器负责所有其他非`@Controller`注解的`Bean`的注册，而`SpringMVC`只负责`@Controller`注解的`Bean`的注册，使得他们各负其责、明确边界。



















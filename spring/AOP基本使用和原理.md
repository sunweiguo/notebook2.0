title: AOP基本使用和原理
tag: spring面试
---

除了IOC之外，spring核心的东西就是AOP。主要的目标是实现关注业务逻辑，解耦非业务逻辑，比如比较典型的日志处理。将日志的处理划分出来，在运行时动态地添加到要拦截的接口方法上，对这个方法的执行前后以及发生异常时实现日志的监控。这种动态的功能是非常重要的功能，本文来介绍一下AOP最基本的使用。
<!-- more -->
## 一、什么是aop

AOP（Aspect Oriented Programming），即面向切面编程（也叫面向方面编程，面向方法编程）。其主要作用是，在不修改源代码的情况下给某个或者一组操作添加额外的功能。像日志记录，事务处理，权限控制等功能，都可以用AOP来“优雅”地实现，使这些额外功能和真正的业务逻辑分离开来，软件的结构将更加清晰。AOP是OOP的一个强有力的补充。

## 二、先简单用一下spring aop

##### 首先要导入依赖：spring-aspects


```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-aspects</artifactId>
  <version>4.3.12.RELEASE</version>
</dependency>
```

##### 写一个业务逻辑类：

```java
public class MathCaculator {

    public int div(int i,int j){
        return i/j;
    }

}
```

我们要向在这个业务逻辑方法运行前，运行结束时，方法出现异常时都将日志文件打印。

##### 定义一个日志切面类

切面类里面的方法需要动态感知MathCaculator.div运行到哪里。

通知方法：

- 前置通知(`@Before`)：`logStart()`：在目标方法div运行之前运行
- 后置通知(`@After`)：`logEnd()`:在目标方法div运行结束之后运行
- 返回通知(`@AfterReturning`)：`logReturn()`:在目标方法div运行正常返回之后运行
- 异常通知(`@AfterThrowing`)：`logException()`:在目标方法div出现异常之后运行
- 环绕通知(`@Around`)：动态代理，手动推进目标方法运行


```java
@Aspect
public class LogAspect {

    @Pointcut("execution(public int com.swg.aop.MathCaculator.*(..))")
    public void pointCut(){}

    @Before("pointCut()")
    public void logStart(JoinPoint joinPoint){
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        System.out.println("【@Before】...方法名：{"+methodName+"}...参数列表是->{"+ Arrays.asList(args)+"}");

    }

    @After("pointCut()")
    public void logEnd(JoinPoint joinPoint){
        System.out.println("【@After】...{"+joinPoint.getSignature().getName()+"}结束...");
    }

    @AfterReturning(value = "pointCut()",returning = "result")
    public void logReturn(JoinPoint joinPoint,Object result){
        System.out.println("【@AfterReturning】...{"+joinPoint.getSignature().getName()+"}正常返回，运行结果是{"+result+"}");
    }

    @AfterThrowing(value = "pointCut()",throwing = "exception")
    public void logException(JoinPoint joinPoint,Exception exception){
        System.out.println("【@AfterThrowing】...{"+joinPoint.getSignature().getName()+"}发生异常,异常信息是{"+exception.getMessage()+"}");
    }

}
```


##### 切面类和业务逻辑类都加入到容器中


```java
@EnableAspectJAutoProxy
@Configuration
public class MainConfigOfAop {

    @Bean
    public MathCaculator mathCaculator(){
        return new MathCaculator();
    }

    @Bean
    public LogAspect aspect(){
        return new LogAspect();
    }

}
```
##### 容器启动


```java
@Test
public void test01(){
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfigOfAop.class);
    System.out.println("容器已经启动成功...");
    MathCaculator caculator = applicationContext.getBean(MathCaculator.class);
    caculator.div(6,2);
    applicationContext.close();
}
```

##### 无异常的情况输出


```
容器已经启动成功...
【@Before】...方法名：{div}...参数列表是->{[6, 2]}
div...
【@After】...{div}结束...
【@AfterReturning】...div正常返回，运行结果是{3}
```

##### 有异常的情况输出

```
容器已经启动成功...
【@Before】...方法名：{div}...参数列表是->{[6, 0]}
div...
【@After】...{div}结束...
【@AfterThrowing】...{div}发生异常,异常信息是{/ by zero}
```


主要是有三步：

> 将业务逻辑组件和切面类都加入到容器中，告诉spring哪个是切面类(@Aspect)
>
> 在切面类上的每一个通知方法上标注通知注释，告诉spring合适何地运行(切入点表达式)
>
> 开启基于注解的aop模式(@EnableAspectJAutoProxy)


## 三、基本原理

AOP是具有特定的应用场合的，它只适合那些具有横切逻辑的应用场合，如性能检测、访问控制、事务管理及日志纪录。

Spring AOP使用动态代理技术在运行期织入增强的代码，Spring AOP使用了两种代理机制：**一种是基于JDK的动态代理；另一种是基于CGLib的动态代理**。这两种机制就是AOP最根本的实现原理，面试中把这两者说清楚即可。

##### 3.1 JDK动态代理

Java1.3后，Java提供了动态代理技术，运行开发者在运行期间创建接口的代理实例。JDK动态代理主要涉及`java.lang.reflect`包中的两个类：`Proxy`和`InvocationHandler`。`InvocationHandler`是一个接口，可以通过实现该接口定义横切逻辑，并通过反射机制调用目标类的代码，动态地将横切逻辑和业务逻辑编织在一起。

具体请看笔记[java基础之JDK动态代理](http://fossi.oursnail.cn/2019/02/17/java-basic/java%E5%9F%BA%E7%A1%80%E4%B9%8BJDK%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/)


##### 3.2 CGLib动态代理

CGLib采用底层的字节码技术，可以为一个类创建子类，在子类中采用方法拦截的技术拦截所有父类方法的调用并顺势织入横切逻辑。

需要注意的是，由于CGLib采用动态创建子类的方式生成代理对象，所以不能对目标类中的final或private方法进行代理。


##### 3.3 为什么会有两种代理机制

- JDK创建代理有一个限制，即它只能为接口创建代理实例，虽然面向接口编程是好的编程习惯，但有时候并不是必须的，这是JDK动态代理的局限性。
- 就性能来说，CGLib所创建的动态代理对象的性能比JDK所创建的动态代理对象的性能高差不多10倍，CGLib在创建代理对象时所话费的时间却比JDK动态代理大概多8倍，但是对于singleton的代理对象或者具有实例池的代理，因为无需频繁创建代理对象，所以比较合适采用CGLib动态代理技术，反之则适合采用JDK动态代理技术。

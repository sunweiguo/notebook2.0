title: IOC的用法
tag: spring面试
---

我们已经知道了IOC的基本思想，它用一种倒置的思想帮助我们实现高层建筑需要什么直接引入底层就行，而不需要关心底层的具体实现，因为具体实现已经交给了我们的IOC去实现了。了解了这些之后，光说不练肯定是不行的，下面我们来看看这种依赖倒置到底是如何去用的。首先介绍一下传统的方式，就是spring中经常用的方式。然后再介绍一下springBoot中是什么样子的。其实它们两是差不多的。

<!--more-->

我们知道，要想用`Bean`，那么必然是要先注册进去才行。`Spring` 启动时读取应用程序提供的`Bean`配置信息，并在`Spring`容器中生成一份相应的`Bean`配置注册表，然后根据这张注册表实例化`Bean`，装配好`Bean`之间的依赖关系，为上层应用提供准备就绪的运行环境。

![image](http://bloghello.oursnail.cn/spring2-1.png)

## 一、@Configuration 和 @Bean给容器注册组件

新建一个`maven`工程，引入`spring-context`依赖。

##### 1.1 以往的方式注册一个bean

新建一个实体类`Person`：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class Person {
    private String name;
    private Integer age;
}
```

那么，我们可以在`beans.xml`中注册这个`bean`，给他赋值。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="person" class="com.swg.bean.Person">
        <property name="age" value="10"/>
        <property name="name" value="张三"/>
    </bean>
</beans>
```

那么，我们就可以拿到张三这个人了：

```java
public class MainTest {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans.xml");
        Person person = (Person) applicationContext.getBean("person");
        System.out.println(person);
    }
}//输出：Person(name=张三, age=10)
```

##### 1.2 注解的方式注册bean

- 配置类 = 配置文件
- `@Configuration` 告诉`spring`这是一个配置类
- `@Bean` 给容器注册一个`Bean`，类型为返回值类型，`id`默认是方法名



```java
@Configuration
public class MainConfig {
    @Bean
    public Person person(){
        return new Person("李四",20);
    }
}
```
如何获取这个`bean`呢？

```java
ApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfig.class);
Person person = applicationContext.getBean(Person.class);
System.out.println(person);//Person(name=李四, age=20)
```

我们还可以根据类型来获取这个bean在容器中名字是什么：
```java
String[] names = applicationContext.getBeanNamesForType(Person.class);
for(String name:names){
    System.out.println(name);//person
}
```

上面提到，`id`默认是方法名。如果我们修改`MainConfig`中的`person`这个方法名，果然打印结果也随着这个方法名改变而改变；也可以自己另外指定这个`bean`在容器中的名字：`@Bean("hello")`，那么这个`bean`的名字就变成了`hello`.


##### 1.3 springboot中用这种方式注册bean以及获取

`springboot`那就更简单了。

定义`bean`跟注解的方式是一样的，不再赘述。只是，我们来到它的启动类，直接可以获取到`ApplicationContext`，然后就可以直接获取到`bean`：


```java
@SpringBootApplication
public class SpringbootdemoApplication {

    public static void main(String[] args) {
        ApplicationContext ctx = SpringApplication.run(SpringbootdemoApplication.class, args);
        Person person = (Person) ctx.getBean("person");
        System.out.println(person.getName());
    }

}
```
我们点进`run`里面：

![image](http://bloghello.oursnail.cn/spring2-2.png)

我们这里不深入探讨里面的实现细节，我们这里只要知道可以用`ApplicationContext`来接收即可。

只是，到现在，我们一直在用这个`ApplicationContext`，它到底是何方神圣？


## 二、BeanFactory和ApplicationContext

我们知道，IOC会帮助我们完成对象的创建并将其送到服务对象即完成对象的绑定，即IOC要实现这两件事情：

- 对象的构建
- 对象的绑定


spring提供了两种类型的容器来实现对`bean`的管理，一个是`BeanFactory`,一个是`ApplicationContext`(可以认为是`BeanFactory`的扩展)，这两者是`spring core`中最核心的两个基础接口。下面我们将介绍这两种容器如何实现对对象的管理。


`Spring` 通过一个配置文件描述 `Bean` 及 `Bean` 之间的依赖关系，利用 `Java` 语言的反射功能实例化 `Bean` 并建立 `Bean` 之间的依赖关系。 `Spring` 的 `IoC` 容器在完成这些底层工作的基础上，还提供了 `Bean` 实例缓存、生命周期管理、 `Bean` 实例代理、事件发布、资源装载等高级服务。

`BeanFactory` 是 `Spring` 框架的基础设施，面向 `Spring` 本身；

`ApplicationContext` 面向使用 `Spring`框架的开发者，几乎所有的应用场合我们都直接使用 `ApplicationContext` 而非底层的 `BeanFactory`。

我们先来看看 `BeanFactory` ，再来看看 `ApplicationContext` 。

##### 2.1 BeanFactory

`BeanFactory` 体系架构：

![image](http://bloghello.oursnail.cn/spring2-3.png)


- `BeanDefinitionRegistry`： `Spring` 配置文件中每一个节点元素在 `Spring` 容器里都通过一个 `BeanDefinition` 对象表示，它描述了 `Bean` 的配置信息。而 `BeanDefinitionRegistry` 接口提供了向容器手工注册 `BeanDefinition` 对象的方法。
- `BeanFactory` 接口位于类结构树的顶端 ，它最主要的方法就是 `getBean(String beanName)`，该方法从容器中返回特定名称的 `Bean`，`BeanFactory` 的功能通过其他的接口得到不断扩展.
- `ListableBeanFactory`：该接口定义了访问容器中 `Bean` 基本信息的若干方法，如查看`Bean` 的个数、获取某一类型 `Bean` 的配置名、查看容器中是否包括某一 `Bean` 等方法；
- `HierarchicalBeanFactory`：父子级联 `IoC `容器的接口，子容器可以通过接口方法访问父容器； 通过 `HierarchicalBeanFactory` 接口， `Spring` 的 `IoC` 容器可以建立父子层级关联的容器体系，子容器可以访问父容器中的 `Bean`，但父容器不能访问子容器的 `Bean`。`Spring` 使用父子容器实现了很多功能，比如在 `Spring MVC` 中，展现层 `Bean` 位于一个子容器中，而业务层和持久层的 `Bean` 位于父容器中。这样，展现层 `Bean` 就可以引用业务层和持久层的 `Bean`，而业务层和持久层的 `Bean` 则看不到展现层的 `Bean`。
- `ConfigurableBeanFactory`：是一个重要的接口，增强了 `IoC` 容器的可定制性，它定义了设置类装载器、属性编辑器、容器初始化后置处理器等方法；
- `AutowireCapableBeanFactory`：定义了将容器中的 `Bean `按某种规则（如按名字匹配、按类型匹配等）进行自动装配的方法；
- `SingletonBeanRegistry`：定义了允许在运行期间向容器注册单实例 `Bean` 的方法；

⭐⭐⭐其中，我们最关心的是`BeanDefinition`、`BeanDefinitionRegistry`、
`BeanFactory`。注意，`BeanDefinition`,它完成了`Bean`的生成过程。`BeanDefinitionRegistry`是将定义好的`bean`，注册到容器中。`BeanFactory` 是一个`bean`工厂类，从容器中可以取到任意定义过的`bean`。

`Bean`的生成大致可以分为两个阶段：容器启动阶段和`bean`实例化阶段：

![image](http://bloghello.oursnail.cn/spring2-6.png)


它是面向`spring`管理`bean`的最核心的一个接口，但是作为使用者，我们往往更关心的是`ApplicationContext`.


##### 2.2 ApplicationContext

`ApplicationContext` 由 `BeanFactory` 派生而来，提供了更多面向实际应用的功能。

在`BeanFactory` 中，很多功能需要以编程的方式实现，而在 `ApplicationContext` 中则可以通过配置的方式实现。

![image](http://bloghello.oursnail.cn/spring2-4.png)

`ApplicationContext` 继承了 `HierarchicalBeanFactory` 和 `ListableBeanFactory` 接口，在此基础上，还通过多个其他的接口扩展了 `BeanFactory` 的功能：

- `ClassPathXmlApplicationContext`：默认从类路径加载配置文件
- `FileSystemXmlApplicationContext`：默认从文件系统中装载配置文件
- `ApplicationEventPublisher`：让容器拥有发布应用上下文事件的功能，包括容器启动事件、关闭事件等。实现了 `ApplicationListener` 事件监听接口的 `Bean` 可以接收到容器事件 ， 并对事件进行响应处理 。 在 `ApplicationContext` 抽象实现类`AbstractApplicationContext` 中，我们可以发现存在一个 `ApplicationEventMulticaster`，它负责保存所有监听器，以便在容器产生上下文事件时通知这些事件监听者。
- `MessageSource`：为应用提供 i18n 国际化消息访问的功能；
- `ResourcePatternResolver` ： 加载资源文件
- `LifeCycle`：该接口是 `Spring 2.0` 加入的，该接口提供了 `start()`和 `stop()`两个方法，主要用于控制异步处理过程。在具体使用时，该接口同时被 `ApplicationContext` 实现及具体 `Bean` 实现， `ApplicationContext` 会将 `start/stop` 的信息传递给容器中所有实现了该接口的 `Bean`，以达到管理和控制 `JMX`、任务调度等目的。
- `ConfigurableApplicationContext` 扩展于 `ApplicationContext`，它新增加了两个主要的方法： `refresh()`和 `close()`，让 `ApplicationContext` 具有启动、刷新和关闭应用上下文的能力。在应用上下文关闭的情况下调用 `refresh()` 即可启动应用上下文，在已经启动的状态下，调用 `refresh()`则清除缓存并重新装载配置信息，而调用`close()`则可关闭应用上下文。这些接口方法为容器的控制管理带来了便利，但作为开发者，我们并不需要过多关心这些方法。


##### 2.3 两者的区别

- `BeanFactory`是`Spring`框架的基础设施，面向`Spring`
- `ApplicationContext`面向使用`Spring`框架的开发者

`BeanFactory`可以理解为汽车的发动机，`ApplicationContext`可以理解为比较完整的一辆汽车。

## 三、@ComponentScan自动扫描组件以及扫描规则

每次都用`@Configure`和`@Bean`着实太麻烦了，有没有什么办法可以自动地装载为`Bean`呢？答案就是这个自动扫描的注解。下面来看看是如何使用的。

配置文件中配置包扫描时这样配置的：

```
<!--包扫描，只要标注了@Controller，@Service，@Repository，@Component，就会被自动扫描到加入到容器中-->
<context:component-scan base-package="com.swg"/>
```

现在用注解来实现这个功能，只需要加上注解`@ComponentScan`即可：


```java
@ComponentScan(value = "com.swg")//java8可以写多个@ComponentScan
//java8以前虽然不能写多个，但是也可以实现这个功能，用@ComponentScans配置即可
```

表示自动扫描`com.swg`包路径下以及子目录下所有的`Bean`，装载进容器中。


上面的扫描路径是扫描所有的，有的时候我们需要排除掉一些扫描路径或者只扫描某个路径，如何做到呢？

用`excludeFilters`来排除，里面可以指定排除规则，这里是按照`ANNOTATION`来排除，排除掉所有`@Controller`注解的类。`classes`也是个数组，可以排除很多。
```java
@ComponentScan(value = "com.swg",excludeFilters = {
        @ComponentScan.Filter(type = FilterType.ANNOTATION,classes = Controller.class)
})
```
那么效果就是`controller`没有了，但是`service`和`dao`都在。

那如果我想只包含`controller`呢？


```java
@ComponentScan(value = "com.swg", includeFilters = {
        @ComponentScan.Filter(type = FilterType.ANNOTATION,classes = Controller.class)
},useDefaultFilters = false)
```

注意要`useDefaultFilters = false`，因为默认为`true`，就是扫描所有，不设置为`false`无效。

关于`springboot`这里强调一下， 启动方法上的注解中已经默认有了扫描的注解，默认扫描的范围是这个启动类所在的路径及其子路径。
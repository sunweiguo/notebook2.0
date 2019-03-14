title: Spring Bean
tag: spring面试
---

一提到Spring的IOC，那么里面的Bean基本就会被问到，我们也知道，Spring的任务就是对这些bean进行管理和装配，所以bean就是spring IOC处理的对象，如此关键的对象，我们需要了解它核心的两点：作用域和生命周期。

<!--more-->

## 一、Spring Bean的作用域

- `singleton`：Spring默认作用域，容器里拥有唯一的Bean实例
- `prototype`：针对每个getBean请求，容器都会创建一个bean实例
- `request`：会为每个HTTP请求创建一个Bean实例
- `session`：会为每个session创建一个Bean实例
- `globalSession`：会为每个全局Http Session创建一个Bean实例，该作用域仅对Portlet有效

对于这个问题面试中我也被问过，即spring中bean默认作用域，如何设置为多例。这个问题我在[注解--组件注册](http://fourcolor.oursnail.cn/2019/03/03/spring/%E6%B3%A8%E8%A7%A3--%E7%BB%84%E4%BB%B6%E6%B3%A8%E5%86%8C/)这篇文章中会详细谈论。

## 二、Bean的生命周期

`beanDefinition`（容器启动阶段）只完成`bean`的定义，并未完成初始化。初始是通过`beanFactory`的`getBean()`时才进行的。

对于普通的Java对象，当new的时候创建对象，当它没有任何引用的时候被垃圾回收机制回收。而由Spring IoC容器托管的对象，它们的生命周期完全由容器控制。Spring中每个Bean的生命周期如下：

![image](http://bloghello.oursnail.cn/spring4-1.png)


##### 2.1 实例化Bean

对于`BeanFactory`容器，当客户向容器请求一个尚未初始化的`bean`时，或初始化`bean`的时候需要注入另一个尚未初始化的依赖时，容器就会调用`createBean`进行实例化。 

容器通过获取`BeanDefinition`对象中的信息进行实例化。并且这一步仅仅是简单的实例化，并未进行依赖注入。 

实例化对象被包装在`BeanWrapper`对象中，`BeanWrapper`提供了设置对象属性的接口，从而避免了使用反射机制设置属性。


##### 2.2 设置对象属性（依赖注入）

实例化后的对象被封装在`BeanWrapper`对象中，并且此时对象仍然是一个原生的状态，并没有进行依赖注入。 

紧接着，`Spring`根据`BeanDefinition`中的信息进行依赖注入。

并且通过`BeanWrapper`提供的设置属性的接口完成依赖注入。

##### 2.3 注入Aware接口

紧接着，Spring会检测该对象是否实现了`xxxAware`接口，并将相关的`xxxAware`实例注入给bean。


##### 2.4 BeanPostProcessor

当经过上述几个步骤后，bean对象已经被正确构造，但如果你想要对象被使用前再进行一些自定义的处理，就可以通过`BeanPostProcessor`接口实现。 

该接口提供了两个函数：

- `postProcessBeforeInitialzation( Object bean, String beanName )` 
    - 当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。
    - 这个函数会先于`InitialzationBean`执行，因此称为前置处理。 
    - 所有Aware接口的注入就是在这一步完成的。
- `postProcessAfterInitialzation( Object bean, String beanName )` 
    - 当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。
    - 这个函数会在`InitialzationBean`完成后执行，因此称为后置处理。



##### 2.5 InitializingBean与init-method

当`BeanPostProcessor`的前置处理完成后就会进入本阶段。 

`InitializingBean`接口只有一个函数：

- `afterPropertiesSet()`

这一阶段也可以在bean正式构造完成前增加我们自定义的逻辑，但它与前置处理不同，由于该函数并不会把当前bean对象传进来，因此在这一步没办法处理对象本身，只能增加一些额外的逻辑。 若要使用它，我们需要让bean实现该接口，并把要增加的逻辑写在该函数中。然后Spring会在前置处理完成后检测当前bean是否实现了该接口，并执行`afterPropertiesSet`函数。


##### 2.6 DisposableBean和destroy-method


和`init-method`一样，通过给`destroy-method`指定函数，就可以在bean销毁前执行指定的逻辑。


对于上面的过程只能理解并且记忆，还是很容易被问到的，是spring的一个高频考点。或许你对这些所说的方法一脸懵逼，对于生命周期这一块，我对里面涉及的所有初始化以及销毁方法进行了汇总，详情见文章:[注解-生命周期](http://fourcolor.oursnail.cn/2019/03/03/spring/%E6%B3%A8%E8%A7%A3-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/)
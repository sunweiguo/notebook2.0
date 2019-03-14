title: SpringBoot使用logback实现日志按天滚动
tag: miscellany
---

日志是任何一个系统都必备的东西，日志的重要程度丝毫不亚于代码。而springboot中经常使用的是logback，那么今天我们就来学习一下在springboot下如何配置logback日志。理解了这里的配置，对于任何的日志都是一样的。
<!-- more -->

## 需求
- 日志按天滚动分割
- `info`和`error`日志输出到不同文件

## 为什么使用Logback

- `Logback`是`Log4j`的升级版，作者为同一个人，作者不想再去改`Log4j`，所以写了`Logback`
- 使用日志框架的最佳实践是选择一款日志门面+一款日志实现，这里选择`Slf4j`+`Logback`,`Slf4j`作者也是`Logback`的作者
- `SpringBoot`从1.4版本开始，内置的日志框架就是`Logback`


## Logback在SpringBoot中配置方式一

可以直接在`applicatin.properties`或者`application.yml`中配置

以在`application.yml`中配置为例


```yml
logging:
  pattern:
    console: "%d - %msg%n"
  file: /var/log/tomcat/sell.log
  level:
    com.imooc.LoggerTest: debug
```

可以发现，这种配置方式简单，但能实现的功能也很局限，只能
- 定制输出格式
- 输出文件的路径
- 指定某个包下的日志级别

如果需要完成我们的需求，这就得用第二种配置了

## Logback在SpringBoot中配置方式二

在`resource`目录下新建`logback-spring.xml`, 内容如下
<?xml version="1.0" encoding="UTF-8" ?>


```xml
<configuration>
    <!--打印到控制台的格式-->
    <appender name="consoleLog" class="ch.qos.logback.core.ConsoleAppender">
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>
                %d - %msg%n
            </pattern>
        </layout>
    </appender>
    
    <!--除了error级别的日志文件保存格式以及滚动策略-->
    <appender name="fileInfoLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--过滤器，将error级别过滤掉-->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>DENY</onMatch>
            <onMismatch>ACCEPT</onMismatch>
        </filter>
        <encoder>
            <pattern>
                %msg%n
            </pattern>
        </encoder>
        <!--滚动策略-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--路径-->
            <fileNamePattern>/var/log/tomcat/sell/info.%d.log</fileNamePattern>
        </rollingPolicy>
    </appender>

    <!--error级别日志文件保存格式以及滚动策略-->
    <appender name="fileErrorLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--只让error级别的日志进来-->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
        <encoder>
            <pattern>
                %msg%n
            </pattern>
        </encoder>
        <!--滚动策略-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--路径-->
            <fileNamePattern>/var/log/tomcat/sell/error.%d.log</fileNamePattern>
        </rollingPolicy>
    </appender>

    <root level="info">
        <appender-ref ref="consoleLog" />
        <appender-ref ref="fileInfoLog" />
        <appender-ref ref="fileErrorLog" />
    </root>

</configuration>
```

每一个`appender`你可以理解为一个日志处理策略。

第一个`appender`的`name="consoleLog"`, 

名字是自己随意取的，取这个名字，表示这个策略用于控制台的日志。

我们重点看第二个和第三个`appender`,因为要把`info`和`error`日志输入到不同文件，所以我们分别建了两个`appender`。

`rollingPolicy`是滚动策略，这里我们设置按时间滚动

`filter`是日志的过滤方式，我们在`fileInfoLog`里做了如下过滤

```xml
<level>ERROR</level>
<onMatch>DENY</onMatch>
<onMismatch>ACCEPT</onMismatch>
```

上述代码翻译之后：拦截`ERROR`级别的日志。如果匹配到了，则禁用处理。如果不匹配，则接受，开始处理日志。

那有的同学要问了，不能这样写吗


```xml
<level>INFO</level>
```


这样不是只拦截`INFO`日志了吗？

不对！

这就得说一下日志级别了

`DEBUG` ->`INFO` -> `WARN` ->`ERROR`

如果你设置的日志级别是`INFO`，那么是会拦截`ERROR`日志的哦。也就是说，如果直接写`info`，那么大于等于`info`级别的日志都会写进去，违背了我们的需求。

整理自：

- http://www.imooc.com/article/19005
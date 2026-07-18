### ，一、概述

BAT目前整合了Hickwall、BAT、Clog、ES kibana等多个监控系统

在BAT我们可以查日志、设置数据可视化监控面板、查看应用/机器的健康状态



### 二、名词解释

Metrics

度量：指对系统中某类信息的统计聚合

Logs：

记录离散事件，目前主要用clickhouse存日志

Traces：

链路追踪

单体系统时代：追踪的范畴局限于栈追踪调试程序时，在IDE打断点 处理异常调用Exception::printStackTrace输出堆栈信息

但是现在微服务时代，追踪不限于调用栈，一个外部请求需要内部若干服务的联动响应，这时候完整的调用轨迹将跨越多个服务，同时包括服务局网络传输信息与各个服务内部的调用堆栈信息，因此分布式系统中的追踪被称为全链路追踪

可以理解为一个请求上下文在多个分布式微服务节点的完整执行链路

Logview：

也称为TraceView，将每次URL、Service的请求内部执行情况都封装为一个完整的消息树



### 三、日志检索

BAT将各种日志的入口统一，可以在一个界面上搜索各种日志，且将clog的日志存储迁移到ck





### 四、CAT接入









### 五、日志接入

Java日志生态分为两层：门面facade和实现impl

slf4j日志门面 只是一套接口 代码只依赖slf4j接口，底层换Logback还是Log4j2，业务代码一行不改，类比JDBC你写Connection/PreparedStatement，底层可以是mysql或postgresql驱动

实现有log4j和logback 

```
  <root level="INFO">                                                                                                                                                       
      <appender-ref ref="TripLog"/>                                                                                                                                         
      <appender-ref ref="Console"/>                                                                                                                                         
  </root> 
```

level=info：info及以上warn、error会输出，debug被过滤掉

一条日志会同时发送给所有appender

appender是日志输出目的地



















**客户端配置**

应用端使用原生slf4j API，根据不同的日志实现配置对应的Appender使用

POM配置 

<dependency>
    <groupId>com.ctrip.framework</groupId>
    <artifactId>triplog-client</artifactId>
</dependency>

使用最新版本的Framework BOM，slf4j可以理解为只是一个API规范，具体功能由具体的实现类完成，可以选用log4j2或者Logback进行配置



使用标准slf4j API记录的日志将转记录到CLOG中

private static final Logger log = LoggerFactory.getLogger(Demo.class);

public void demo() {
  log.info("Hello World");
}

如果希望记录CLOG Tags可以使用slf4j的Marker API 需要使用TagMarker实现类

同时如果希望对应的部分日志落入到Clickhouse，可以指定要输出的scenario，数据将转发到ck中落地

一般如果日志具有业务含义，可以格式化，有根据某些字段做报表的需求，使用Clickhouse，如果是开发阶段使用的纯文本调试日志或是异常报错日志，使用CLOG



这里注意一点是，如果想指定日志的title的话，也是放在Tag里的，指定Key为"tagA"即可





### 六、日志查询

现在框架这边对LOG做了收口，按道理所有的日志都能在BAT里查到

查询条件有：

标题title（如果不指定 默认是NA）

服务器IP（比如想某台机器的日志）

来源（想看哪个类的log 记得是全类名）

Tags（K-V格式 每条log会有自带的一些信息）

Message（正文 只支持前128字节搜索）

region（SHA FRA SGP）

时间



### 七、性能指标


































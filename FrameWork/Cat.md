### 1、与clog的区别

Clog呢定位是一个日志系统，支持日志的多维度查询

CAT是一个基于分布式Trace的实时监控，我们可以根据CAT跟踪每一个分布式调用的详细信息，监控服务性能、异常，和对自定义事件的告警，定位来说还是一个监控系统



并且其实CAT还是倾向于指标链路事件监控，不太适用大量业务日志的场景









### 2、Cat的基石--两个抽象模型

**Trace**

用于记录一次完整的请求过程，由多个Transaction和Event组成



**Transaction**

trace中的一个事务，支持嵌套

程序中的一个步骤，有开始、结束，消耗一段时间

通过Type，Name两级分类



**Event**

Trace中的一个事件

程序中的一个时间点，即一个时刻无耗时，也是Type，Name两级分类



**Heartbeat**

一分钟采集一次



**LogView**

CAT监控系统将每次URL、service请求内部执行情况封装为一个完整的logview



### 3、调用示例

Cat.newTransaction(type,name)表示过程的开始

如果在业务代码里遇到了Exception，我们需要将Transaction的status设置为exception，有必要的话可以通过Cat.logError把错误记录下来  

那么cat就是通过这样的埋点将事务和属性发送到服务端

  http://docs.fx.ctripcorp.com/docs/bat/tutorial/cat#%E5%BC%82%E6%AD%A5%E5%9F%8B%E7%82%B9%E7%A4%BA%E4%BE%8B







### 4、Cat性能指标

SOA报表：可以看某一个APP调用其他SOA服务这样一个情况，以及APP提供了哪些服务，被哪些应用调用

Thread Dump：可以看线程运行情况，CPU占用 

Transaction：即我们自己的服务端埋点

Event：即时刻埋点



###  5、Cat和Clog联动

即比如你在Cat的Transaction里写了一条clog，那么你可以在clog里跳转到这条clog的上下文



git@git.dev.sh.ctripcorp.com:mitbackendexam11/backendmitexam.git



### 6、异步埋点




























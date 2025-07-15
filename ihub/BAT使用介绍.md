## 1、BAT All In One Overview

之前的日常排障路径是：收到一个告警--看监控大盘--查看详细的调用链路数据---查日志信息

可能会发现应用方面的异常，也可能是某些实例的异常，这时候就需要看系统指标

在以上阶段可能会接触到bigeyes告警 hickwall监控大盘 cat/bat1.0调用链 clog/es kibana日志

基本上有4-5个系统 于是BAT把这些整合到一起

整合后收到告警直接进入BAT的指标页，做了trace和log的联动

然后再trace的调用链也集成了某些实例的指标



## 2、APM应用/服务性能监控

监控术语：（不仅是cat的术语）

cat是美团开源的一个系统 带有一些概念在 我们保留下来了

- Problem 项目在运行过程中出现的问题（错误 长访问等）
- Transaction 适合记录跨越系统边界的程序访问行为，比如远程调用，数据库调用，较长的业务逻辑
- Event 记录一件事发生的次数 例如系统异常等 开销比Transaction小
- Heartbeat 程序内定期产生的统计信息 如CPU% MEM% 连接池 系统负载

前三个记为Metric：记录应用/业务指标

这四个加起来为logview消息树，即BAT将每次URL、Service的请求内部执行情况封装为一个完整的消息树logview

在携程内部，logview、trace、采样点称为分布式追踪，记录在微服务间的调用情况

常用logview的页面去排查全链路的情况



App详情监控各模块预览

1. summary 应用概览 多种指标组合
2. Problem 异常 长耗时 实例心跳 失败transaction
3. Transaction 服务指标 看调用和被调用的情况即SOA2Sevice SOA2Client
4. Event 记录事件次数 只有次数没有耗时
5. Heartbeat 系统指标
6. Thread 线程阻塞 死锁 看JVM一些堆栈信息
7. Timeout超时
8. SOA详细报表 看调用与被调用的情况



多维度分析 比如我观察到某个时间点异常很多 我可以直接点击这个点 看到对应的信息



全局报表与数据开发：

1、中间件版本报表 这是框架埋的点 想看bu有多少升级到了多少版本

2、Transaction报表 查看统一BU的SOA访问情况

BAT-API 可以获取到这些监控和日志的数据



## 3、Log模块

为了更灵活的TTL，Clog一键迁移到了CK，这使得日志数据变得结构化，这使得你可以基于某一列去加索引

例如对tag进行提取加速查询



## 4、Dashboard



## 5、实操

这个讲得非常好 建议反复多观看思考

http://ihub.ctripcorp.com/front/course/detail/18577?itemid=40529



注意 在SOA详细报表里的数据是采样1%的结果，其目的在于分析当前服务被哪些appid调用，调用的比例是多少

如果需要查看具体的调用数量可以在Transaction里看
























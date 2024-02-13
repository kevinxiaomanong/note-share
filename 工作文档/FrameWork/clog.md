 clog是基于java开发的实时日志系统，支持.net/java客户端（以前主要是net后续迁到了java）

java客户端支持SLF4J规范API

clog支持实时和历史日志存储和查询 



**一条日志包含哪些**

1. message 即你的日志正文
2. source 来源，指的是日志的记录者，我们在java中一般记录全类名
3. title 标题
4. tags标签 由一组k-v组成  clog会默认记录一些线程id等信息，我们也可以加一些自己的tags



_

**架构**

![image-20231122123653252](D:\Users\haoxiang_zhang\AppData\Roaming\Typora\typora-user-images\image-20231122123653252.png)

 client调用过程

![image-20231122123838457](D:\Users\haoxiang_zhang\AppData\Roaming\Typora\typora-user-images\image-20231122123838457.png)

**基本流程**

日志发送：

- 客户端会通过读取FrameworkFoundation的环境，然后根据环境获取日志收集服务的IP列表
- 用户记录日志，日志暂时放入一个内存队列
- 后台异步线程从内存队列中取出日志进行批量打包压缩并发送到日志收集服务器



**使用场景**

- 记录错误/异常，为了排查异常
- 记录运行日志，来分析程序运行状态
- 对异常日志进行监控告警



**注意点**

- 压缩的日志暂时存放在内存queue内，最多占用5%最大内存（Jvm最大内存，非机器最大内存）
- 对于产生大量日志的客户端，在内存queue满的情况下会发送日志丢失












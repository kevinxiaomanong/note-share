### 一、Mysql访问方式

携程里大多走的是Dal Cluster和Titan，标准dal

通过qconfig拉到DataSource Config，拉到动态IP数据源

这样 当发生主从的时候 会走qconfig拉取 更新

而如果走的是DataSource 例如python、C++这样的应用 走IP直连 那就需要更新DNS缓存



### 二、复制架构

![image-20240201143546885](D:\工作文档\ihub\assets\image-20240201143546885.png)

当IDC挂了 会将流量切换到另一个IDC 相当于是做灾备

而如果是master挂了 那会在同一个IDC内找副本

为什么这样设计呢？

因为同一个IDC内的节点，走的是内网，副本一定比另一个IDC的节点要新





![image-20240201143753263](D:\工作文档\ihub\assets\image-20240201143753263.png)

而如果是这种三机房DR部署，那会优先切IDC 而不是阿里云

因为IDC走骨干网 阿里云是专线 这样一般IDC更新



**复制方式--半同步复制**

mysql两阶段提交问题

client做一次commit的时候，首先会写redo log，并把commit的事务置为prepare状态，之后对binlog写入

binlog写入完后会向slave节点发送Ack请求，即确认binlog是否传到了slave，且slave是否已经生成了redolog中继日志

Ack回包到master后，才会在引擎层写入成功，返回commit ok



这样可以保证一定有一个slave节点是最新的数据，因为在事务提交前是保证写到下游的，前提是不超过半同步的阈值



假设在Ack阶段crash掉了，在salve节点拿到的redo log一定是最新的 保证数据不丢失 在切换过程这是很重要的机制 大大降低数据丢失概率



**CMHA**

携程基于MHA封装了一套，里面有Mysql sentinel哨兵（基于Redis哨兵）和MHA(切换、一致性)

找最新的slave，并尽可能把master的日志补过去，然后发起切换



**切换流程**

CMHA发现master节点挂掉了，发起切换

会通知DNA Server、Dal Cluster、Qconfig



但是如果master没有死透 还有应用层访问，然后新的应用也在写，双写导致出现脑裂问题

这种会优先把老master杀掉，然后从binlog比对差异数据 反馈给业务侧 让业务侧选择需要哪些数据

 

### 三、mysql工具

主要有dbtools

还有面向开发的idb，这个推荐好用



里面有个slow log的功能 帮助排查慢查询sql

此外idb还主推一个数据库诊断功能



### 四、mysql使用规范

富文本不要存mysql，因为这种字段很长，一个是网络带宽问题，二是mysql不适合存放这个东西

建议用分布式文件系统存

日志还是放ES、CK



外键对性能损耗严重 不允许用

不建议使用TEXT类型 容易把IO打爆 网络带宽 不要当文档数据库

字符集utf8mb4



字段声明NOT NULL 且必须显示指定默认值

NULL这个很坑 count的时候查不到



精度用demical



前缀索引 



组合索引 把区分度高的放前面



如果有出海需求增加userdata_location UDL字段



### 五、索引规范

原则一：避免全表扫描

不要用前导%查询

原则二：避免排序

原则三：避免回表



禁止使用子查询  -- mysql对子查询的支持力度较低

禁止使用select * 要指定字段 



不允许出现多于一次的join，且需要join的字段字段类型要绝对一致（不可以有隐式的类型转换），多表关联时确保关联字段有索引

不要有大事务，这会导致binlog很大 顺序写会锁住db 导致db一个不可用的状态








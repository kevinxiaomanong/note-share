### 一、概述

DAL是一个数据库访问框架，通过DAL Cluster提供数据库访问

DAL Cluster可以简单理解为包含n个数据库完整连接信息的配置文件，通过这种配置文件，可以获取到数据库的连接信息，从而为应用创建数据源



### 二、使用指南

首先你要保证你的应用有访问数据库的权限，这个在pass平台区申请

Dal是依赖于Framework Foundation的环境配置，所以要设置好相应的server.properties文件

其次就是maven的依赖引入，我们还是建议使用Dal作为ORM框架



**重要的配置文件**

1、Dal.config 

是DAL的核心配置文件，包含逻辑数据库和物理数据库的映射，以及DAL客户端的配置项

这个可以放在本地的resource下，当然也可以放qconfig上 本地config优先级是更高的

最重要的还是把你的db集群名称配上



2、datasource.properties

这个是主要是一些数据库连接池相关属性，如果你不需要自己定义连接池属性，那其实不用配置这个文件，因为不配置的情况下会自动拉取全局默认配置

在qconfig上有个继承文件，我们可以继承下来然后改一些自己的配置



### 三、使用流程

- 架构、DB评审
- 申请数据库
- 使用codegen生成特定数据库访问代码
- 将代码放入项目
- 运行&测试





### 四、常用API

**参数**

1、DalHints

会保留Dal处理的上下文或中间数据，每次调用DAL的API时请重新生成DalHints的实例



2、sql

这里可以是一个标准sql，也可以是where条件，DAL根据开头是否命中关键词来判断是否为完整sql，例如select、delete等等

支持？占位符，参数顺序要和？顺序一一对应



3、SQLArg

作用：

- 对于敏感参数需要先使用SQLArg.value封装，然后调用senstive()
- 可以用SQLArg.list传参,做到动态参数



4、SQLResult

- type 返回值映射类型
- mapper 自定义结果集映射
- keyHolder 保存主键
- sorter 排序
- page 分页参数



5、Object... args

与sql中参数一一对应



**单表API**

通过DalOperationsFactory获取DalTableOperations



这里关于insert有三种：

1. insert 多事务、每个事务插入一条记录
2. combinedInsert 以一条sql的形式插入
3. batchInsert 在一个事务中一条接一条插入



























### 五、运行流程

**初始化**

- 获取、解析Dal.config配置文件
- 根据配置的Titan Key，从Titan服务器获取数据库连接串
- 根据连接串，创建连接池



**跑SQL**

DAL将请求路由到对应DB，从连接池获取一个连接，响应后将连接归还给连接池，并将响应结果转化为特定对象返回



### 六、数据库集群架构方案

![image-20231204115204931](D:\Users\haoxiang_zhang\AppData\Roaming\Typora\typora-user-images\image-20231204115204931.png)

我们主流采用读写分离+分片的方式来进行存储，这和Redis集群其实大同小异

既可以保证几十亿的数据正常承载，MHA来做故障转移



这个分片算法其实大致上可以分为两类：

- 范围 即根据主键id范围划分
- hash 主键id取模



**引申-一致性Hash算法**

分布式系统的负载均衡算法

我们先分析一下Hash算法，有个致命的问题是如果节点数量发生了变化，也就是在对系统做扩容或缩容的时候，必须迁移改变了映射关系的数据，最坏的情况下所有数据都需要迁移，所以我们需要一个新的算法，来避免分布式系统在扩容或缩容时发生过多的数据迁移



这个新的算法即一致性哈希算法

1、普通的hash算法是对节点数量取余，但一致性hash会对2^32进行取模运算，是一个固定的值

2、一致性hash会进行两步哈希 第一步先对存储节点做哈希索引（例如根据节点IP进行哈希）第二步对数据进行hash，因此它是将存储节点和数据都映射到一个首尾相连的哈希环上，数据哈希结果值往顺时针方向找到的第一个节点即存储改数据的节点



这样一来，当节点增删时仅仅是该节点在哈希环上顺时针相邻的后继节点需要变更



单一致性hash算法并不能保证节点能够在哈希环上分布均匀，这可能会导致有大量请求集中到一个节点上，这会容易引起雪崩的连锁反应



那么如何让节点分布均匀呢？

一种思路是有大量的节点，节点越多哈希环上的节点分布就越均匀，但实际中我们没有那么多节点于是就加入虚拟节点，即对一个真实节点做多个副本

具体实现：我们将虚拟节点映射到哈希环上，并将虚拟节点映射到实际节点，所以这里有两层映射关系



并且虚拟节点还能提高系统的稳定性，当有节点变化时会有不同的节点共同分担系统的变化因此稳定性更高



**负载均衡策略**

其实轮询这类策略只能使用于每个节点数据都相同的场景，访问任意节点都能请求到数据，但不适用分布式系统，因为分布式系统意味着数据水平切分到了不同的节点





### DAL接入思路

1、本地启动环境配置 标识当前运行环境 即本地的server.properties文件 Java应用的环境获取（http://conf.ctripcorp.com/pages/viewpage.action?pageId=74102518#FrameworkFoundation:%E8%8E%B7%E5%8F%96AppId,%E5%BD%93%E5%89%8D%E7%8E%AF%E5%A2%83,%E6%95%B0%E6%8D%AE%E4%B8%AD%E5%BF%83%E7%AD%89%E5%9F%BA%E7%A1%80%E9%85%8D%E7%BD%AE%E7%9A%84.NET/JavaAPI-%E8%8E%B7%E5%8F%96%E7%8E%AF%E5%A2%83）dal这里是借用了Framework Foundation的环境设置



2、dal.config 即对应dal cluster的配置

你可以把它放在项目的resource根目录下 或是放在qconfig上 而且本地dal是优先的

![image-20240624103540144](D:\note\工作文档\FrameWork\assets\image-20240624103540144.png)

注意 dalcluster需要放置在 `databaseSets`标签下，每一个dalcluster都使用`cluster`标签包装，`cluster`标签的`name`属性设置为你的dalcluster名称。因此你的应用用了几个集群 就要配置几个



当你需要debug Dal.config时 可以上BAT 选择机器发布时间附近的时间段 然后输入dal过滤



3、Dal cluster介绍

dal cluster是数据库集群信息配置系统，实现客户端逻辑库配置中心化

简单来说 就是配置的信息让数据库端配置 客户端配置繁琐统一出错 例如分片 主从 实例等等

这些配置信息都放在Dal.config里 然后Dal会从服务端自动拉取且动态切换





4、表明应用

resource/META-INF/app.properties 



5、datasource.properties

数据库连接池属性 用户无需自定义 默认会用dal的全局配置

伏羲项目里 继承了datasource.properties 自定义了serverWaitTimeout 方便debug代码



### 代码生成器应用

当我们创建好数据库的表之后 哪些pojo和dao的代码其实可以自动生成 即代码生成器





## Ihub---陈呈

后端系统从分层角度上看分为表示层、业务逻辑层、数据访问层、数据库四层

Dal属于数据访问层框架，业界的架构主要分两类--客户端模式和代理模式



1、客户端模式

应用持有多个数据源，每个数据源对应一个数据库，应用和数据库直连

2、代理模式

应用持有一个数据源，并不直接与db相连，与中间代理层相连，由中间代理层转发到不同数据库



DAL属于客户端模式



**总体架构**

<img src="D:\note\工作文档\FrameWork\assets\image-20240703182609078.png" alt="image-20240703182609078" style="zoom:50%;" />

DAL Client以jar包的形式寄存在应用内

数据库连接串是指 数据库地址 用户密码等 为了安全起见 不允许用户自己在应用里配置 DAL托管了一套保存的方法 都存储在qconfig上面

DAL给携程内部各日志监控系统都有日志输出

代码生成器是帮助自动生成pojo和dao的辅助开发工具



![image-20240703183016670](D:\note\工作文档\FrameWork\assets\image-20240703183016670.png)

其实数据访问层框架核心是ORM和数据源层 两个不同层最终打上不同jar包

ORM主要是和代码打交道 Java对象---数据库语言  

数据源就是和数据库打交道



这个反应在接入的场景里，即你是选用DAL ORM还是Mybatis+DAL DataSource

建议是DAL ORM 能自动支持分库分表



接入：

1、DBA申请数据库

2、到DAL Cluster看名称

3、代码生成器 entity dao dal.config



目前dal支持两大类dao的操作--DalTableOperations和DalDatabaseOperations

前者通过pojo来和数据库交互 对标Hibernate 

后者通过sql来交互 对标Mybatis



本地开发 好像是所有环境都在本地 我们这边好像不太适用 这种方式就需要本地自己起一个mysql实例

这对于我们这边单独开发项目来说不必要



**监控日志**

bat transaction

sql执行次数和平均耗时


















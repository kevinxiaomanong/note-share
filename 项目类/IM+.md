# 背景&准备

http://conf.ctripcorp.com/pages/viewpage.action?pageId=2694940839



### 项目背景

- 管理层查阅全BU重要服务指标数据
- 服务会给各BU数据表现复盘，目前线下人工收集

现状是IM后台嵌入Nova报表，已接入机票酒店火车票IBU的部分服务指标数据，但是交互形式体验较差且有部分BU及指标缺失，因此重新规划前后端开发



项目会覆盖机酒火旅+IBU核心BU 以及19各指标数据，从19年1.1至今



首先理解IM+是客服用的软件，IM+有人工服务也接了智能服务，主要就是看各个BU的业务指标

所以说白了现在是在对IM全链路看板



### 项目关键节点

26号交互出来



然后我们27号可以确认一下交互，明确开发内容、范围

29号拆分一下前后端+数据端模块 然后根据29号我们拆分的情况 去定技术评审

前端开发芹林、我写后端、玉龙管数据端 产品汤春艳老师

我们需要各个BU确认表结构才好定技术方案

最终会数据看板上线然后嵌入IM+后台

9.4号机票的表预计能给出



一期是没有下钻 然后用nova开发

二期是加了下钻 然后前后端 SOA开发



周末可以来看下dscim这个项目 一个全新的SOA框架的项目



### 答疑

1、一期改造的背景：

用nova实现，nova会有很多小问题，交互体验较差且部分BU以及指标缺失

而且nova的性能也较差

此外有些用户希望能按照日 周维度去看 现在还是月纬度

综合以上来看进行一期改造



2、二期改造和一期的区别：

因为二期其实就是加了下钻维度+更改了项目的实现 从nova改成了前后端

所以这么看其实一期改造完后和二期其实变化不大

产品的答复是：

一期主要是看概览，下钻的维度没有二期多

二期其实只有机票酒店 

一期是机票酒店商旅旅游火车票IBU







### 一期迭代

#### 交互搞

https://design.ctripcorp.com/s/66c75e5282c84e0bb8ee5868

1、哪些人有权限能看到这些数据 如果是只有老板可以的话 那需要给一份名单 而且一期迭代要做一个事情就是说让各BU的老板能看到自己BU的优先

2、趋势图里的异常数据？？怎么界定什么是异常数据，需要有一个清晰的定义 让程序感知

3、从一期的交互来看 貌似并没有发现下钻的点

4、服务水平SL这个指标是不做了嘛，我看表结构文档/交互稿理都没有这个指标，但是看板口径里有这个

5、我看各BU给的表结构里有度假门票租车用车等等，这个需要有事业部字段拆分，那我们到时候选旅游BU的时候做拆分嘛？



### 二期代码参考

disciimdataservice可以参考这个服务

金薇拿IM+二期给我展示的时候说 这些数据都是实时计算的 而且数据量很大 有千万级别

嘶？？这个实时计算应该就不是后端计算 感觉是玉龙flink job实时跑放进redis里的

不然后端算那么多肯定有问题的啊

而且看数据源 好像后端这边只有redis  

我们可以结合交互来看这个项目 



#### 多module模块的maven项目架构

1、子模块会继承父模块的groupId、version和一些配置，这样子模块的pom.xml会继承这些信息

2、一般父项目会选择pom打包，因为本身不执行，子项目选择jar或者war



#### 日志写入

用ConcurrentHashMap本地封装了一层缓存 

此外对Logger做了一层API的封装



#### **Redis**接入

首先注入CacheProvider对象

然后封装了一个ImSciRedisService即 可以理解为Mysql的Dao对象 专门对接Redis的





#### 代码改进点：

1、条件判断合并

```
if (CollectionUtils.isEmpty(keys)) {
    return null;
}
if (CollectionUtils.isEmpty(keys.stream().filter(Objects::nonNull).collect(Collectors.toList()))) {
    return null;
}
```

可以替换为

```java
if (CollectionUtils.isEmpty(keys) || keys.stream().noneMatch(Objects::nonNull)) {
    return null;
}
```



2、关于节假日日期这个点是否应该放配置中心

然后启动的时候直接先加载到本地呢？一是本地缓存最快减少网络开销的

而是也可以降低redis的qps



3、convert是否应该放在qconfig里

用热更新来处理？

这种情况可以使用@qconfig注解的监听模式，监听模式会在初始化阶段被调用一次

此外在qconfig发布配置的时候也会更新一次



4、key的设计

```
CHART_INFO_KEY_PATTERN = "detail_%s_%s:%s_%s_%s_%d";
DATA_KEY_PATTERN = "card_%s:%s_%s_%d_%s";
```

其实这里可以再规范一下的



#### 设计思想

1、无任何异常、且所有输出都做了兜底数据处理

即所有的异常都全部捕获 然后记录到Log里



2、无mysql，数据源全部上redis



3、请求全部异步化

所有的任务执行全部交给线程池

而且这个线程池还不是普通线程池，是一个异步服务专用线程池

你可以上mom平台上看 契约设计的时候就是Async异步







#### 代码借鉴点：

1、Preconditions.checkArgument（boolean expression,errorMessage）

如果不符合表达式会抛出IllegalArgumentException的异常 且带上errorMessage



2、线程池引入

```
public static final ListeningExecutorService SOA_ASYNC_POOL = MoreExecutors.listeningDecorator(CatAsync.wrap(new ThreadPoolExecutor(
        ThreadPoolConfig.getInstance().getOrDefault("soa.async.executor.core.size", 8),
        ThreadPoolConfig.getInstance().getOrDefault("soa.async.executor.max.size", 32),
        ThreadPoolConfig.getInstance().getOrDefault("soa.async.executor.keep.time", 5),
        TimeUnit.SECONDS,
        new LinkedBlockingDeque<>(ThreadPoolConfig.getInstance().getOrDefault("soa.async.executor.queue.size", 1024)),
        new ThreadFactoryBuilder().setNameFormat("soa-async-executor-%d").build(),
        new ThreadPoolExecutor.AbortPolicy())));
```

这里有好几个技术点：

- CatAsync.wrap() 这里是为了接入CAT异步埋点，还有种写法是对Runnable/Callable进行wrap，这里我们直接把Executor给wrap掉
- MoreExecutors.listeningDecorator是guava对JDK线程池的封装
- 我们用了guava的ThreadFactoryBuilder来帮助给线程池命名 方便定位问题
- 我们用了guava的MoreExecutors.listeningDecorator来封装ExecutorService，为了方便去使用ListenableFuture，相比JDK的Future，可以在一开始添加回调函数



3、SOA异步服务端怎么编写













#### 业务学习点

同比与环比

是两个常用的统计指标，用于比较不同时间段的数据，帮助分析数据趋势

同比：是指与上一年度同一时期的数据进行比较，通常用于分析年度变化趋势

环比：是指与上一时间周期（通常是上个月或上个季度的比较），帮助了解在连续时间周期的变化



那这里其实就有个疑问了：

我理解节假日这个选项点的意义：因为节假日意味着用户出行的需求会极大增加，老板会想看假节日数据会有什么变化

但是环比这个点的意义在嘛？是想看节假日和平常的对比嘛？

两点：

1、假设选的中秋 15 16 17三天 如果算环比应该是 12 13 14的对比，但其实收到中秋临近的影响 13 14的数据可能就没有那么“平常”了

2、第二个问题是节假日连在一起将来怎么办呢？比如有时候中秋国庆一起放假这种情况我们应该怎么设置呢？



#### 分工点

看起来下载这个接口也是前端做的

还有多语言支持看起来也是

然后获取筛选项里的那些locale啥的都没有用到



#### 代码实现思路

1、获取筛选项

纯粹走配置



2、日期格式化

采用LocalDate.parse来将日期进行格式化

用已经定义好的DateTimeFormatter定义好的格式





### 会议

#### 第一次会议 8.27

会议前准备点：

**会议目的**

这次会议主要是在交互搞出来的背景下，我们拉上春艳姐一起把交互和文档一起过一遍，达成一致

然后如果还有时间的话定一下前后端的交互方式，然后周四再定一下前后端技术实现、具体模块拆分



我的一些问题：

1、哪些人有权限能看到这些数据 如果是只有老板可以的话 那需要给一份名单 而且一期迭代要做一个事情就是说让各BU的老板能看到自己BU的优先

2、趋势图里的异常数据？？怎么界定什么是异常数据，需要有一个清晰的定义 让程序感知

3、从一期的交互来看 貌似并没有发现下钻的点 因为安装表结构的文档来说是有下钻字段的

4、服务水平SL这个指标是不做了嘛，我看表结构文档/交互稿理都没有这个指标，但是看板口径里有这个，确认一下IBU是15个指标 其他BU是19还是20个指标

5、我看各BU给的表结构里有度假门票租车用车等等，这个需要有事业部字段拆分，但是我看交互搞上面其实没有做这个点，那我们到时候选旅游BU的时候做拆分嘛？



会议后反馈：

 1、权限问题 应该大概率会白名单配置 有哪些权限 权限控制粒度到BU

2、趋势图里的异常数据算法：

小于25percentile-1.5*IQR或者大于75percentile+1.5*IQR

在JavaApiDemo里写了一个例子

这个例子适用于double也适用于long

3、一期的下钻会放在筛选框里 且不做服务SL指标

4、旅游BU给表的时候是度假 门票 租车 用车一起给 选BU的时候也分开选





#### 第二次会议 







### 文档修改点

http://conf.ctripcorp.com/pages/viewpage.action?pageId=2694940839

1、需求背景重新写一下 规范一点



2、写一下需求具体事项拆分、人力、风险点以及具体人力开发

风险点：权限校验风险点：哪些人该怎么配 员工号和eid还是有不同 有些老员工



还有个点后端怎么感知数据ready了？我建议也是走redis

这里有个点是我们数据日期什么时候出：

- 需要下钻维度的数据都ready 那一天的数据才可以ready
- Q季度内的也不行 只有全部好了才可



3、后端怎么感知数据ready时间？

两个key？

buTime:

{

buname

buEndTime

buBeginTime

}



4、节假日判断这块前端能做吗？如果不能做

我是不是应该把19年1月之后所有的节假日日期都返回给你











# 项目实现

## 项目搭建

#### 如何接入IM+客服管理平台

目前和芹林、鹏飞聊过一版后 我们不一定要接Ares轻应用发布 后续可以再研究一下

既然芹林已经上线过一版了，我们就按照他的来



在pass平台创建的时候发现Linux_tomcat类型

有web和service和Job三种类型，比较好奇是不是不同类型的流水线不一样

可以找一些项目试一下



参考：

**React中文官网**：https://zh-hans.react.dev/learn/start-a-new-react-project

**Ant-design**-PRO: https://pro.ant.design/zh-CN/

**UMI JS**:https://umijs.org/docs/guides/getting-started#%E4%BB%8E%E6%A8%A1%E6%9D%BF%E5%88%9B%E5%BB%BA%E9%A1%B9%E7%9B%AE

**Webpass开发传统应用：**https://node.fx.ctripcorp.com/docs/tutorial/common/guides/intro





#### UmiJS生成

我们使用nodejs20来开发，使用npx create-umi@lateset

在本地初始化代码

然后可以通过npm dev命令启动项目

如果要对接发布系统发布，即执行pnpm build命令即可，产物默认会生成 .dist目录下 完成构建后就可以把dist目录部署到服务器上



Node安装完成后会自带npm依赖管理工具，但Umi.js推荐使用pnpm来管理依赖



#### 目录结构



#### Redis接入

1、申请

生产环境：需要组织Redis审核人 + DBA审核

非生产环境：我们可以一键创建



创建成功后，我们可以在集群预览页面查看集群元信息（已用内存、节点信息等等）

然后hickwall上可以查看详细信息（qps、redis性能）



命令执行和调试：credis提供了命令行来操作redis里的数据



客户端配置：redis的超时时间概念

- connect timeout针对TCP三次握手的超时设置
- readTimeout 针对Redis的请求 长时间未回复的超时
- Pool Wait Timeout 是credis为了连接复用，使用了Apache对象池后针对从池子中拿到连接的超时时间，其逻辑很像线程池

此外客户端的配置很推荐用qconfig，新建继承文件继承10008425的credis.common.properties文件

然后再子文件里填写对应配置项和值（分集群粒度和应用维度）



我们在子文件里配了三个超时时间

并且开启了连接池预热功能



清明节：Ching Ming Festival

劳动节：Labor Day

端午节：Dragon Boat Festival

中秋节：Mid-Autumn Festival

国庆节：National Day

春节：Spring Festival of PRC

































## 后端项目实现

#### 筛选项+权限功能实现

对于权限这个点，因为业务特殊只有老板看，那么其实是可以让qconfig拉取到本地，然后本地做本地缓存的，因为这个信息是基本上不会变的，所以完全可以使用监听模式



#### 获取缓存数据实践

我们先思考极端情况：用户选择了92天，然后刚好环比同比日期都有

然后又是其他bu，有19个指标

那就是我们每个指标都会封装92*3个key

然后又有19个指标



https://www.cnblogs.com/andong2015/articles/10616582.html

根据这篇文章我们推荐单次mget的key在100以内最佳

这其实和我们项目预计最多的92天相符合 



因此在这个场景的设计上初步还是说

每个指标code 的同比环比 还有当前日期都去做一次查询



我们设置几个key




















































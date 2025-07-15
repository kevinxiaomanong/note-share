### 一、名词解释

- subject 消息的分类 隔离区分消息内容 是producer和consumer之间的桥梁
- consumer group 一条消息可以被多个消费者组消费，但一个消费者组内只能有一个消费者消费该消息



### **二、消息类型**

分为四种类型的消息

事务型消息

持久化消息 发送消息时会把消息放到db里

非持久化消息

非可靠消 息

当我们在系统中调用send以后，消息并不是马上发送到broker，而是先放到队列，由一个后台线程异步发送到broker，这时候有个问题如果当应用崩了或是宿主机宕机了，那么消息会丢失，  可以发送前先放到数据库中，发送完后再删掉，用补偿任务来做  

或是把订单状态更新和发送信息放在一个事务里



### 三、表初始化

针对事务型/持久型消息，会把消息存入db，这个db是业务的db，需要进行初始化



### 四、API调用

**producer**

qmq里的消息是通过K-V的形式来保存的，

发送消息的API里有两个参数，一个是message实体，一个是一个Listener，即一个发送成功或失败的回调函数

我们通过message

问题：是否在失败回调时重试

不建议，因为qmq自身有重试，默认三次，更应该去寻找发送失败的原因



如果消息过大，因为我们是发送成功后再删消息的，我们可以开启StoreAtFailed，即发送失败后，我们才把消息存起来



**consumer**





### 五、注意事项

1、qmq生产者发送消息记日志，不要在调用send方法前就记录日志，在onSuccess方法里记即可

2、消费时，在获取信息后，记日志其实记录messageId即可，没必要把整个Message都记下来





### 六、代码demo

**生产者**

```
@Configuration
public` `class` `ProducerConfig{
  ``@Bean
  ``MessageProducer producer(){
    ``MessageProducerProvider provider = ``new` `MessageProducerProvider();
    ``provider.init(); 
    ``return` `provider ;
 ``}
}
 

```

```
   @Service
public` `class` `QmqProducerDemo{
 ``private` `static` `final` `Logger logger = LoggerFactory.getLog(QmqProducerDemo.``class``);
  ``@Resource
  ``private` `MessageProducer producer;
 
  ``public` `void` `placeOrder(Order order){
    ``...
    ``Message message = producer.generateMessage(``"hotel.order.status.changed"``);
    ``message.setProperty(``"orderId"``, order.getOrderId());
    ``producer.sendMessage(message, ``new` `MessageSendStateListener() {
      ``@Override
      ``public` `void` `onSuccess(Message message) {
        ``logger.info(``"成功"``+message.getMessageId());
      ``}
      ``@Override
      ``public` `void` `onFailed(Message message) {
        ``logger.error(``"失败"``+message.getMessageId());
      ``}
    ``});
    ``...
 ``}
}
```

**消费者**

```
   @Service
public` `class` `QmqConsumerDemo{
 ``private` `static` `final` `Logger logger = LoggerFactory.getLogger(QmqConsumerDemo.``class``);
  ``@QmqConsumer``(prefix=``"hotel.order.status.changed"``, consumerGroup=``"推荐使用appid"``)
  ``public` `void` `onMessage(Message msg){
    ``//业务处理
    ``logger.info(``"开始消费"``+message.getMessageId())
  ``}
}
```





### 七、qmq原理

**术语**

消息：业务数据的载体，qmq用Message封装，数据结构为Map<String,String>

主题Topic：下游消费订阅数据时的唯一键，命名要规范

分区：消息主题的分片单元，本质是为了sharding数据，将数据打散到多个分区中



一般消费者组建议设置为应用id



消息可以分为：

普通消息（吞吐高）、顺序消息（分区内有序）、延迟消息、持久化消息（借助mysql实现生产者消息高可靠投递）



消费语义：

一般有至少一次、至多一次、精确一次

QMQ默认至少一次，这意味着某些场景下可能会导致重复消费



顺序消费

这个与生产者无关，通常采用一个分区在任何时刻绑定一个消费者方式，利用分区顺序性



消息位点：

消息按先后顺去存储在指定主题的分区中，每条消息在分区中都有一个唯一的序号，这个序号被定义为消息位点

而消费者消费的最新一条消息的需要，被定位为消费位点，qmq中以消费组为单位暴露消费位点

然后你的消息位点-消费位点，即为瞬时积压值，积压往往是瞬时的，如果积压指标持续上升需要处理



 首先 消息队列客户端为了高可用考量，通常维护一个内存队列，暂存即将要发送的消息，当达到预设的批次大小或者延迟时间超时后，才尝试将内存消息投递至服务端broker，即这种发送行为是异步的



消息轨迹：即一条消息从生产者发出到消费者接收并处理过程中，全链路的信息



消息标签：给消息打上一个或一组的标签tag





**流程图**

异步发送失败后，客户端自动发起重试

比较自然的做法是通过消息暂存后再补偿，暂存可以是实例本地磁盘/存储服务

这是一个本地事务场景

![image-20240116152916289](D:\工作文档\FrameWork\assets\image-20240116152916289.png)

watchdog：独立于业务进程的补偿服务

store：目前仅支持Mysql、SqlServer、MongoDB





**幂等消费**

因为网络的不可靠性等客观因素，QMQ不能保证一定不重发消息，所以consumer需要确保接收到重复的消息也能够正常处理，大多数情况我们的业务自身就具备去重功能，例如更改一个订单的状态，那你重发的时候业务很容易判断只来，但有些业务自身无法判断时QMQ提供了幂等检查器通过外部存储的方式来对消息去重

目前QMQ提供基于Mysql和Redis的幂等检查器

默认是根据messageId去重，也可自定义去重逻辑



**消息Tag**

生产者在生产消息时，可以为消息打上一些Tag，这样消费方可以只订阅含有自己感兴趣Tag的消息

注意一个应用实例内，同一主题和消费组仅能注册一个消费者，Tag规则的不同并不影响这个规则



**消费者组和分区的关系**

我们知道一个Topic下可以有多个分区，而一个消费者组内可以有多个消费实例，那么它们的对应关系是怎样的呢

1. Topic可以有多个消费者组，每个组都能拿到这个Topic下的所有数据，qmq里的消费位点是组隔离的
2. 一个分区的数据只能被某个消费者组里的某个消费者消费，因为如果多个消费者尝试消费同一个分区，会导致数据的重复消费和顺序混乱
3. kafka设计分区的初衷是为了让消费者组可以并行消费和负载均衡





### 八、接入事项

需要到qmq portal 门户上去创建主题（有BU专属独立集群的应优先选择相应集群）

QMQ里的一个主题可以对应多个生产者和消费者，有了topic后 还需要注册生产者、消费者信息

这些做好以后 就可以写代码了



如果要接入顺序消费，需要发邮件到R&D框架进行有序主题分区申请，然后消息要设置orderKey，会根据这个进行路由到同一个分区

顺序消费的原理是，这种模式下消费节点之间通过分工，互不共享数据分区，对于每个数据分区都由一个消费节点独占消费



如果要接入持久化消息，要因为qmq-dal，然后指定dal cluster



## conf：

https://pages.release.ctripcorp.com/framework/fx-docs/docs/qmq/tutorial/terms



### 术语：

消息，业务数据的载体，也是消息队列核心数据结构，qmq采用Map<String,String>落地

主题：用于对业务数据进行分类，也是下游消费订阅数据时的唯一键，主题名称需要按照“部门简称.产线.应用功能描述”的英文小写字符构成

分区：消息主题的分片单元，qmq利用分区来对数据进行分片，类比redis使用slot来进行分片

生产者：发布消息的实例，携程每个生产者会关联一个应用id，通过appid关联到owner和BU

消费者：订阅消息的实例，一般消费者以一个组为单位存在，被称为消费组，推荐用appid作为消费组名称

消息类型：普通消息、延迟消息、顺序消息、持久化消息

消费语义：一般有至少一次、至多一次、精确一次，QMQ默认提供的是至少一次的语义，意味着某些场景下可能导致重复消费 可以开启幂等消费来近似实现精确一次的效果

顺序消费：对于生产者无所谓顺序语义，顺序是消费方的需求，即分区内顺序，消费严格按照生产者投递至分区中的顺序，为保证顺序消费，通常采用一个分区咋任意时刻只会绑定至一个消费者的方式实现

消息轨迹：在一条消息从生产者发出到消费者接收并处理过程中，各个相关节点的时间、地点等数据汇聚而成的完整链路信息

消息标签：给消息打上一个或一组有特定意义的标签

消息位点：消息按照到达消息队列服务端的先后顺序存储在指定主题的分区中，每条消息在分区中有唯一的序号，这个序号被定位为消息位点

消费位点：消费者消费的最新一条消息的序号，被定义为消费位点，QMQ中以消费组为单位暴露消费位点

消费积压：消息位点-消费位点即为瞬时积压值，积压往往是瞬时的，如果积压指标持续上升，需要关注并处理



### demo

找mq热线的同事给了一个mq的demo，正好看看学习一下

这个demo其实提供了原生、spring、springboot三个版本的示例

我们来研究一下：



原生：

这里配置类用到了单例模式，发现我们对单例模式的理解和实现还是有偏差的

demo里使用饿汉式实现了Config的单例设计模式：

实现思路：

基于类加载机制来避免了多线程的同步问题



此外我们还可以使用推荐的双重检查锁机制来实现单例模式：

分别理解一下两次检查的作用：

第一个if是为了避免不必要的同步开销，如果实例已经创建，直接返回

第二个if是保证在多线程环境下实例只被创建一次，进入同步块后再次检查实例是否已经创建，防止多个线程同时进入同步块创建多个实例



双重检查锁 要加volatile关键字：

原因在于instance = new Singleton()这段代码其实分三步执行：

一是给对象分配内存空间

二是初始化instance

三是将instance指向分配的内存地址

由于JVM存在指令重排的特性，执行顺序可能变成1 3 2

只会导致在多线程环境 ：一个线程获得还没有初始化的实例



一般建议使用第三种饿汉方式，如果明确要实现lazy loading考虑静态内部类

如果有特殊需求考虑双重检查锁



这里其实还要了解一下类加载机制



### portal

在正式使用QMQ SDK进行消息发布与订阅前，需要现在protal上创建相关资源

要注意主题Topic是串联起生产者和消费者的一级资源，先有主题才有生产者和消费者
























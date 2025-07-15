### 一、核心功能

- 定义业务系统运行时所需的各种参数配置
- 对这些参数配置的变更能动态、实时地推送到SDK客户端（这样杜绝了为了改一个配置还要重新发布的场景）



其实配置中心是一个非常重要的组件，用于集中管理和动态更新应用配置

类似的技术栈有：spring cloud config、nacos等等



注意 不管是用java-api还是spring-boot注解 都可以做到热更新 java-api要加一个监听器

而使用qconfig注解所在的类必须为单例，否则会导致OOM



推荐使用@QMapConfig 可以将 kv文件解析成Map str str  

这里有个具体如何使用的配置搭配注解使用的例子



配置文件匹配：

map<str str>是可以和Properties类匹配的



最佳实践：

即远端去配置一个config.properties文件 然后利用@QMapConfig本地用map<string string>去接收 

接着我们每次通过map.get获取值 这样的好处在于 我们的值更新后 qconfig能推送过去 即热更新



### 二、架构

按照读写分离的架构去设计，即配置管理后台负责整个



### 三、环境

机器：

- -windows : C:\opt\settings\server.properties
- linux: /opt/settings/server.properties

qconfig:

fat、lpt、pro、uat 



两者对应关系

![image-20231124100752278](D:\Users\haoxiang_zhang\AppData\Roaming\Typora\typora-user-images\image-20231124100752278.png)





### 四、API使用

QConfig和Springboot基本上是零配置，注入依赖，用注解即可



通过mapconfig获取，如果想获取动态配置变更通知，使用addListener()

即异步回调，回调条件：

1、配置第一次加载成功

2、配置出现变更 



注解使用：@QConfig这个注解是支持热更新的，远端的配置变更也会跟着变

 如果是非实时场景，可以使用@QConfigProperty与@Value组合

使用客户端的feature特性，抑制配置不存在情况下不抛出异常，应用上线后新增配置



推荐@Qconfig解析json配置

@QMapConfig来读取properties配置



@QMapConfig的功能是很强大的，可以将kv解析成Map<STRING,sTRING>、Properties、对象

作用在类上，将kv的文件属性转为类的成员变量



此外还支持日志级别，即可以设置logLevel



### 五、VI点火插件

 客户端点火，用户可以指定一份文件，Qconfig在点火时会自动加载并实时监听变更

需要点火加载的文件必须实际存在且有权限访问，否则点火失败，例如你去加载其他应用的私有文件也会点火失败

即在resource下添加qconfig-ignite.properties文件



### 六、本地开发模式

在未连入公司网络下，我们可以将env设置为local来启用本地开发模式

env=local

在本地开发模式下，客户端会读取本地磁盘获取配置，不再请求网络，添加、修改配置都需要通过修改本地文件达成，本地的配置路径在c:\opt\config{appid}\qconfig



### 七、权限

1、captain中加入某应用，管理员默认可以浏览，但不能编辑、审核、发布

2、应用owner在qcofnig页面手动添加人员



## ihub--陈呈

qconfig作为集中管理应用程序/公共组件的参数配置，支持动态推送配置变更

可以通过Portal和API两种方式接入 有些时候当你的程序需要主动改配置



环境方面 qconfig提供了个resource环境 相当于各个环境的降级 可以把每个环境都相同的放resource下

此外通过子环境来支持分区域部署/差异化配置的需求，例如不同IDC需要不同配置，不同group下需要不同配置

文件类型：最多还是KV格式 还可以继承



依赖方面需要qconfig核心+VI的包



注解依赖：

<img src="D:\note\工作文档\FrameWork\assets\image-20240703171141360.png" alt="image-20240703171141360" style="zoom: 67%;" />

还有java api方式引入



同时有些时候希望预热时就加载好配置 而不是等流量来了再加载 可以配置点火预加载



**架构与原理**

<img src="D:\note\工作文档\FrameWork\assets\image-20240703171511146.png" alt="image-20240703171511146" style="zoom: 50%;" />



海外的服务一定程度是依赖上海的 因为都是



这里有一个关于上云的问题：

qconfig要上云 比如在SGP机房 部署 但是qconfig的配置希望不一样 应该怎么操作：

通过子环境来实现

即PRO环境---下面有SGP-ALI、SHARB等等region的子环境

然后再SHA的子环境操作即可



VI页面查看：

你可以在VI上看到某台机器上拉取到的qconfig的内存配置



基本原理是：客户端应用读取配置文件后，会通过长轮询请求持续监听该文件的变更，qconfig服务端感知到客户端的长轮询请求才会把客户端IP展示在前端页面

这里介绍一下长轮询：

1. 客户端请求：客户端想服务端发送HTTP请求
2. 服务器处理：服务器收到请求后，如果没有新数据会保持连接打开状态，直到有新数据或超时
3. 服务器响应：一旦有新数据或超时，服务器返回响应给客户端
4. 客户端处理：客户端处理完响应后，立即发送新的请求重复上述过程

长轮询在需要实时数据更新，但不希望频繁轮询的场景：

- 即时通讯 聊天应用中长轮询用于获取新消息
- 实时通知 例如监控系统某个指标超过阈值，服务器长轮询将通知推送至客户端
- 在线协作工具，长轮询获取其他用户的实时编辑操作
- 数据更新 例如股票行情、体育比分

而web-socket是一种全双工通信协议，允许在单个TCP连接上双向通信

客户端和服务器通过HTTP握手升级连接到WebSocket协议

一旦连接建立，客户端和服务器可以随时互相发生消息，而不需要重新建立连接

应该讲长轮询和websocket都是为了解决实时通信需求存在的技术，前者基于HTTP协议改进后者是一个独立的协议，专门为实时双向通信设计












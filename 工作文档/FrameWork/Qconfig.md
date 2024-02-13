### 一、核心功能

配置编辑、审核、发布

多文件类型支持 



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



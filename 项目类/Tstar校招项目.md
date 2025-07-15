## 前后端项目搭建

### 后端搭建问题

遇到个问题：项目本地运行无误 且push到gitlab后的流水线正常，最后也成功打包出来镜像，奇怪的是我基于这个镜像进行发布的时候，会出现ContainersNotReady状态问题，一直卡住发不上去，captain这边的报错是：

<img src="D:\note\工作文档\项目类\assets\image-20240727205916812.png" alt="image-20240727205916812"  />

点火404的问题，通常这类问题一般是db没发布或是qconfig变更这类的环境问题引发偏多，然后bat上是没有log的，甚至我还去找了DBA的人看我db的dal cluster是不是没发布，总之排查这个花了很久，主要没有很明显的报错，而且几乎是没什么log的（因为这里tomcat其实没跑起来都）

后来排查到发现pipeline里Java镜像构建的步骤里Tomcat版本是9，springboot3是必须用tomcat10的

于是我们修改一下pipeline的配置，然后再触发一次流水试试，果然这次顺利发布成功

![image-20240727210657925](D:\note\工作文档\项目类\assets\image-20240727210657925.png)





问题二：发布成功后发现访问404

解决办法：应该是servlet的问题 我们把一开始生成的代码的servlet跑上去

然后重新跑pipeline 再发布发现能跑上 我记得这个好像是springboot和tomcat的问题

后续加上了ServletInitializer得到了解决，这个可以查询一下Springboot的文档看看，这是因为

我们的镜像是用captain的镜像跑，是用外置的tomcat启动的，这也是为什么我们本地跑启动类能起来

但是发布到captain上404的问题



后续好像一直可以正常CI CD交付了



问题三：发布上去后发现后端服务访问百度接口报错，网关超时

这个问题我先咨询了SLB热线，然后再SRE群里问管理

得到的答案是：测试环境不能直接访问外网，需要特殊安全域qate-outbound才可以

并且创建这样的group是需要审批的

这里发现可以在captain上申请机器的登录权限，然后登录上机器后可以用curl去测试连接



问题四：接入SSO获取不到信息

接入文档：http://conf.ctripcorp.com/pages/viewpage.action?pageId=691693902

引入maven包会自动注册CtripSSOFilter，拦截路径进行登录认证（但如果servlet版本过低就需要手动配置Bean了）

获取不到信息很可能是因为：

1. 请求没有被配置到permissionconfig.xml
2. 实际访问的路径和xml不一致

我发现我的问题还是SSO Filter没有注册进去的原因，找了以前项目剑钊配的SSO filter注册进去就没问题了



问题五：跨域请求问题

在前后端联调的时候发现，前端请求后端服务的时候直接返回了<html>标签

说明前端的请求没有路由到后端，所谓的跨域其实是浏览器的同源策略限制了从一个源(域、协议和端口)

加载的脚本和不同源的资源进行交互，这在web开发里非常常见

即需要保证协议、域名、端口相同才能相互访问资源，如果两个URL不满足这个条件被视为不同源



解决方案：

服务器可以设置Access-Control-Allow-Origin头来指定允许访问的源

在同源的服务器上设置一个代理，将跨域请求转发到目标服务器，这样浏览器只与同源的代理服务器通信

JSONP动态创建<script>标签



问题六：如何通过两点的经纬度坐标计算距离

Haversine公式：

```
public static double calculateDistance(PointModel p1, PointModel p2) {
    final int R = 6371;
    double latDistance = Math.toRadians(p2.getLat() - p1.getLat());
    double lonDistance = Math.toRadians(p2.getLng() - p1.getLng());
    double a = Math.sin(latDistance / 2) * Math.sin(latDistance / 2)
            + Math.cos(Math.toRadians(p1.getLat())) * Math.cos(Math.toRadians(p2.getLat()))
            * Math.sin(lonDistance / 2) * Math.sin(lonDistance / 2);
    double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    double distance = R * c * 1000;
    return distance;
}
```





### 前端搭建流程

从代码搭建这块来 还是比较顺利的 

设置NPM镜像仓库后 安装miniapp脚手架工具

然后直接下载微信小程序开发者工具

然后直接本地miniapp init HelloMiniApp 用开发者工具导入项目 然后把申请到的moduleid放进配置文件里



发布流程：

1、先在MCD平台注册频道 然后再程里人运营管理平台配置相关信息 获取moduleid

2、MCD发布平台 选择好模块后 选择要发布的Trippal版本 并关联代码仓库设置发布分支 分别发送各个环境 如果需要再trippal工作台展示该小程序 需要配置并申请上线



后续发现TP小程序调用不了百度API接口，因为需要域名加白



前端思考点：

1、找出这个项目的元信息

appid（公司内部应用标识）、git仓库地址、

域名是什么？访问入口有哪些信息？

pipeline流水线配置（每一步做了什么事情）

集群和别的有什么不一样（为什么）？

 前端应用是怎么请求到后端的? 为什么要这么做？

代码分支是怎么组织的大家怎么协助开发的？

2、搭建流水线的时候遇到问题

我仓库流水线一开始用的是node serverless 无线版本

然后captain是接入Gitlab的CI CD的，有个意思的点是gitlab切换回正常的javascript后captain

跑的流水线并没有改，我是通过换了个仓库解决的，但是现在看来其实可以再换一次解决



### 后端技术选型

在启动后端项目前 我有思考后端服务的实现是走SLB访问还是SOA

嗯用确实能用，在mom系统构建好npm的依赖给前端用，但感觉现在SOA服务办公网络访问被禁止，前后端本地联调确实会不太方便，还是走SLB入口开发会好些，特别是这个项目目前不发布到生产只到测试做演示，且开发时间有限，您觉得呢？

其实我还是感觉SOA适合后端应用间调用，虽然前端调用也可以



### 后端搭建流程：

pass上新建应用---

然后上start/ 用新的项目底座 jdk21+tomcat10 把项目初始化

gitlab创建好group和app

captain上建好group集群 和slb入口

然后就可以push代码 跑流水线出镜像 然后captain发布了

发布完成后即可访问



**db层面**

申请数据库---DNA群里

等待db集群部署

部署完成后 就可以通过dal cluster连接了



### 后端组件集成

1、SSO登录集成 看下怎么获取到用户当前登录信息

2、swagger集成契约 怎么引入 这个看起来比较简单 引入以下依赖即可

```
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.0.2</version>
</dependency>
```

然后用一下swagger的一些依赖 就可以生成

3、HTTP连接池

基本原理是重用已经建立的HTTP连接，而不是为每个HTTP请求都建立一个新的连接，建立HTTP连接是一个相对耗时的工作涉及DNS解析、TCP三次握手，因此当一个HTTP请求完成后连接不会立即关闭而是返回连接池中

Apache HttpClient是一个流行的HTTP客户端库，提供了丰富的功能和灵活的配置选项，它支持同步和异步请求，提供内置的连接池管理，但要注意的是可能服务端会关闭这个连接

接入方案：

```
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
</dependency>
```

```
@Bean
public CloseableHttpClient httpClient(){
    PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager();
    connectionManager.setMaxTotal(64);
    connectionManager.setDefaultMaxPerRoute(16);
    return HttpClients.custom().setConnectionManager(connectionManager).build();
}
```





### 对接百度地图API

我们接地图组件 关键是获取AK

然后我们使用Web服务API

我们需要拼接HTTP请求URL，将申请的AK作为必填参数一同发送，然后web api的服务需要服务端AK调用

现在就是能调通 但是我们定一下几个关键点

返回结果有效信息：

routes返回的方案集：

里面有以下信息：

distance方案距离

duration:耗时

toll：过路费

![image-20240802150819690](D:\note\工作文档\项目类\assets\image-20240802150819690.png)



然后有个steps



和王小飞这边初步定了下后端一些指标的计算逻辑，下面就是看看我们怎么封装一下和百度API的交互

我们把需要用到的一些信息封装起来

我记得有http工具

怎么封装？

```
@Autowired
public BaiduMapService(CloseableHttpClient httpClient, ObjectMapper objectMapper) {
    this.httpClient = httpClient;
    this.objectMapper = objectMapper;
}
```

这种写法spring会自动将其Ioc容器中对应类型的bean注入到构造函数

奇怪的是Object Mapper这个类并没有声明Spring Bean但我还是正常引入进来了

因为我的项目引入了Jackson库的依赖，Spring Boot自动配置机制会扫描类路径中的依赖，并根据

这些依赖创建一些常用的Bean，而ObjectMapper就是其中之一

我换了一种写法：发现依赖也被顺利加载出来了

@Autowired
private CloseableHttpClient httpClient;

@Autowired
private ObjectMapper objectMapper;

但是这种方式改变了对象的不可变性

https://api.map.baidu.com/directionlite/v1/driving?origin=40.01116%2C116.339303&destination=39.936404%2C116.452562&ak=Dms3AJq6MgjbBTNUm36uvCnNB5EXGaIz
































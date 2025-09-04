## 一、项目信息

daas接口地址：

http://daas.ops.ctripcorp.com/api-config
搜索 stamkt

prd：http://conf.ctripcorp.com/display/dsjyaiyy/STA+Marketing+Campaign+Performance+Monitoring+Solution+-+PRD

UI：https://www.figma.com/design/GwYuojd6mPzSyYbTsBnt4Q/%E8%BF%AA%E6%8B%9C%E6%95%B0%E6%8D%AE%E5%A4%A7%E5%B1%8F?node-id=316-2052&p=f&t=oUYZdlPQrj0qBWSC-0

STA前端应用：https://captain.release.ctripcorp.com/app/100055477/info

后端应用：https://captain.release.ctripcorp.com/app/100055203/info

TRIP登录：https://pages.release.ctripcorp.com/ibu-gcc-group/ibu-account-group-doc/accounts/url.html



测试环境查找验证码：http://databank.fws.qa.nt.ctripcorp.com/DataBank/LoginRegister.jsp



STA二期PRD：https://trip.larkenterprise.com/wiki/ZNTMwHSiSiw76Gk5gQUcYpd8nJe







## 二、接口文档

1、获取



UI与指标对齐：

Saudi order&trend

http://daas.ops.ctripcorp.com/api/v1/adm_sci_prj_stamkt_index_mi

搜索指数--订单指数 （月维度）



Competing Dest

http://daas.ops.ctripcorp.com/api/v1/adm_sci_prj_stamkt_index_mi

短线搜索指数



Saudi best things to do(分页接口)



竞对国家信息:

http://daas.ops.ctripcorp.com/api-config/1456

长短线



月度指标这块：

GMV:

缺少一个酒店取消率



{
    "Trains": "http://kefu.train.ctripcorp.com/offline/order/train/detail?orderNumber=",
    "Hotel": "http://supplierweb.order.hotel.ctripcorp.com/order/orderdetail/",
    "Flight": "http://process.flight.ctripcorp.com/fltorder/order/index?module=61386&orderid=",
    "Piao": "http://ordermanage.ttd.ctripcorp.com/v2/detail?orderid=",
    "Activity": "http://htlint.ctripcorp.com/OrderOperate/Order/OrderDetail/",
    "Car": "http://order.car.ctripcorp.com/sdo/customerservice/dist/#/orderInfo/info?orderId=",
    "Tuan": "http://supplierweb.order.hotel.ctripcorp.com/order/orderdetail/",
    "QiChe": "http://customer.qiche.ctripcorp.com/#/offline/orderDetailBus?lineType=1&orderNumber=",
    "Cruise": "http://order.cruise.ctripcorp.com/detail?orderId=",
    "Customized": "http://order.package.ctripcorp.com/tour/order_index/operateOrder?orderId=",
    "Exchange": "http://forex.ctripcorp.com/fx-order-web/Order/Detail?OrderID=",
    "Market": "http://offline.ypmall.ctripcorp.com/mall-business/index#/serverOrderList?orderID=",
    "FishTrip": "http://bnb.ctripcorp.com/bnb-oms/order/",
    "MktSupermember": "http://adv.ctripcorp.com/membercardoffline/?module=221128#/guestCardSearch/",
    "Ship": "http://customer.qiche.ctripcorp.com/#/offline/orderDetailBus?lineType=2&orderNumber=",
    "SceneryHotel": "http://order.shx.ctripcorp.com/SHX-Order-OrderOperate/OrderDetail/OrderDetail?orderID=",
    "Meeting": "http://order.mtg.ctripcorp.com/mtg-order-ordermis/OrderDetail/OrderDetail/",
    "Sport": "http://sport.market.ctripcorp.com/sport-offline/#/info/",
    "Guider": "http://order.package.ctripcorp.com/Tour-Order-Orderoperate/OperateOrder.aspx?orderid=",
    "FinIns": "http://htlint.ctripcorp.com/OrderOperate/Order/OrderDetail/",
    "Mice": "http://kefu.train.ctripcorp.com/offline/order/train/detail?orderNumber=",
    "Vacation": "http://membersint.members.ctripcorp.com/modulejump/tNetv.aspx?module=4126&location=frame&OrderID="
}



## 三、代码优化

handler/Advice

可以看到这个类里面两个核心接口：

- beforeBodtWrite: 当supports方法返回true时，该方法会在响应体被写入之前调用，允许对控制器的返回值进行二次处理，这里我们主要是将返回值包装成统一的RestResponse格式







sheet

我们确定一下怎么拼sheet

长短线拼一个

按照rank+type排序



htl和flt拼一个

前半部分放htl 后半部分flt flt的excel里的star按空处理





GMV 除以1w --> 取整（.前面取整 后面保留两位有效数字00做展示）

AVG price： （直接取整+两位数字展示00）

长短线竞争系数（小数 真实两位数字）

star 0-3星整合 方便展示 防止过多

机票取消率 保留四位有效数字 excel下载样式带百分号5.68% 





## 四、海外目的地代码迁移



```
.DS_Store
.idea
.vscode
node_modules
.next
package-lock.json
next.config.js
release
.history
eslint-report.json
build
_nfes_types
coverage
```





### 迁移问题-如何实现多个海外目的地基于同一trip.com体系的权限校验

首先我们看Umi框架的前端应用配置文件app.ts

```
getInitialState
```

- 这些数据可在整个应用的组件中通过 `useModel('@@initialState')` 访问。



```
render
```

Umi的渲染拦截器，在应用首次渲染前执行登录校验，控制页面是否允许访问

如果是开发环境：直接跳过登录校验，方便本地开发调试

生产环境：通过getSession检查登录状态

如果登录了将maskedEmail存入localStorage 正常渲染

如果未登录 那么清楚本地存储的邮箱 跳转到登录页面，并携带当前页面URL作为回调地址，登录成功后自动返回原页面



request：

请求拦截器

所有请求发送前，自动从localStorage中读取maskedEmail 并添加到请求头中





现在我们其实希望：

- 用户访问www.trip.com/overseas/sta 

首先校验用户登录态--->提示无权限



现在有个问题是：

如果用户用自己的账号登入，其实页面也能看到

所以我们其实少了一个鉴权的逻辑，具体鉴权我们放在页面组件上面

但我在想这样会不会存在单点登录问题





### 优化点：

- 酒店星级三星以下过滤
- 城市-星级（倒序）组合排序
- 机酒竞争系数（保留三位）
- GMV除以一百万+保留两位有效数字 前端加Million处理



## 五、海外目的地架构方面

1、海外用户访问：

默认解析并接入到akamai，akamai是CDN服务提供商

akamai就近接入后默认转发到SIN-AWS的SLB

SIN-AWS的SLB通过path，结合当前应用配置策略判断这部分流量：

1. SIN-AWS本地处理
2. 转至IBU私有VPC
3. 转发回SH

SIN-AWS到SH 链路也是先AWS->HK: 公网 然后HK->SH 专线 这也是默认链路



2、国内用户访问trip.com

由于trip.com未本案



**CDN内容分发网络**

内容指的是：静态资源比如图片、视频、文档、JS/CSS/HTML

分发网络：即将静态资源分发到位于多个不同地址位置机房中的服务器上这样就可以实现静态资源的就近访问

并且减轻服务器以及带宽的负担

然后我们再聊流量链路SOTP



## 六、代码优化

**1、http客户端连接参数**

```
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
cm.setMaxTotal(200);                // 最大连接数
cm.setDefaultMaxPerRoute(50);       // 每个路由（域名）的最大连接数
cm.setValidateAfterInactivity(2000); // 连接复用前的验证时间

CloseableHttpClient httpClient = HttpClients.custom()
    .setConnectionManager(cm)
    .setConnectionTimeToLive(30, TimeUnit.SECONDS) // 连接存活时间
    .build();
```

我们以Apache HttpClient为例，核心参数：

- maxTotal（最大连接数）
- defaultMaxPerRoute ()针对单个域名（路由）的最大并发连接数
- validateAfterInactivity(连接复用前的验证时间)：连接空闲超过此时间后，复用前会检查连接有效性
- connectionTimeToLive：连接创建后允许的最长存活时间，避免长时间占用资源



要注意的是 这个连接存活时间（keep-alive）双方都可以设置，因为TCP连接是双向的，任何一方都可以主动关闭连接

此外关于validateAfterInactivity这个参数是在复用长连接时避免使用失效连接，从而提高HTTP请求的稳定性，即HTTP连接池里的连接空闲一段时间后，再次被使用时需要进行有效性的验证，来确保连接仍然可用

Apache HttpClient为例来看验证逻辑：

1、查看连接是否已关闭：底层Socket的isClosed

2、验证连接空闲时间是否超过服务器规定的keep-alive超时时间（响应头Keep-Alive：timeout=xxx获取）

HTTP长连接的核心是复用连接以减少握手开销



**长连接介绍**

HTTP协议最初的设计是短连接：请求完成后TCP连接被立即关闭，因此每次请求都需要三次握手新建议TCP连接然后四次挥手关闭连接，而TCP握手的耗时（通常几十到几百毫秒）会显著增加请求延迟

为了解决这个问题，HTTP1.1引入了长连接（Keep-Alive），通过在响应头中添加Connection：Keep-Alive，客户端和服务器约定在一次TCP连接上可以连续发送多个Http请求，连接不立即关闭而是保持一段时间供后续请求复用

而HTTP连接池的原理：基于HTTP长连接，通过池化技术对连接进行生命周期管理（创建、复用、销毁），实现高效复用，降低请求延迟



**过度封装优化**

```
public List<FltIndexDto> getFltInfo(String reqParamJson) {
        List<FltIndexDto> fltIndexDtoList = requestDassStaApi(DaasStaApiEnum.FLT_AVG_RANK, reqParamJson, FltIndexDto.class);
        return fltIndexDtoList;
    }

    public List<HtlIndexDto> getHtlInfo(String reqParamJson) {
        List<HtlIndexDto> htlIndexDtoList = requestDassStaApi(DaasStaApiEnum.HTL_AVG_RANK, reqParamJson, HtlIndexDto.class);
        return htlIndexDtoList;
    }
```

你看其实这里代码没有逻辑就是纯透传，这样很容易陷入回调地狱



**异常捕获思考**





**2、Excel下载优化**

发现如果去掉poi的maven包之后 在执行EasyExcel.write(response.getOutputStream()).build()的时候报错：

java.lang.NoClassDefFoundError: org/apache/xmlbeans/XmlException



因此还是需要把



**3、异常处理**

```
    public static String toJson(Object object) throws JsonProcessingException {
        try {
            return objectMapper.writeValueAsString(object);
        } catch (JsonProcessingException e) {
            LOGGER.error("Error converting object to JSON, Object: " + object, e);
            throw e;
        }
    }
```

这里我们在JsonUtils内把异常捕获的方式是：

先记录日志，再向上抛出，其实这样有点不太合理

作为工具类建议移除日志记录，保持工具类职责单一，让调用方负责日志记录和处理是更好的选择

因为如果你这里抛出了，调用方可能也会记录相同日志，可能操作日志重复

而且还有一个点是 你这里其实不好排查





**4、限流**

背景：对于downloadExcel这个接口，其实对数据库的查询压力较大，因为我们会一次性下载多个sheet

也就意味着我们会查询多次请求，而我们的数据库是starrocks这种OLAP引擎，OLAP是重分析的B端，对于高并发场景支持不友好，为了保护集群我们要尽量控制qps



而Starrocks是开源的MPP大规模并行处理分析型数据库，专注海量数据的实时分析场景，支持高并发查询、低延迟响应和复杂SQL分析，设计上融合和传统数据库的数据管理能力和现代查询引擎的跨源计算能力

存储：

- 自有存储引擎：支持数据直接写入并存储
- 元数据管理：
- 完整SQL支持
- 独立计算调度：基于MPP架构，集群节点分为FE（SQL解析、优化、调度）和BE（数据存储和计算），可独立完成“数据写入-存储-查询”

查询：

- 支持外部表External Table：无需将数据导入StarRocks
- 联邦查询能力：将自身存储数据与外部数据源的数据联合查询，例如StarRocks中的订单表JOIN Hive用户表
- 数据湖架构中，用户常使用Spark SQL、Presto作为查询引擎，但StarRocks响应快部署简单，注解替代这些工具直接查询

两者身份不矛盾，而是现代分析系统“存储与集散解耦”趋势的体现



**5、登录与鉴权&无权限异常提示**

背景：之前我们约定的是前端请求把maskedEmail放在request head里

但这个maskedEmail的值其实是de***t@trip.com  这样从安全的角度上看 可能会伪造请求头攻击

而且如果有相同的账号可能存在小概率掩码冲突，因此我们需要重新设计一套鉴权逻辑：

前端采用bindEmail拿到真实Email，然后加salt后：再加密放在请求头里请求传给后端



那么这里我们应该怎么处理通用返回呢？

--前后端统一返回类RestResponse来做

后端在处理请求时，会通过该类统一封装响应：

- **成功场景**：返回`success=true` + 业务数据`data`（如`RestResponse.success(userList)`）。
- **失败场景**：返回`success=false` + 错误信息`msg`（如`RestResponse.failed("参数错误")`），通常配合全局异常处理器（`@ControllerAdvice`）捕获所有异常并转换为`RestResponse`格式。

前端处理：

通过拦截器（Axios拦截器）来对后端响应进行统一解析，根据success字段判断请求状态，来实现“一处配置，全局生效”



这里记录一个问题：HTTP协议规范请求头字段是大小写不敏感的，但在实际传输中浏览器或底层网络库可能对字段名称格式化处理，所以即使我们在前端代码里写：

```
bindEmail: hashEmail(getLocalStorage(BINDEMAIL_EMAIL)),
```

但在浏览器前端会转为Bindemail



对Java的Throwable类，也就是Exception的父类来说，getMessage()和getLocalizedMessage(）的区别：

- getMessage() 获取原始异常信息，即返回异常对象创建时传入的原始消息字符串，消息内容固定和本地化无关，无论系统语言或地区如何，返回的都是构造时的原始字符串
- getLocalizedMessage：获取本地化异常信息，即根据当前系统的语言/地区Locale适配后的信息



**6、无限循环打印异常**

```
    @ResponseStatus(HttpStatus.OK)
    @ExceptionHandler(Exception.class)
    public RestResponse exceptionHandler(ServletWebRequest request, Exception e) {
        final String requestURI = request.getRequest().getRequestURI();
        LOGGER.error("Path: {}, Cause: {}", requestURI, e);
        String error = StringUtils.isNotBlank(e.getMessage()) ? e.getMessage() : e.toString();
        return RestResponse.failed(error);
    }
```

打印内容：

/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta/OverseasDest/sta

分析：无限循环打印同一个路径说明：

在处理异常的过程中又抛出了新的异常，而这个新的异常再次被全局异常处理器捕获，导致异常，比如日志打印本身可能触发了异常



**7、前端提示**

这里分两种情况：（以HTTP响应码是否正常为例）

- HTTP响应码正常，但response.fail 我们在Umi的request拦截器里面实现

```
responseInterceptors: [
    (response) => {
      // 拦截响应数据，进行个性化处理
      const { data } = response;
      if (!data.success) {
        message.error(data.msg || 'request fail!');
      }
      return response;
    },
  ],
```

- HTTP请求异常，我们在前端进行try-catch


```
const handleRequestError = (error: any, defaultMsg: string) => {
  const errorMsg = error?.response?.data?.msg || defaultMsg;
  message.error(errorMsg);
  throw new Error(errorMsg);
};

// 获取POI排名
export async function getPoiRank(data: any) {
  try {
    return await request('/OverseasDest/sta/getPoiRank', {
      method: 'POST',
      data,
    });
  } catch (error) {
    handleRequestError(error, 'getPoiRank failed');
  }
}
```

**8、线程池管理**

其实在我们没有显示使用线程池技术的情况下，应用也能处理并发请求，其核心依赖嵌入式服务器（Tomcat）的线程池管理，Tomcat服务器本身是基于线程池设计的，用于处理并发HTTP请求

以默认Tomcat为例，其线程模型分为：

- Acceptor线程 负责“接收连接” tomcat会启动少量Acceptor线程来监听服务器端口（如8080），接收客户端发起的TCP连接，接收连接后Acceptor线程不会直接处理请求而是将连接交给工作线程池处理
- Worker线程池 负责“处理请求” 从解析HTTP请求到调用Spring MVC组件（DispatcherServlet->Controller->Service）最后生成HTTP响应返回给客户端，每个并发请求分配一个Worker线程处理完后回到线程池等待下一个请求（避免频繁创建/销毁线程的开销）

因此我们的应用能够处理并发请求核心在于Tomcat的Worker线程池



而我们的Tomcat本身就是一个Java进程，如果是走内嵌Tomcat，作为内嵌组件与应用代码共享同一个JVM进程

思考一下war包的好处在于对于独立的Tomcat来说有更成熟的监控、日志收集、启停脚本等工具支持，但jar包其实更贴近云原生，但也可将独立Tomcat作为基础镜像，将War包放入镜像的webapps目录，本质仍是容器内独立的tomcat进程，



这里记录一下Controller层关于CompletableFuture的写法：其实Springboot框架会帮忙处理，等待获取异步结果后才返回客户端



**9、线程池调参**

先明确哪些地方将来会被用上线程池：并发http调用daas接口

对于IO密集型任务，线程池参数的优化核心在于“让线程尽可能多的处于等待IO的状态”，同时避免线程过多导致的内存开销和上下文切换损耗，我们以服务器（2核CPU、4GB内存、IO密集型任务）配置为例来做优化：

首先这类任务特点是线程大部分时间都在等待IO响应（CPU空闲），因此需要更多线程去填满CPU的空闲时间，来提高资源利用率，经验公式：

- 核心线程数 = CPU核心数*2-4 （根据IO等待时间长短调整）
- 最大线程数=核心线程数*2-4避免线程过多导致内存溢出

拒绝策略：CallerRunsPolicy是IO密集型任务的优选：当线程池满时让提交任务的线程（比如Tomcat的worker线程）亲自执行任务，其实相当于让调用方放慢提交速度，既避免任务丢失，又天然起到限流作用（比直接抛异常友好）



那我们接着来看如果是计算密集型任务，即存在复杂数据运算、大量循环处理，该如何优化？首先明确该类任务的核心特点在于线程大部分时间在占用CPU进行计算（几乎没有IO等待），此时过多的线程会导致CPU上下文切换频繁，反而降低效率，此类任务性能瓶颈在于CPU处理能力而非线程数量，核心原则是线程数量应尽量匹配CPU核心数，避免线程过多导致的CPU上下文切换损耗，经验公式：

- 核心线程数：CPU核心数或+1（+1的目的是为了应对某个线程因GC或因极少量IO短暂阻塞），这样有备用线程去应对CPU空闲
- 最大线程数 = 核心线程数

同时计算密集型任务的队列应尽可能大，缓冲更多任务避免频繁创建新线程，并且空闲线程的存活时间应较长

此外拒绝策略建议使用：AbortPolicy或DiscardOldestpolicy，不适合让提交线程去执行（占用tomcat线程，影响其他请求）

对于计算密集型任务有一些额外的优化建议：

- 任务分片，将大计算任务拆分成小分片，比如把100w条数据拆分成10个10w条的子任务交给线程池并行处理，避免单任务占用线程过久
- 避免同步阻塞，计算密集型任务应尽量避免线程同步（如synchroinzed、锁竞争），减少线程等待时间，让CPU始终处于计算状态
- 监控CPU利用率稳定在70-80（预留部分资源给系统和其他进程），若长期低于50说明线程数不足，若长期100且响应变慢，可能是线程过多导致切换损耗

核心是少而精的线程+大队列



**10、第三方类注入**

```
@Configuration
public class ThreadPoolMonitorConfig {
    @Resource(name = "asyncTaskExecutor")
    private ThreadPoolTaskExecutor asyncTaskExecutor;
    @Resource
    private MeterRegistry meterRegistry;
```

这里很有意思：MeterRegistry是一个第三方类，我们没有手动创建bean，但确能注入这个Bean

这是因为Springboot帮我们做了自动配置，是一种约定大于配置理念的实现，减少开发者手动配置的工作量

根据类路径中存在的依赖，自动注册必要的Bean到容器中，在我们这个demo里面**`MeterRegistry`**

我们引入了spring-boot-starter-actuator一来，触发Actuator自定配置类，会帮我们自动注册一个MeterRegistry的实现类，默认是SimpleMeterRegistry，如果引入了prometheus等扩展，会是PrometheusMeterRegistry

只有当你需要自定义`MeterRegistry`的行为（如修改指标命名规则、添加全局标签等）时，才需要手动定义`MeterRegistry`的`@Bean`，此时自动配置会优先使用你的自定义 Bean。



**自动配置**

Springboot将常见场景的Bean注册逻辑封装成自动配置类，开发者只需引入依赖即可快速开发，日常开发中Web服务器、数据源、日志、缓存等几乎所有基础组件的集成都可以依赖

并且我们可以去通过同名@Bean自定义覆盖，和通过配置文件来修改默认参数



**11、进程与JVM**

一个Java进程对应一个独立的JVM实例，启动一个Java进行会在操作系统中创建一个独立的JVM，相互隔离资源不共享，我们要理解进程是操作系统进行资源分配（内存、CPU时间片）的基本单位，每个进程有自己独立的内存空间、fb文件描述符等，JVM本身以进程的形式存在于操作系统中

**二者关系：**

- JVM是逻辑规范，定义了类加载机制、垃圾回收、内存模型等，负责将Java字节码翻译成机器码并执行
- 进程是物理实体，JVM的运行依赖操作系统的进程机制，进程为JVM提供了运行时所需环境（内存、CPU）和隔离环境



**12、线程池+连接池优化excel下载场景**

优化核心：串行获取多份--->并行执行http接口调用，利用线程池减少整体数据准备时间

首先要明确这些是否可并行，有无依赖关系，将独立的数据获取操作提交到线程池并行发起http请求减少总等待时间，使用CompetableFuture管理等待所有完成后再组装

![image-20250811203423647](D:\编程文档\note-share\项目类\assets\image-20250811203423647.png)

生产上面这个接口性能不好 要3s左右 在资源紧张的情况下甚至要到5s多

我们尝试进行优化 并且前端这个提示好像有点问题



前端提示这个问题在于：

下载excel这个接口返回的其实是bob类型，但我们在前端返回拦截器里面

```
 if ('success' in data && !data.success) {
        message.error(data.msg || 'request fail!');
      }
```

一开始是直接去拿data.success，这肯定拿不到然后就会报错



其实这个异步Competable优化还有一个点：就是如果我异步的结果要等待返回，返回后我还要处理，该怎么和spring框架联动呢？



**13、ThreadLocal线程隔离存储当前market信息**

ThreadLocal（线程本地变量）是Java一种线程隔离机制，

```
    private static ThreadLocal<List<String>> MARKET_HOLDER = new ThreadLocal<>();
   
```

存储当前线程的market列表，核心是让每个线程持有自己的变量副本，其底层依赖Thread类里一个特殊的成员变量threadLocals（类型为ThreadLocalMap），存储该线程所有ThreadLocal变量

ThreadLocalMap则是以ThreadLocal实例为key，以线程私有变量为value

线程隔离的本质：每个请求由Tomcat的一个独立线程处理，实现请求之间market隔离



**为什么需要？**

在Web场景，多个请求会被不同线程同时处理，market是请求级别的参数

- 如果是全局普通变量来存储，会导致线程安全问题（线程A的market可能被线程B覆盖）
- 如果通过方法参数来传递market会导致代码冗余

而ThreadLocal+Request拦截器很好的解决了这个问题



**注意点**

使用ThreadLocal要注意内存泄漏问题（线程结束后，threadLocals中数据未被清理，长期占用内存）

Tomcat等web服务器的线程是线程池复用的（线程不会随请求结束而销毁，会被重新用于处理新请求）

并且ThreadLocalMap的key是弱引用，value是强引用，可能导致value无法被回收



这里聊下弱引用：当JVM进行GC的时候无论内存是否充足都会回收被弱引用关联的对象

两个问题：

1、为什么value是强引用可能导致无法回收

当key（ThreadLocal实例）被GC后，value仍然被ThreadLocalMap的Entry强引用，如果线程长期存活比如说是Tomcat的核心线程，那么value会一直占用内存

```
public class MarketContext {
    private static ThreadLocal<List<String>> MARKET_HOLDER = new ThreadLocal<>();
    public static void setMarket(List<String> marketList) {
        MARKET_HOLDER.set(marketList);
    }
    public static List<String> getMarket() {
        return MARKET_HOLDER.get();
    }
    public static void clear() {
        MARKET_HOLDER.remove();
    }
}
```

目前MARKET_HOLDER被MarketContext类的静态变量强引用，因此key（MARKET_HOLDER）不会被GC回收，因为有强引用，可如果MARKET_HOLDER被重新赋值或是MarketContext类被卸载，那么MARKET_HOLDER被回收了以后，ThreadLocalMap中对应的Entry的key变为null，但Entry对value的引用是强引用，且线程可能长期存活，形成了一条Thread-->ThreadLocalMap-->Entry-->value因此无法被GC回收，即使已经没有实际用途还存在内存中

而我们通过在afterCompletion中调用了ThreadLocal.remove()，相当是在当前线程直接清除了ThreadLocalMap中对应的Entry（key和value），这也是ThreadLocal使用的黄金原则：“用完必须手动清理”



2、为什么ThreadLocalMap里Entry里的key是WeekReference呢？

即如果你的代码中不再使用ThreadLocal对象，但Threads里的ThreadLocalMap的key仍然指向它，那么不再被需要的ThreadLocal会一直被持有无法被GC，设计为弱引用可以让外部没有强引用指向ThreadLocal对象时GC回收掉，回收之后Thread里的ThreadLocalMap中的key变为null



3、ThreadLocalMap里的key实现

ThreadLocalMap里的Entry其实不是普通HashMap的键值对结构，它通过继承WeakReference<ThreadLocal<?>>实现了对ThreadLocal实例的弱引用

引用描述的是一个对象是怎么引用另一个对象的，而一个对象可以同时持有多种引用类型，就像一个人一样，可以弱握着一根香蕉的同时可以硬握着一个苹果，这是一个道理



4、日常开发中弱引用怎么引入



**14、本地缓存**

这里可以针对getMonth()做一个优化













## 七、 异步翻译

前端如何做限流？

阿联酋迪拜+阿布扎比7日跟团游·【全景体验 让你省心】登顶棕榈岛52层&世界奇迹全览+卢浮宫+总统府+大清真寺+复古木船游河+古堡集市等全部入内|打卡双地标：迪拜塔&金相框|沙迦文化精髓深度游|国际5钻酒店|纯玩含小费 A线

ight UAE group tour: Dubai + Abu Dhabi + Sharjah + Ajman four-country tour - Route B

p tour in Dubai + Abu Dhabi, UAE - [Comprehensive experience for your convenience] Ascend to the 52nd floor of Palm Island & explore world wonders + Louvre Abu Dhabi + Presidential Palace + Grand Mosque + traditional dhow cruise + visit to Al Seef Heritage Market | Visit two landmarks: Burj Khalifa & Dubai Frame | In-depth cultural tour of Sharjah | International 5-diamond hotels | All-inclusive with tips - Route A



阿联酋7日5晚跟团游·迪拜+阿布扎比+沙迦+阿治曼四国游 B_B线

ight UAE group tour: Dubai + Abu Dhabi + Sharjah + Ajman four-country tour - Route B





maxQps 已线下沟通，实际会低于10

测试环境token: 100055203-TEST-faf27a76
 生产环境 token: 100055203-PROD-cbfaba44



![image-20250814135203685](D:\编程文档\note-share\项目类\assets\image-20250814135203685.png)



现在线上翻译有问题，需要你修改你怎么做？

首先消费这个是需要支持修改的





ight UAE group tour: Dubai + Abu Dhabi + Sharjah + Ajman four-country tour - Route B

p tour of Dubai and Abu Dhabi, UAE · Palm Island Observation Deck 52nd floor + Louvre + Presidential Palace + Grand Mosque all included + Abra boat experience | Visit Burj Khalifa & Dubai Frame | Explore Sharjah Museum of Islamic Civilization + Iran Town | 5-diamond hotels throughout | Tips included (Option B)



ight group tour to Dubai + Abu Dhabi, UAE - [Official Flagship Recommended] High meal inclusion rate*Includes desert safari*Palm Island luxury car tour*EK direct flight | International 5-diamond hotel | Louvre Museum entry + Dubai Creek cruise | Full-day free time in Dubai | Includes guide service + hotel tax | Selected departures receive complimentary Lost Chambers Aquarium



100055203-c0a83201-487542-2000046





## 八、优化点记录

1、考虑引入本地缓存，降低Daas调用频率，保护sr集群，同时提升性能

本地缓存的实现方案要么是HashMap手动实现（考虑线程安全&过期处理）或是比较成熟的库（Guava Cache、Caffeine）这些库已经封装了过期策略和并发控制，稳定性更好

**缓存操作：**

查询缓存，如果存在且未过期则返回；否则调用daas接口获取数据更新缓存后返回，同时要处理并发问题（缓存穿透/击穿），击穿可用互斥锁或Caffeine的load方法（自动处理并发加载）即只有一个线程去加载其他线程等待

**缓存结构**

key（请求参数序列化后的字符串）

value（接口返回数据） 过期时间1小时 Caffeine支持写入后多久过期

**容量限制**

避免内存溢出，当超过时使用淘汰策略



2、关于stream流的写法

背景：在STA T站我们有两种写法

```
return searchOrderVtoList.stream().map(StaTripSearchOrder::format).sorted()
                .collect(Collectors.groupingBy(StaTripSearchOrder::getLocale));
```



```
 return staTripDecisions.stream().collect(Collectors.groupingBy(StaTripDecision::getLocale,
                        Collectors.collectingAndThen(
                                Collectors.toList(),
                                list -> {
                                    list.sort(StaTripDecision.bizMonthComparator());
                                    return list;
                                }
                        )
                ));
```

我们分析一下这两种写法是否都能保证返回List的有序性呢？

--其实不然，sorted()是在groupingby之前执行的只是保证进入分组前整个流的顺序，分组过程会破坏之前的排序

第二种写法collectingAndThen在每个分组完成后执行了排序，为每个分组的List单独调用了sort方法

而且Java Stream API不保证groupingBy收集器会按照元素进入的顺序处理，可能重新排序或并行处理元素



因此我们来学习一下Stream API里的一些写法Collector，作用是对另外一个收集器的结果进行二次处理

允许我们再收集操作结束后，对最终结果执行一个额外的转换或处理步骤

```
public static <T, A, R, RR> Collector<T, A, RR> collectingAndThen(
    Collector<T, A, R> downstream,  // 基础收集器（先执行的收集操作）
    Function<R, RR> finisher        // 对收集结果的二次处理函数
)
```

这是我们的用法：使用 `groupingBy` 分组后，若需要对每个分组的结果做转换（如转为不可变集合），可嵌套 `collectingAndThen`：



**Java Stream操作优化**

Stream流属于Java8引入的核心特性，设计初衷是为了简化集合的批量数据处理，通过声明式API提高代码可读性和可维护性，且天然支持并行处理以提升大数据量下的效率

将数据源（如集合、数组）转换为流，通过一系列 “中间操作” 构建处理管道，最终通过 “终端操作” 得到结果

中间操作可被优化（如合并、短路），提升效率，其实有点类似Flink



重要：Stream 的**中间操作是 “惰性的”**—— 仅当终端操作被调用时，中间操作才会实际执行。这种设计允许 Stream 优化处理过程（如合并操作、提前终止）。



无状态与有状态操作分离：

中间操作分无状态和有状态两种，这种区分是为了优化并行处理

- 无状态操作：每个元素的处理不依赖其他元素（filter、map），可独立并行处理
- 有状态操作：处理某元素可能依赖其他元素（sort、distinct、limit）并行时需要额外协调成本

stream可以通过 `parallelStream()` 或 `stream().parallel()` 转换为并行流，其内部基于Fork/Join框架自动拆分数据、分配多线程最终合并，极大简化了并行编程





**Stream流操作分类**

1、创建流

- 基于集合/数组
- 自己传入参数

2、中间操作

- flatMap（扁平处理），即将T扁平成stream子流，会自动合并进入主流
- peek 只用来遍历元素

3、终端操作

- count 统计元素数量
- forEach（Consumer） 遍历元素 无返回值
- sum/max/min 需要先转换为数值
- match类（anyMatch/allMatch/nonMatch）
- find类（findFirst、findAny）
- reduce类（归约）将元素合并成单个结果
- collect(Collector):收集流结果（如转集合、分组等）

Stream 的处理过程可分为**三个阶段**：**创建流 → 中间操作链 → 终端操作**

要注意终端操作执行后，Stream就被消费了，无法再次使用否则会抛IllegalStateException



其中Collector这个接口比较复杂，封装了收集过程的四个核心步骤：

1. 创建容器 supplier
2. 累加元素accumulator
3. 合并容器 combiner
4. 转换结果 finisher

JDK提供了很多Collectors工具类，内置了大量常用收集器，主要场景有：

1. 基础收集：转为集合或数组
2. 聚合统计类  （大多是转数值后分析）
3. 分组与分区

- 分组 groupingBy 按某个属性将元素分为多个组

groupingBy(Function)  groupingBy(Function, Collector)

partitioningBy(Predicate)

- 分区 partitioningBy 按boolean条件分为两组

4、字符串处理 即将流中的字符串元素拼接成一个字符串

支持直接/分隔符 拼接

5、collectingAndThen二次转换与包装

如果是很复杂的场景还可以通过Collector.of()去自定义收集器



**踩坑点：peek**

```
List<String> collect = Stream.of("zh-CN", "en-US", "fr-FR")
                .peek(s -> s.replace("-", "_"))
                .collect(Collectors.toList());
List<StaTripHtlIndex> staTripHtlIndexList = safeGet(htlIndexFuture, Collections.emptyList()).stream()
                .peek(staTripHtlIndex -> staTripHtlIndex.setLocale(localeMarketMap.get(staTripHtlIndex.getLocale())))
                .map(StaTripHtlIndex::processFormat).collect(Collectors.toList());
```

这两次peek一个成功修改了对象的属性 一个却没有

这其实是取决于元素本身是否是“可变对象”：

- peek的作用 主要是消费流元素，如果元素是可变对象，peek中可修改对象的属性，因为操作的是对象本身，但像String这种不可变对象，不可变对象的操作会返回新对象，原对象不变

以s -> s.replace(...)为例，只是调用了方法并生成了新对象，但没有将新对象替换回流中，流中依然是原来的String对象，因此如果你需要修改String等不可变对象需要用map操作，因为map会将函数返回值作为流的新元素去替换



3、代码优化 简洁

使用peek优化



## 二期需求点：

STA二期PRD：https://trip.larkenterprise.com/wiki/ZNTMwHSiSiw76Gk5gQUcYpd8nJe



好的那总结一下开发工作： 

1、STA C站+T站外部看板大改 （前端加菜单页切换） 

2、外站营销活动数据上传&展示（存储用户上传数据无系数处理） 

3、清单工具实现（要求PAX倒序、时间段、随机三种筛选+清单实时效果预览） 

4、营销活动数据调节器（分locale+自然月维度 实时效果预览） 

5、PKG翻译审批流（接入飞书审批或开发审批系统+支持用户编辑翻译结果）



### 2.1 迁移&改造

我们把之前DET系数调节的接口都迁移到新应用上面

这样fws 工具类的请求为：

fws：http://bdsci.overseas.fat0.tripqate.com/OverseasDest/tool/***

prod: 可以申请一个新的域名



新起一个工具前端应用 去取代之前的

fws：http://bdsci.overseas.fat0.tripqate.com/internal/tool



那我们首先要把之前的服务迁移到后端服务上面



换个思路：

我们不影响之前的请求





我们先列一下后端有哪些需要迁移：

1. 系数调节相关

```
DetCoefficientController
```

我们依赖要迁移这个类下的接口

依赖项：

```
DetCoefficientService
```

```
DetDataService
```



我们还是新起了一个工具后端应用：



### 2.2 建表设计

```
use bdcoefficientdb;
CREATE TABLE `ctrip_order_operate` ( 
id bigint NOT NULL AUTO_INCREMENT COMMENT '主键id' ,
operator varchar(50) NULL default NULL COMMENT '操作人' ,
operateTime varchar(50) NULL default NULL COMMENT '操作人' ,
datachange_lasttime datetime(3)  NOT NULL default CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间' ,
PRIMARY KEY (id),
KEY ix_datachange_lasttime(datachange_lasttime),
)  DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='C站订单明细表';

```

索引设计主要取决于查询模式：

1、会根据订单id、站点、产线、城市、下单日期查询 

2、根据PAX 间夜量 票量去排序

3、根据bizMonth+id_deleted去分析PAX和GMV



单字段在多条件查询时可能效率不高，联合索引的设计需要遵循最左前缀原则，把过滤性高（区分度大）的字段放在前面，此外不要过度索引，会增加写入的开销占用存储空间，只需要为常用的查询条件创建索引即可



这里注意一个概念，过滤性高的应该是高基数字段过滤性更强



针对WHERE/ORDER/GROUP字段建立索引，此外注意避免冗余索引（即如果已存在联合索引a/b 则无需单独为a建索引）

is_deleted这个逻辑删除字段几乎所有查询都会携带，因此如果数据量大缺少索引可能走全表扫描，但is_deleted=0的记录占据大多数，单字段的索引效果很有限，建议与其他高频字段组成联合索引



那既然我们聊到了索引，其实不妨深入一点了解Mysql的B+树索引结构

首先B+树是一种分层的平衡树结构，所有数据都存储在最底层的叶子节点中，非叶子节点（根节点、中间节点）仅存储索引键和子节点指针，用于导航查询路径，一个经典的B+树分为三层（实际层数根据数据量动态增长）

第一层：根节点：最顶层节点存储少量索引键和指向中间节点的指针 为查询的入口



B+树索引分为聚簇索引和二级索引，二者结构类似，只是叶子节点存储的数据不同（前者存储整行记录，后者存储主键id）一般通过二级索引找到记录主键id再回表查询完整记录



B+树结构优势：

- 查询性能稳定，因为每次查询都是从根节点到叶子节点
- 非叶子结点不存储数据，仅存储索引项+子结点索引，使得非叶子节点能存储更多索引项，使得整个B+树索引的层高一般都在3-4层，减少磁盘I/O，而磁盘I/O是数据库类的瓶颈
- 对范围查询很友好，因为叶子节点被双向链表串联起来了，当找到起始叶子节点时不再需要回溯上层节点，可以直接通过叶子节点双向链表直接遍历下去
- 排序友好，ORDER BY索引键的时候可以直接利用叶子节点排序，减少额外排序的开销



**树高与磁盘IO**

为什么B+树高会影响磁盘IO次数呢？

---这和B+树存储结构和磁盘数据的读取方式有关

B+树的每个节点在物理存储上都对应一个或多个磁盘页（默认页大小是16kb）

磁盘的读写有一个重要特点：无法像内存一样随机访问单个字节，而是按页读取，即每次IO操作会读取一整个磁盘页的数据，即使只需要其中的一个值，所以每次访问B+树的一个节点，就需要触发一次磁盘IO，因为节点存储在磁盘页，需先加载到内存

树高越低，从根节点到叶子经过的节点数越少，触发磁盘IO次数越少，可以用矮胖的形象来比喻



**磁盘IO为什么是数据库性能的主要瓶颈**

核心还是在于磁盘与内存的速度鸿沟以及数据库的核心操作（读写数据、索引查询）对磁盘的强依赖



数据库的数据最终存储在磁盘（机械硬盘HDD/固态硬盘SSD），但所有运算（查询过滤、排序、连接）都必须在内存中完成，因此数据从磁盘-->内存的加载读IO 内存--->磁盘的持久化写IO本质是“慢设备”与“快设备”之间的数据搬运，而两者的速度差异大到无法忽视



**快慢设备其实很类似计算机存储体系**



| 存储设备        | 单次读写延迟（典型值） | 每秒 IO 次数（IOPS，理想值）         |
| --------------- | ---------------------- | ------------------------------------ |
| 内存（DDR4）    | 约 10-100 纳秒（ns）   | 无上限（内存带宽通常以 GB/s 计）     |
| 固态硬盘（SSD） | 约 100-1000 微秒（μs） | 数万次（普通 SSD 约 1-10 万 IOPS）   |
| 机械硬盘（HDD） | 约 5-10 毫秒（ms）     | 仅数十次（普通 HDD 约 100-200 IOPS） |



所以哪怕一次不起眼的磁盘IO 消耗的时间比内存执行非常多运算总时间还长

数据库的核心工作是 “频繁读写数据”（比如查一条记录要读索引 + 数据页，写一条记录要写数据页 + 日志页），每一次操作几乎都绕不开磁盘 IO（除非数据已在内存缓存中），这种 “慢 IO” 自然会成为性能的主要拖累。



此外数据库的IO多是随机IO，磁盘对随机访问更不友好，尤其是机械硬盘对随机访问效率极低

顺序IO VS 随机IO：

- 顺序IO 数据在磁盘上连续存储，读取时磁头HDD或闪存控制器SSD无需频繁移动，一次能读写一大片数据效率高
- 随机IO 数据在磁盘上分散存储，每次读写需要定位到不同的物理为止（HDD要移动磁头 转动磁盘找扇区 SSD虽没磁头 但随机寻址也有额外开销）单次IO的定位成本远高传输成本

数据库查询场景中随机IO占比很高，

以机械硬盘为例：一次随机 IO 的 “寻道时间 + 旋转延迟” 可能占总耗时的 90% 以上（比如磁头移动到目标磁道要 3 毫秒，磁盘转动到目标扇区要 2 毫秒，而实际传输 16KB 数据仅需 0.01 毫秒）。这种 “随机 IO 效率低” 的特性，进一步放大了磁盘 IO 对性能的影响。



此外内存缓存优先



二期外部看板接口：

1、Ctrip 内部活动接口开发&T站内部活动开发 （C内部活动参与系数调节，依赖系数表）

2、C站外部接口开发（依赖站外数据上传工具，本期走上传表，调用Daas接口落库）

3、T站外部接口开发（接入daas即可）

4、预定天数接口开发

5、Total Pax





数仓

1、Market ---> Business Unit  (Auto Populated)



活动：

站内站外放一起

C站直接读mysql外站表+系数表

T站内：Daas接口下载



订单明细：

读订单明细表



6保留Excel结构即可 无数据



7保留结构



8 T写死Saudi Arabia 

C写死China-MainLand  Saudi Arabia



C下载的时候 F列展示Destination Market (Drop Down) 值为：Saudi Arabia

T下载的时候 F列保留Destination City (Drop Down) 值为 Daas接口里城市字段

C下载的时候 N列会展示PKG均价
























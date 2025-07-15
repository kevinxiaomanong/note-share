http://conf.ctripcorp.com/pages/viewpage.action?pageId=159204317

携程的三个核心概念：

- Pool 服务器集群
- APP 一个具体的应用
- Group 一个应用具体的部署实例

SLB中的三个核心概念

- SLB 代表一个具体的SLB实例 是由一组具体的SLB服务器组成
- VS 代表一个域名管理单元 可以有多个域名 但它们的职责功能配置绝对一致
- Group 一个应用在SLB上的投影，是SLB的基本管理单元，大多数运维操作是针对Group的



**VS**

是域名服务在SLB上的投影，VS定义了一组域名和一个端口，代表了可以接收来自这组域名的这个端口的流量

一个VS组里的域名拥有完全相同功能，有个VS规约

- 想通域名、协议的VS在同一套IDC上只能有一个
- 但允许在不同的IDC的SLB上出现 ctrip域名双活即为此原理



**SLB的基本原则**

1. 路由负载 根据应用配置的访问入口 将请求路由到一个具体的Group，再根据Group中服务器的权重设置，实现Group内部服务器的负载
2. 扩缩容 可以增加或缩减Group中的服务器
3. 拉入拉出 可以临时将Group中的故障服务器拉出，停止服务，也可以将已恢复的故障服务器拉回Group
4. 监控检测 即对Group中的服务器自动健康检测
5. 灰度分流 将流量在不同Group间进行灰度分流



**添加访问入口**

即应用服务暴露给外界的访问方式，携程目前绝大多数使用HTTP(S) SLB

在captain里集群管理新增SLB访问入口



**解绑访问入口**

即将域名访问入口从应用中下线 可以直接在Captain上操作



**服务器的拉入拉出**

在SLBPortal查询到需要修改的group 在member面板对机器进行拉入拉出

ServerStatus：

MemberStatus：拉入拉出Group成员服务器，仅拉出当前Group中的这台机器

PullStatus：发布拉出状态，由发布系统控制拉入拉出，例如mirror group就是让发布系统控制是否拉入拉出

HealthStatus：即健康检测状态，由健康检测系统负责拉入拉出



### 实践

我们以100030389应用为例来在SLB Portal和Captain上看看

在生产环境里发现下面有4个Groups（两个生产一个镜像一个虚拟的阿里云）

其中mirror这个group的流量入口是封住的，让captain发布系统去接管，这样在其他group无法提供服务的时候

再开启mirror入口



然后每个group内挂载了多个虚拟服务器即VS 



感觉SLB维度的东西不应该给我们查

因为这个SLB下面关联了太多应用了



所以我们重点关注：group即可用区的集群，member即可用机器，VS即虚拟入口




















































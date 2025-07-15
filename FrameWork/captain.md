### 一、使用说明

先在pass平台创建一个应用，成功会得到一个唯一的appId

绑定好gitlab里的仓库 就可以开通流水线

通常应用提交代码后，会经由gitlab ci生成pipeline，跑完流水线后会生成captain可发布的镜像，审批通过的镜像可以发测试和生产



### 二、DR高可用

DR即Disaster Recovery灾难恢复，我们通过创建多数据中心的DR Group提供可用性



### 携程CI CD现状

Gitlab用于存储代码触发Pipeline，Pipeline过程中会使用Sonar、JaCoCo，安全扫描等第三方工具，最终产生Docker镜像，Captain订阅产生的镜像用于发布，SLB、SOA等Operator对Pod进行拉入拉出



真正的云原生：镜像+容器+声明式API（K8s controller）



captain的发布方式--堡垒+金丝雀+滚动发布

即我们是先把一台实例从集群里拉出，进行堡垒测试，堡垒测试是不接入流量的，测试完后，我们再接入一小部分流量，即灰度发布，看在真实流量下能否抗住，没问题再滚动发布金版本

当发布只改镜像时，重建容器而非Pod，Pod是K8s创建或部署的最小单位，一个Pod封装一个或多个容器，存储资源、网络IP



## Captain官方文档介绍

Captain是基于k8s的新一代pass平台，提供了弹性伸缩、基于镜像的CI CD等面向云原生的功能



使用说明：

先到pass上创建一个新应用 选择框架与语言 然后生成app id 接着绑定gitlab仓库 后续便可以到captain里发布



**镜像管理**

通常应用提交代码后，由gitlab ci生成pipeline，跑完流水线后会生成Captain可发布的镜像，审批通过后才能发测试和生产




















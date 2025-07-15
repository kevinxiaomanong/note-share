http://conf.ctripcorp.com/pages/viewpage.action?pageId=1902716722



DR即Disaster Recovery 灾难恢复，为了保障重要的应用能够持续稳定提供线上服务，用户可以通过

创建多IDC的DR Group提高应用的可用性



**DR 创建流程**

方法一：

先部署好一个集群

1. 新建一个group
2. 如果是web应用需要再group上创建好slb入口
3. 新建变更并发布成功

新建DR group 

这种方式可以自动创建出GSLB 无需再找SRE手动配置



方法二：适用于没有使用方法一，但已经手动部署好两个集群的情况

绑定DR group 然后如果有slb入口的话需要找SRE配置GSLB

配置了GSLB的域名称作DR域名，即入口的域名解析可以指向多个集群，（域名指向可在webinfo/中查找）

也就是说一个域名指向多个不同集群

如果没有配置GSLB的被称作单边域名 配置了GSLB后才能在st2/上切换不同集群的流量

DR域名通常是比较重要且用途较广的域名

traveldata.bdai.ctripcorp.com

traveldata.ctrip.com

http://webinfo7.ops.ctripcorp.com/#/relation/tstar2024group12cofare.fws.qa.nt.ctripcorp.com#chain

tr043713.mit.ctripcorp.com

https://traveldata.bdai.ctripcorp.com/

http://dataapprove.fat3875.qa.nt.ctripcorp.com/

tstar2024group12cofare.fws.qa.nt.ctripcorp.com



DR治理

![image-20240725142748557](D:\note\工作文档\conf\assets\image-20240725142748557.png)



接着我们去看了100030389的BAT 发现有个IDC的访问量明显不如另一个

猜测原因即traveldata.bdai.ctripcorp.com这个域名解析出来的流量都去了日版这个IDC

所以我们可以试下 当把域名解析的DR解决好之后 再去BAT看下两个不同IDC下请求量是否一致

**思考**

之所以XY这个IDC还有流量 是因为这个DR group有些域名是能正常被解析到两个IDC的





tstar.cofare.fws.qa.nt.ctripcorp.com




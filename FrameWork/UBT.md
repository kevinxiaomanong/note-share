### 埋点类型

业务类埋点：BI trace 曝光 PV 业绩类统计

埋点和日志挂钩的，日志是流水账，体积大

埋点是行为追踪，pageview核心PV数据

业务侧主要关注四个：pageview、trace、action、exposure









UBT关键概念：

CID：标识一个设备的唯一ID，APP卸载重装后CID不变，生成方式设备id传入服务端根据appid生成cid

VID：用于标识一次完整App声明周期，APP卸载重装后会重新生成，生成方式UUID存本地文件

SID：用于标识一次会话，PV间隔超过30min生成一次会话，超时创建新会话，本地自增id

PVID：在一次会话中PV的ID 本地自增ID

GEO：

- gpsXXX：基于客户端GPS定位拿到的准确位置
- latitude/longtitude/city 通过IP定位到的大概位置

UID：账号信息，app和web端有差异

buID&brandID：部门ID 区分海外UBT国内UBT数据 以及哪个团队的





客户端上报给到服务端采集入口， 采集到后会写到UBT的kafka里，bi天池的分发给业务用，然后给到各个业务线取数据，zeus离线采集到hive

还有就是flink实时消费topic，写到starrock库fxubt里面，给mpaas平台用记录用户访问流（已经从CK转到SR了）注意SR只保存一个月的数据，一年的数据在离线hive库里面



信息量：

pageview：

![image-20251015144636456](D:\编程文档\note-share\FrameWork\assets\image-20251015144636456.png)



在SGP上面也布置了一套ubt采集，这个就是给出海的应用埋点采集用（比如trip.com travix），但目前还是通过mirror load方式回流到上海（目前已经阻断了UBT回流）



这里很有意思：

通过固定维度采样上报埋点（appID/Bu）查看总体上报埋点趋势

基于Clickhouse物化视图来做 记录所有埋点count 实时分析



实时存储1个月 离线数据1年



如果你查询一年以前的hive，那其实需要去回刷数据，成本比较高，查询一次十几块钱

支持各维度查询，各类ID映射成VID（如果根据uid查不到说明你没有跟vid关联起来）



全量UBT数据实时写入Clickhouse


































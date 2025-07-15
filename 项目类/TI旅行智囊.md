## 数据存储架构

后端这边就是存mysql -- ditidb 存一些业务元数据

数仓这边是hive+starrocks 数据在hive这边处理加工 数仓开发生产好的数据导入到starrocks里 加速查询



## 业务架构

TI主要的服务还是卖大屏以及提供数据服务

主要是大数据可视化以及一个取数的SOA服务



![image-20240711112402919](D:\note\工作文档\项目类\assets\image-20240711112402919.png)

人工取数审批流程系统100043021

对外输出数据job 100030081



携程统一指标系统 100033528

大数据可视化 100030389

旅行智囊 100017114

数据智能权限系统 100016107  没有业务依赖使用 流量入口已销毁

SOA服务 100027314 20340



100018538 这个app 挂载了elasticsearch服务



项目现状 旅行智囊已经没有使用 现在主要是大数据可视化 然后权限管理是交给另一个系统来做

那个系统的管理员只有周伟

此外TI有个重要的SOA服务对外 对内也有很多在用

还有个人工取数审批系统和对外输出job

现在主要搞清楚两点：

1、starrocks怎么查的

2、权限怎么管控的 这个是通过mysql表做的



### 技改需求

在100030389这个项目 支持中英两种 代码实现上是通过语言类来实现的 希望让语种业务和代码能解耦开

本质是通过请求头里的Lang来判断语种 然后如果是英文在TIRequest的feature字段加一个_eng

区分是在mysql表里有 

provinceName -- provinceengname 这样的区别 即每个语种多加一个字段



这里技改其实不难 关键是得把这个项目的业务捋清楚

还是得搞清楚 这些指标哪来的



遇到个很有意思的问题

我在本地跑前端 有些地方调不通

但如果我直接跑localhost：8082能跑起来 好奇怪 是前端路由的问题嘛？



SELECT resident_prov_name, CAST(persons * 1.0 / SUM(persons) OVER () AS DECIMAL(9, 4)) AS ratio FROM ( SELECT resident_prov_name, SUM(person_num) AS persons FROM ctripdi_prodb.adm_order_allbu_share_day WHERE d >= '2021-11-01' AND d <= '2025-11-01' AND to_country_name ='中国' AND to_prov_name ='湖北' AND to_parent_city_name ='荆州' AND resident_country_name ='中国'  AND resident_country_name = '中国'  AND resident_prov_name IS NOT NULL GROUP BY resident_prov_name ORDER BY persons DESC LIMIT 20 ) t



这里的逻辑是优先看feature指标的engine查询 如果没有默认用qconfig里的配置兜底



接下来做技改：

1、中英文拆解 拆解的时候一定要考虑如果后期还要加语种 进来 能够方便加入

先想下 如果要做多语种 那说明语种就要做成可配置的形式



全局search 中英文默认不一样 一个中国上海 一个Singapore

searchCountry接口





## SR部分

**集群监控地址**

http://bat.fx.ctripcorp.com/d/wpcA3tG7z/dp_starrocks?from=now-3h&to=now&var-cluster_name=di_ti_xy&var-fe_master=10.110.63.198:8030&var-fe_instance=10.110.63.163:8030&var-be_instance=10.110.63.169:8040&var-interval=1m&orgId=0&refresh=30s

可以定期观察一下



此外要理解 不是每个请求都会到SR的 如果feature指标和client一致 会命中本地缓存


























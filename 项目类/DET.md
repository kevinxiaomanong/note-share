### 方案研究

1、IM看板 我们采用前后端做 东方总被投诉过 不会再用前后端去做这种看板 导致我们后续IM需求都接不了

2、系数调节 听起来是一个数据端的需求 为什么会牵扯到前端？预览这个事情其实完全可以做两个 这个功能其实完全不依赖前端 本质还是数仓是否具备实时调节能力 

3、用nova做 不用数据看板 没有那么重前端



本质还是如果能把sql隐藏 

这是一个对外的看板 系数变动是不能被外部看到的

所以如果sql不能隐藏 那么系数的变更 可能要有个原始表+系数 那么逻辑的加工要放在数据链路里

如果能隐藏的话 那其实就把数据的逻辑直接做在sql里面 就可以实时生效

用户调整即可 维护两个看板



目前看来这个项目的方案就是前后端

系数调节器的前端



### 链接

nova内部看板：

https://artnova.ops.ctripcorp.com/#/dashboard/b32b3e78-efdf-400a-af0a-15216d3602f9

PRD：

http://conf.ctripcorp.com/display/dsjyaiyy/DET+Marketing+Campaign+Performance+Monitoring+Solutio+-+PRD

开发文档：

http://conf.ctripcorp.com/pages/viewpage.action?pageId=3759838950

系数调节器UX：

https://qzo46dfhzd.feishu.cn/docx/Ar0DdvWZRoeGY2x1YlociIAYnbg

外部看板UX：

https://design.ctripcorp.com/legacy/6800e065a10aafb8326c0d5f

Swagger：

http://det.coef.sci.ctripcorp.com/swagger-ui/index.html#



### 提点：

1、尽量快速先把一个功能实现 样式先不管 能给大家去check 提点



2、如果有UED 那其实就是照着UED去实现 但如果没有 你就先把功能点实现出来 然后让大家去check这个东西

最忌讳的就是 你按照你的审美最后调出来一版 但可能最终的那个效果 其实不是大家想看到的



### UED交互评审问题：

1、目标用户是迪拜



2、登录校验问题

看下怎么接集团的登录



### 项目搭建

系数服务后端：

bbz：平台研发中心





系数服务前端：



mysql：

bdcoefficientdb

bdcoefficientdb_dalcluster



### 数据库设计

指标:



系数：



### 本地缓存引入

1、问题解决

异常提示更改：别报空指针异常



做一个缓存开关（qconfig控制）





























































## 后端开发

### 一、引入http线程池





### 二、http请求

http://conf.ctripcorp.com/pages/viewpage.action?pageId=2979390460







### 三、指标相关

指标转换率 的计算 其实都是除 近九十天



第一类：

impression：曝光（周维度）

click：点击（周维度）

CTR：点击（周维度）/ 当周曝光





Homepage、pkg、flt、htl 大搜 跟团游 机票 酒店搜索

total：整体相加



第二类：

homepage：大搜/90天曝光量 先各自聚合

nova：计算逻辑：homepage搜索量





pkg：跟团游/90曝光



第三类：

ordernum：订单量系数

![image-20250513164243641](D:\note\工作文档\项目类\assets\image-20250513164243641.png)















### ESlint

我们直接删掉.eslintrc.js



后端接口：





## 前端开发







## todo：

1、后端下载excel

分sheet下载



我们先考虑把RestResponse的逻辑 放在切面里面



2、request head-鉴权 maskedemail





#### 系数初始化流程：

1、首先进入系数调节页面

我们会有个接口 获取当前周状态

该接口逻辑为：从筛选项接口获取最新有数周 并且填充周状态



接着我们会获取当周原始数据，其实这里只需要拿原始userIndex+orderIndex即可了

这里就不涉及任何系数调节，纯读daas接口



接着会查询当周系数情况，注意这里如果当周没有系数，我们需要做一个初始化，初始化系数为1



然后用户就可以在页面上实时调节了



调节完成保存好系数后，如果觉得OK，那就可以锁定一下周状态为对外展示



那么在对外获取筛选项的接口，我们就可以：

如果发现周状态为1展示态，那么可以将其该为2 

且将该周的所有系数锁定 

如果发现周状态为0 调节态 那么需要把这个周移出筛选项



这样状态即结束

todo：

1、增加展示态按钮



2、修改对外筛选项接口



3、添加系数









[detteam@trip.com](mailto:detteam@trip.com)
Dubai123.



https://www.fat1.qa.nt.tripqate.com/account/signin?backurl=http://sci.coefficient.fws.tripqate.com/&hidethirdparty=true





bd.sci.det.trip.com





如果trip.com域名默认回上海的话，



170194

2142913





843289





1、下午介绍一下功能

- 页面实时调节
- 并发保存检测
- 开关&系数条件状态
- 系数历史



2、目前代办：

前端无法访问----智哥解决

生产环境系数数据初始化



如何接入TP通知









https://www.trip.com/overseas/det



https://www.trip.com/overseas/det







slb已经提示：

添加新访问路径时需要在每个IDC都创建入口，如果对应IDC没有Group的需要创建跨IDC的访问入口,除了当前入口还需要于下面几个IDC创建入口。(SHA-ALI,SHARB), time: 93

添加新访问路径时需要在每个IDC都创建入口，如果对应IDC没有Group的需要创建跨IDC的访问入口,除了当前入口还需要于下面几个IDC创建入口。(SHA-ALI,SHARB)



现在有这样一个场景问题：

1.如果某一周数据没调--->会remove掉



那怎么补系数呢？

那我建议还是系数调机器页面应该支持 查看过往周数据





det为什么不对





2025-10-20~2025-10-26   1.702 575

2025-11-10~2025-11-16  1.602 570












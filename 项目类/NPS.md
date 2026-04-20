### 一、项目元信息

app：https://captain.release.ctripcorp.com/app/100035077/info

前端打包地址：http://eros.ares.ctripcorp.com/#/files/pages.ares.ctripcorp.com/nps-coding-analysis-module

appid：100035077

db：spqanpscodinganalysisdb

zeus：http://new.metadata.ops.ctripcorp.com/#/metadata/HIVE/qa_db/adm_tel_answer_final_nps_output

生产：http://nps.coding.analysis.kefu.ctripcorp.com/home

NPS系统task_detail更新逻辑梳理：https://trip.larkenterprise.com/wiki/ZSkQwP6L1ioSX9k8vZFc947mnSf

hive:qa_db.adm_tel_answer_final_nps_output

```
pip install D:\Users\haoxiang_zhang\Downloads\vadata_utils_public-main-py3-none-any.whl --no-deps
```

```
pip install "D:\path\to\vadata_utils_public-main-py3-none-any.whl" --no-deps
```



AI 打码结果同步job：

http://zeus.bi.ctripcorp.com/#/job/1460475  common_codes

http://zeus.bi.ctripcorp.com/#/job/1428341 common_task_ai_resultsave

```
python manage.py runserver 12345
```



### 二、同步链路

#### 4.1 hive2Mysl

datax

NPS调研：电话客服完成NPS回访后，系统生成结构化答案和VOC客户原声，需同步到分析系统进行人工编码或自动分析

```
select 
  a.import_task_id,
  a.phcall_projectid,
  a.project_name,
  a.answers,
  a.vocs,
  a.datachange_lasttime,
  a.datachange_lastby,
  a.error_reason,
  a.is_delete 
from qa_db.adm_tel_answer_final_nps_output as a 
left join qa_db.adm_tel_answer_final_nps_output as b 
  on a.import_task_id = b.import_task_id 
  and b.d = '${zdt.addDay(-1).format("yyyy-MM-dd")}'   -- ← b 是“昨天”的数据
where 
  a.d = '${zdt.format("yyyy-MM-dd")}'                -- ← a 是“今天”的数据
  and (a.project_name like '%2022Q2%' or ...)        -- ← 项目白名单
  and (
    (a.answers <> b.answers or a.vocs <> b.vocs or a.is_delete <> b.is_delete) 
    or 
    b.import_task_id is null                         -- ← 关键！
  )
```

找出今天a相比昨天b有变化或今天新增记录，用于增量





#### 4.2 python定时job

功能: 从common_import_task表读取数据，解析并导入到NPS分析系统的各个表中

机器发布后还需要在服务器中开启定时任务scheduleJob.py



importData逻辑：import_data_job--IpollTaskToNPSTargetPlatformRMS.run

从iPoll系统读取任务并写入Mysql数据库

方法本质是import_task

```
import_task 表 → 数据转换 → projects/tasks/answersheets/vocs 表 → 删除原始记录
```

数据流向: common_import_task → 项目表/任务表/答案表/VOC表等



**importData**

执行时机：每半小时

1、加载import_task近20w数据

2、拿import_task_id和common_tasks第一条匹配 如果发现tasks里存在或是自身被delete，将其逻辑删除，并返回import_task_id+false 

否则开始处理projectid，如果projectid存在 通过projectid填充字段信息，否则通过projectname解析得到项目信息

3、接着处理answer答案

answer是一个json字符串，解析后，可以看到这里会把answer里value为空的过滤掉，



如果bu为trip.com，要在回答里找出locale



{"uid":"_tihk20lcl6xd97oi","ipoll_id":"192883755","voc_emp_id":"TR052350","voc_emp_name":"Liangsu Xie （谢良素）","voc_createtime":"2025-12-22 15:13:39","rec_no":"07438544652134400182","customer_name":"Cheung/Ching Yi","oid":"1359040990760057","start_date":"2025-12-11","end_date":"2025-12-17","addl_1":"2025-12-11~2025-12-17","addl_2":"香港铜锣湾皇悦酒店(Empire Hotel Hong Kong - Causeway Bay)","addl_3":"香港","addl_4":"2025-12-11 00:00:00","addl_5":"成交","addl_6":"zh-hk","addl_7":"","addl_8":"4","addl_9":"","addl_10":"","ansersht_remark":"","locale":"zh-hk","pra_date":""}



get() returned more than one Locale_weights -- it returned 2!



4、然后处理voc

最后组装出来一个模版后 执行task_to_mysql，执行成功后会把import task删掉





**任务明细数据同步**

00:05执行：任务明细数据同步run_threading_job

过滤：所有进行中的项目，每天更新

进行task的同步

具体如何进行：

```
read_task_data_format
```







#### 4.3 计算nps结果

每天21:00计算



怎么接入clogmanager





demo:

```
ClogManager.warn("localeWeightsInspection",
                        f"Processed duplicate group: {duplicate_count} records, kept: {keep_record.locale_weight_id}, deleted: {delete_count}")

ClogManager.error("localeWeightsInspection", error_msg)

ClogManager.info("localeWeightsInspection",
            f"Inspection completed: processed={report['processed_count']}, "
            f"deleted={report['deleted_count']}, errors={report['error_count']}, warnings={report['warning_count']}")

```

















### 三、DB取数解读

**Hive表结构：**

![image-20260310155010825](D:\编程文档\note-share\项目类\assets\image-20260310155010825.png)

**Mysql 表结构解读**

common_resultsave_indicators_v1

common_resultsave_indicators_v2

这个是存档 项目有哪些指标 v2是把三层都平铺 v1是一个一个来





















### 四、项目模块解读

**整体理解**

本质是hive to mysql 放进common_import_task里面

nps后端去消费common_import_task里的task，拆成具体的表

支持人工操作，操作后得到task_details

这个task_detail即对外提供的数据

















**路由模块**

```
NPSTargetPlatformRMS/urls 主路由配置文件 汇总所有子应用路由（每个模块下有urls

```





**定时job**

![image-20260204170201360](D:\编程文档\note-share\项目类\assets\image-20260204170201360.png)



1、importData job：

日志写clog 用ClogManager

importData会把import task拆出来五张表：

- project
- 



2、计算nps job

项目的状态：

```
PROJECT_STATUS_CHOICES = (
    ('1', '正常'),
    ('2', '已锁定'),
    ('3', '已删除'),
)
```



项目筛选逻辑：

```
SELECT DISTINCT p.project_id, p.biz_id_id, b.bu_id_id
FROM common_projects p
LEFT JOIN common_biztypes b ON p.biz_id_id = b.biz_id
WHERE NOT (
    b.bu_id_id IN (1, 11, 12, 13, 15)
    OR p.biz_id_id IN (49, 52)
    OR p.project_id IN (1, 13)
)
  AND p.is_delete = 0
  AND p.project_status = 1
  
```

后续nps计算都是项目维度：对于每一个项目都会去跑calculate_nps

两步：季度汇总和月汇总

```
nps_select_by_q2_by_quarter

计算逻辑：
FROM common_projects p
JOIN common_tasks t ON p.project_id = t.project_id
JOIN common_task_vocs v ON t.task_id = v.task_id
WHERE p.project_id = 3
  AND t.is_delete = 0
  AND t.status IN (1, 2, 3)
  AND v.question_id = 2  -- Q2: nps分值


聚合逻辑：
SELECT 
    p.project_id,
    p.project_status,
    p.config_project,
    p.biz_id,
    lw.weight,
    bt.biz_name,
    bu.bucode,
    bu.bu,
    p.project_year,
    p.project_quarter,
    -- 推荐者数量（9-10分）
    COUNT(DISTINCT CASE 
        WHEN v.voc IN (9, 10) THEN t.task_id 
    END) AS promoter_num,
    -- 诋毁者数量（0-6分）
    COUNT(DISTINCT CASE 
        WHEN v.voc IN (0, 1, 2, 3, 4, 5, 6) THEN t.task_id 
    END) AS detractor_num,
    -- 被动者数量（7-8分）
    COUNT(DISTINCT CASE 
        WHEN v.voc IN (7, 8) THEN t.task_id 
    END) AS passive_num,
    -- 总样本量
    COUNT(DISTINCT CASE 
        WHEN v.voc IN (0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10) THEN t.task_id 
    END) AS all_num
FROM common_projects p
JOIN common_tasks t ON p.project_id = t.project_id
JOIN common_task_vocs v ON t.task_id = v.task_id
JOIN common_biztypes bt ON p.biz_id = bt.biz_id
JOIN bu_lists bu ON bt.bu_id = bu.bu_id
LEFT JOIN common_locale_weights lw ON t.locale_weight_id = lw.weight_id
WHERE p.project_id = 3
  AND t.is_delete = 0
  AND t.status IN (1, 2, 3)
  AND v.question_id = 2
GROUP BY 
    p.project_id, p.project_status, p.config_project,
    p.biz_id, bt.biz_name, bu.bucode, bu.bu, lw.weight,
    p.project_year, p.project_quarter

聚合结果：

<QuerySet [{'project_id': 3, 'project_status': '1', 'config_project': None, 'biz_id': 30, 'project_year': '2022', 'project_quarter': 'Q1', 'biz_name': '拿去花', 'weight': '1', 'bucode': 7, 'bu': '金融', 'promoter_num': 424, 'detractor_num': 68, 'passive_num': 251, 'all_num': 743}]>

然后算nps：


存入db：

[{'all_num': 961, 'biz_id': 29, 'biz_name': '信用贷', 'bu': '金融', 'bucode': 7, 'config_project': None, 'detractor_num': 206, 'detractor_rate': Decimal('0.21436004'), 'nps': Decimal('0.24453694'), 'passive_num': 314, 'passive_rate': Decimal('0.32674298'), 'project_id': 304, 'project_quarter': None, 'project_status': 'inprogress', 'project_year': None, 'promoter_num': 441, 'promoter_rate': Decimal('0.45889698'), 'time_dimension': 1}]


```



月汇总：

nps_select_by_q2_by_month





```
download_task_file

构建任务


```

![image-20260205180208332](D:\编程文档\note-share\项目类\assets\image-20260205180208332.png)

![image-20260205180223706](D:\编程文档\note-share\项目类\assets\image-20260205180223706.png)



**与前端交互**

```
NPSTargetPlatformRMS/views.py 模块

在urls里可以看到前端静态路由到templates/index.html里面 

index.html引用EROS，版本从qconfig的index.json里面读取
```





#### **4.1锁定某项目**

找归档版本resultsave_indicators_v1



#### 4.2归档某项目

首先获取项目id、员工号、文件id

然后保存到四张表























46236















### 五、Bug修复

看log最近有两个主流程问题

一共是扩uid即可

还有一个很特殊

然后我们要看task_details哪些地方写入：run_threading_job 这有个定时hob，每天凌晨去跑这个job



排查iq看板 NPS-独立出游-国内-2025Q47 projectid：4173



4376 码值消失





**码表替换**







### 六、测试环境如何构建数据

补充project-->补充tasks->补充answerSheet-->补充voc--->补充codes

4082

4362这个应该是有两个任务关联不上：

6980	20	en-sg	32419	1.016377921157741	ipoll	2025-09-24 09:30:03.607351	2025-12-22 14:02:50.636448	TR049110	4362
7169	13	ko-kr	41799	1.3104531517465812	ipoll	2025-09-24 10:30:01.568131	2025-12-22 14:02:50.645888	TR049110	4362
7505	14	zh-hk	32104	0.7548766953217584	ipoll	2025-10-01 09:30:07.199231	2025-12-22 14:02:50.655857	TR049110	4362

项目4082：

问题task：4913623 问题ind_id 536







5027







### 七、飞书机器人开发

appid：cli_a9986f13b3da100e

app secret：gkT8DVRBu2el0svzYAk1TeNTyhwQMUjO



原nps：直接对接TP消息

1、发送文本消息

```
imPublicSendTextMessage
```



2、发送富文本消息

```
imPublicSendRichTextMessage
```





**飞书调试台**

普通文本

{

 "receive_id": "tr029836",

 "msg_type": "text",

 "content": "{\"text\":\"test content\"}",

 "uuid": "a0d69e20-1dd1-458b-k535-dfeca4015204"

}



富文本



如何构建消息文本内容https://open.larkenterprise.com/document/server-docs/im-v1/message-content-description/create_json



催发：重要！邀您参加携程档案部满意度调研！
本次调研由携程集团服务研发中心牵头，旨在了解携程员工对档案部的满意度感受，诚邀您作为员工代表，分享您对档案部服务水平的体验。
[点击此处（请用Chrome浏览器打开）](https://trippoll.ctrip-it.com/trippollweb/newpollanswer?surveygUID=5e8495e6-8810-4b9c-8c39-0bae6374a82b&locale=zh-cn&needlogin=1&bacth=)感谢您的参与~
如果您对本次调研有任何疑问，或是需要提供任何支持，请回复邮箱：调研及客户知识管理中心 diaoyan@trip.com



这里有个技术点：

nps之前访问TP是在内网，但是飞书的服务在公网，生产上服务访问不到



走https: 二级代理

https://trip.larkenterprise.com/wiki/MNAGwmVasiFdinkNwIEcQ8knnvg



http://proxygate2.xxx.com:8080 这样一个二级代理地址



https://open.feishu.cn/open-apis/im/v1/messages





#### 飞书开发文档



1、构建client

```
import lark_oapi as lark

client = lark.Client.builder() \
    .app_id("APP_ID") \
    .app_secret("APP_SECRET") \
    .build()
```



2、构造api请求

先根据接口文档接口的URL 去引入对应的包







重要！邀您参加携程技术中心-AI研发部满意度调研！
您好，诚邀您参加携程技术中心-AI研发部满意度调研。本次调研由携程集团服务研发中心牵头，旨在了解携程员工对AI研发部的满意度感受，诚邀您作为员工代表，分享您对AI研发部服务水平的体验。
[点击此处（请用Chrome浏览器打开）](https://trippoll.ctrip-it.com/trippollweb/newpollanswer?surveygUID=54622126-58b6-4307-8bac-2cffd1698413&locale=zh-cn&needlogin=1)感谢您的参与~
如果您对本次调研有任何疑问，或是需要提供任何支持，请回复邮箱：调研及客户知识管理中心 diaoyan@trip.com



携程调研-技术中心-平台研发中心

重要！邀您参加携程技术中心-平台研发中心满意度调研！ 您好，诚邀您参加携程技术中心-平台研发中心满意度调研。本次调研由携程集团服务研发中心牵头，旨在了解携程员工对平台研发中心的满意度感受，诚邀您作为员工代表，分享您对平台研发中心服务水平的体验。 点击此处（请用Chrome浏览器打开）感谢您的参与~ 如果您对本次调研有任何疑问，或是需要提供任何支持，请回复邮箱：调研及客户知识管理中心 diaoyan@trip.com





6153

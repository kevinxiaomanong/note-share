### 一、读懂执行计划

优化sql第一步应该是读懂SQL的执行计划

SQL执行计划是指一条SQL语句在经过MySQL查询优化器后具体执行的方式

通过执行计划我们可以了解到数据表的查询顺序、操作类型、索引命中、每个数据表多少行机会查询等信息

通过EXPLAIN + select查询语句 来获取执行计划

然后各个字段含义如下：

https://javaguide.cn/database/mysql/mysql-query-execution-plan.html#table



### 二、优化案例

EXPLAIN select distinct t.*
from di_dna_user_portrait_requisition t
left join di_dna_user_portrait_requisition_subs s on t.seq_no = s.seq_no
join di_dna_sys_user u on u.user_name =('pan.wu') and (s.user_name = u.user_name or t.owner = u.user_name)
left join di_dna_user_portrait_requisition_label p on p.seq_no = t.seq_no
join di_dna_user_portrait_label q on q.label_id = p.label_id
where t.req_type IN('applyLabel', 'removeLabel') AND (t.seq_no LIKE ('%%') OR t.bu LIKE ('%%') OR t.scene LIKE ('%%')
OR t.app_effect LIKE ('%%') OR t.app_id LIKE ('%%') OR t.app_name LIKE ('%%') OR p.label_id LIKE ('%%')
OR q.name LIKE ('%%')OR u.user_name LIKE ('%%') OR u.display_name LIKE ('%%') )
order by t.datachange_createtime desc limit 10;







### 三、索引规范




























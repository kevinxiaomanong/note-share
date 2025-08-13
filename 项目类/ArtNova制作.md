### 一、背景

已有三个sql 来进行nova仪表盘的制作

```

select 
    substr(t1.order_time, 1, 7) as biz_month
    ,t3.aprovincename
    ,count(distinct coalesce(t1.primary_order_id, t1.order_id)) as order_cnt
from ctripdi_prodb.edw_ord_flt_df t1
left join (
    select
        orderid
        ,acountryname
        ,aprovincename
        ,row_number() over(partition by orderid order by case when flightclass='N' and sequence = 1 then -1 when flightclass = 'I' and segmentno = 1 then sequence * -1 else sequence end) as rk
    from flt_bidb.dw_factfltsegment
) t3
on t1.order_id = t3.orderid
and t3.rk = 1
left join ctripdi_prodb.v_edw_ord_flt_df t4
on t1.order_id = t4.orderid
left join dw_engdb.fact_ibu_order t5
on t1.order_id = t5.orderid
and t5.d = '2025-07-27'
where t1.d = '2025-07-27'
and substr(t1.order_time, 1, 10) between '2025-05-01' and '2025-06-30'
and t3.acountryname = '土耳其'
and t5.orderid is null
and t1.is_corp = 0  -- 非商旅订单
and t1.is_vac = 0  -- 非度假订单
and t1.is_trans2b = 'F'  -- 剔除移花接木订单
and t1.is_postpone_fee = 'F' -- 剔除延迟出行订单
and t1.is_redundant_order = 0  -- 剔除冗余订单
and t1.order_type_extra<>'机酒' 
and not (t1.order_status = 'C' and t4.userpaydate is null) -- 剔除提交未支付订单
group by
    substr(t1.order_time, 1, 7)
    ,t3.aprovincename
;


-- 酒店
select 
substr(t1.order_date,1,7) as biz_month
,count(*) as order_cnt
from ctripdi_prodb.edw_ord_htl_df t1
left join dwhtl.edw_htl_ord_flag t2
on t1.order_id = t2.orderid
and t2.d = '2025-07-27'
where t1.d = '2025-07-27'
and order_date between '2025-05-01' and '2025-06-30'
and t1.arrive_country_name = '土耳其'
and t1.is_ibu = 0
and t1.sub_order_type=0 
and t2.is_grouporder = 'F'
group by substr(t1.order_date,1,7)
;


-- 参展大屏 出境热门国家排名
select count(distinct concat(cast(coalesce(primaryorderid, orderid) as string), passengername))  AS persons --人次
    ,case  
        when primary_acityid=59 then '澳门'  
        when primary_acityid=58 then '香港' 
        when primary_aprovincename like '%台湾%' then '台湾'
        else primary_acountryname end as locationname    
from flt_bidb.edw_flt_order_passenger_profile
where d='2025-07-28' and substr(takeofftime, 1, 7) = '2025-06'
and passenger_card_countryname = '中国'
and primary_acountryname <> '中国'
and orderstatus in ('S','T')
group by case  
        when primary_acityid=59 then '澳门'  
        when primary_acityid=58 then '香港' 
        when primary_aprovincename like '%台湾%' then '台湾'
        else primary_acountryname end
;


```










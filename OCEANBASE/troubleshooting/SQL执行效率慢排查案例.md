# SQL执行效率慢排查案例

## 问题现象

开发人员反馈某接口压测显示某sql执行效率慢，单次执行需要10-20s

sql文本如下:

```sql
select aa.aae003 aae041,
       aa.aae003 aae042,
       aa.aae140 ,
       aa.jfrlx,
       aa.aae795 aae180,
       aa.aae082,
       case when ts=1 then least(aab191,aab150) else '' end aab191,
       case when aa.jfrlx='1' then '' else bb.aab999 end aab001,
       case when aa.jfrlx='1' then '' else bb.aab004 end aab004,
       case when ((aae202=0 and aae082=0) or aae792='05') then '退账' when ts=1 and aae202>1 then t.aaa103 else decode(aaa115,'31','补缴','') end aae013
  from (select a.aae003 ,
               a.aae140 ,
               SUM(A.AAE202) aae202,
               sum(case when a.aaa115 in ('21','35') then 0 else a.aae795 end) aae795,
               sum(a.aae082) aae082,
               count(1) ts,
               min(case when a.aaa115 in ('31','10') and a.aae082<>0 then to_char(aab191) else '88888888' end) aab191,
               max(to_char(aab191)) aab150,
               substr(max(aae002||aab191 || a.aae792), 15) aae792,
               min(case when a.aaa115 in ('10','31') then a.aaa115 else '99' end) aaa115,
               min(case when (a.aaa093='1' or a.aac066='102' or a.aab001 like '3199%') then '1' else '2' end) jfrlx,
               decode(a.aab001,'319900031990001','319900031990000',a.aab001) aab001,
               min(a.aae793) aae793
          FROM imp_basicinfo.v_ac43 A, EINP_BASICINFO.AC02 B
         WHERE A.AAZ159 = B.AAZ159
           AND A.AAC001 = B.AAC001
           AND B.AAE100 = '1'
           and a.aae078 <>'2'
           AND A.AAE792 IN ('01','04','05')
           AND NOT EXISTS (( SELECT 1 FROM imp_basicinfo.v_ac43 C WHERE C.AAZ159 = A.AAZ159 AND C.AAZ686 = A.AAZ686 AND A.AAE793 = C.AAE793 AND C.AAC001 = A.AAC001 AND C.AAE792 IN ('06') AND A.aae140 = C.aae140  and  nvl(A.AAZ223,'999') = nvl(C.AAZ223,'999')))
           and a.aac001 = to_number(?)--入参 
           and a.aae140 in ('110', '120','310','210')
           and a.aae002 >= to_number(?)--入参起始年月
           and a.aae003 >= to_number(?)--入参起始年月 
           and a.aae003 < 202312--固定值
           and a.aae202 is not null
           and a.aae793 not in ('J11','J21','J40','J50','J42')--剔除未对应明细数据和预缴
           and a.aae082 <>0
           and a.aae793 not like 'Z%'
           and (a.aaa115 not in ('40','90') or (a.aaa115='90' and aae793 in ('B42',
                            'B7A',
                            'B70',
                            'B7D',
                            'B74',
                            'B76',
                            'B75',
                            'B79',
                            'B78',
                            'B77',
                            'B93',
                            'B7E',
                            'B7F',
                            'B7N')))
         GROUP BY a.AAE003,a.aae140,decode(a.aab001,'319900031990001','319900031990000',a.aab001)) aa,imp_basicinfo.ab01 bb,imp_parameter.aaa3 t
       where aa.aab001=bb.aab001
         and t.aaa100='AAE793'
         and t.aaa102=aa.aae793--对应费款所属期统模前缴费（不含预缴、征地、补足、历史信息保存）
 union 
select to_number(min(c.aae041)) aae041,
       to_number(max(c.bae042)) aae042,
       a.aae140,
       '2' jfrlx,
       sum(c.aae795) aae180,
       sum(a.aae082) aae082,
       to_char(max(a.aab191)) aab191,
       d.aab999 aab001,
       max(d.aab004) aab004,
       '预缴' aae013
FROM imp_basicinfo.v_ac43 A, EINP_BASICINFO.AC02 B,imp_basicinfo.acb5 c,imp_basicinfo.ab01 d
         WHERE A.AAZ159 = B.AAZ159
           AND A.AAC001 = B.AAC001
           AND B.AAE100 = '1'
           and a.aae078 <>'2'
           AND A.AAE792 IN ('01','04','05')
           AND NOT EXISTS (( SELECT 1 FROM imp_basicinfo.v_ac43 x WHERE x.AAZ159 = A.AAZ159 AND x.AAZ686 = A.AAZ686 AND A.AAE793 = x.AAE793 AND x.AAC001 = A.AAC001 AND x.AAE792 IN ('06') AND A.aae140 = x.aae140  and  nvl(A.AAZ223,'999') = nvl(x.AAZ223,'999')))
           and a.aac001 = to_number(?)--入参 
           and a.aae140 in ('110', '120','310','210')
           and a.aae002>=to_number(?)--入参起始年月
           and c.aae041 >=to_number(?)--入参起始年月
           and c.aae041 <=to_number(?)--入参截止年月
           and c.aae140 in ('110', '120','310','210')
           and a.aae082<>0
           and a.aaz159=c.aaz159
           and a.aac001=c.aac001
           and a.aae002=c.aae002
           and a.aab001=d.aab001
           and a.aab001=c.aab001
 group by a.aae140,a.aae002,a.aab001,d.aab999--预缴，按预缴起始年月确定范围
union
select aa.aae003 aae041,
       aa.aae003 aae042,
       aa.aae140 ,
       aa.jfrlx,
       aa.aae795 aae180,
       aa.aae082,
       case when ts=1 then least(aab191,aab150) else '' end aab191,
       case when aa.jfrlx='1' then '' else bb.aab999 end aab001,
       case when aa.jfrlx='1' then '' else bb.aab004 end aab004,
       case when aae202=0 and aae082=0 then '退账' when ts=1 and aae202>1 then t.aaa103 else decode(aaa115,'31','补缴','') end aae013
  from (select a.aae003 ,
               a.aae140 ,
               SUM(A.AAE202) aae202,
               sum(case when a.aaa115 in ('21','35') then 0 else a.aae795 end) aae795,
               sum(a.aae082) aae082,
               count(1) ts,
               min(case when a.aaa115 in ('31','10') and a.aae082<>0 then to_char(aab191) else '88888888' end) aab191,
               max(to_char(aab191)) aab150,
               substr(max(aae002||aab191 || a.aae792), 15) aae792,
               min(case when a.aaa115 in ('10','31') then a.aaa115 else '99' end) aaa115,
               min(case when (a.aaa093='1'  or a.aab001 like '3199%') then '1' else '2' end) jfrlx,
               a.aab001,
               min(a.aae793) aae793 
          FROM imp_basicinfo.v_ac43 A, EINP_BASICINFO.AC02 B
         WHERE A.AAZ159 = B.AAZ159
           AND A.AAC001 = B.AAC001
           AND B.AAE100 = '1'
           and a.aae078 <>'2'
           AND A.AAE792 IN ('01','04','05')
           AND NOT EXISTS (( SELECT 1 FROM imp_basicinfo.v_ac43 C WHERE C.AAZ159 = A.AAZ159 AND C.AAZ686 = A.AAZ686 AND A.AAE793 = C.AAE793 AND C.AAC001 = A.AAC001 AND C.AAE792 IN ('06') AND A.aae140 = C.aae140  and  nvl(A.AAZ223,'999') = nvl(C.AAZ223,'999')))
           and a.aac001 = to_number(?)--入参 
           and a.aae140 in ('110', '120','310','210')
           and a.aae002 >= 202312 --固定值
           and a.aae003 >= 202312 --固定值
           and a.aae003 <= to_number(?)--入参截止年月 
           and a.aae202 is not null
           and a.aae793 not in ('J11','J21','J50','J40')
           and a.aae793 not like 'Z%'
           and exists (select 1 from imp_basicinfo.ac43 x WHERE x.AAZ159 = a.AAZ159 AND x.AAE793 = a.AAE793 AND x.AAC001 = A.AAC001 AND x.AAE792 ='01' and nvl(a.aaz625,'x')=nvl(x.aaz625,'x') and x.aae082=a.aae082 and x.aae002=a.aae002 and x.aae003=a.aae003) 
           and a.aaa115 not in ('40','90')
           and a.aae082 <>0
         GROUP BY a.AAE003,a.aae140,a.aab001) aa,imp_basicinfo.ab01 bb,imp_parameter.aaa3 t
       where aa.aab001=bb.aab001
         and t.aaa100='AAE793'
         and t.aaa102=aa.aae793--统模后当场结算
union
select aa.aae003 aae041,
       aa.aae003 aae042,
       aa.aae140 ,
       aa.jfrlx,
       aa.aae795 aae180,
       aa.aae082,
       case when ts=1 then least(aab191,aab150) else '' end aab191,
       case when aa.jfrlx='1' then '' else bb.aab999 end aab001,
       case when aa.jfrlx='1' then '' else bb.aab004 end aab004,
       '一次性补缴' aae013
  from (select a.aae003 ,
               a.aae140 ,
               SUM(A.AAE202) aae202,
               sum(case when a.aaa115 in ('21','35') then 0 else a.aae795 end) aae795,
               sum(a.aae082) aae082,
               count(1) ts,
               min(case when a.aaa115 in ('31','10') and a.aae082<>0 then to_char(aab191) else '88888888' end) aab191,
               max(to_char(aab191)) aab150,
               substr(max(aae002||aab191 || a.aae792), 15) aae792,
               min(case when a.aaa115 in ('10','31') then a.aaa115 else '99' end) aaa115,
               min(case when (a.aaa093='1'  or a.aab001 like '3199%') then '1' else '2' end) jfrlx,
               a.aab001,
               min(a.aae793) aae793 
          FROM imp_basicinfo.v_ac43 A, EINP_BASICINFO.AC02 B
         WHERE A.AAZ159 = B.AAZ159
           AND A.AAC001 = B.AAC001
           AND B.AAE100 = '1'
           and a.aae078 <>'2'
           AND A.AAE792 IN ('01','04','05')
           AND NOT EXISTS (( SELECT 1 FROM imp_basicinfo.v_ac43 C WHERE C.AAZ159 = A.AAZ159 AND C.AAZ686 = A.AAZ686 AND A.AAE793 = C.AAE793 AND C.AAC001 = A.AAC001 AND C.AAE792 IN ('06') AND A.aae140 = C.aae140  and  nvl(A.AAZ223,'999') = nvl(C.AAZ223,'999')))
           and a.aac001 = to_number(?)--入参 
           and a.aae140 in ('110', '120','310','210')
           and a.aae002 >= to_number(?)--入参起始年月 
           and a.aae003 >= to_number(?)--入参起始年月 
           and a.aae003 <= to_number(?)--入参截止年月 
           and a.aae202 is not null
           and a.aae793 not in ('J11','J21','J50','J40')
           and a.aae793 not like 'Z%'
           and a.aaa115 in ('40','90') 
           and a.aae082 <>0
         GROUP BY a.AAE003,a.aae140,a.aab001) aa,imp_basicinfo.ab01 bb,imp_parameter.aaa3 t
       where aa.aab001=bb.aab001
         and aa.aae082>0
         and t.aaa100='AAE793'
         and t.aaa102=aa.aae793  --特殊缴费：征地、补足
```

初步查看sql文本，看到查询条件和关联条件已经有AAC001人员ID字段了，表示该sql查询结果正常来说不会特别大。

## GV$OB_SQL_AUDIT 排查

根据sql文本查看该sql在gv$ob_sql_audit中的执行情况:

```SQL
SELECT
/*+query_timeout(3600000000) */
 USEC_TO_TIME(REQUEST_TIME) AS REQUEST_TIME,
 DATE_ADD(USEC_TO_TIME(REQUEST_TIME),INTERVAL ELAPSED_TIME / 1000000 SECOND) AS EXECUTE_END_TIME,
 SVR_IP,
 QUERY_SQL,
 PARAMS_VALUE,
 PLAN_ID,
 SQL_ID,
 RET_CODE,
 ELAPSED_TIME / 1000000 AS "收到请求到结束总时间(S elapsed_time)",
 EXECUTE_TIME / 1000000 AS "SQL执行时间(S execute_time)",
 GET_PLAN_TIME / 1000000 AS "get_plan_time(S)",
 ROUND(REQUEST_MEMORY_USED / 1024 / 1024, 2) "request_memory_used(MB)",
 TRACE_ID,
 pl_trace_id,
 TENANT_NAME,
 RETRY_CNT,
 RETURN_ROWS,
 AFFECTED_ROWS,
 TABLE_SCAN,
 IS_HIT_PLAN,
 IS_BATCHED_MULTI_STMT
  FROM GV$OB_SQL_AUDIT T
 WHERE 1 = 1
   and tenant_name='oboracle_yhcs'
   and query_sql like '%特殊缴费：征地、补足%'
   AND REQUEST_TIME >= TIME_TO_USEC('2025-11-14 11:00:00') 
 ORDER BY REQUEST_TIME DESC;
```

![image-20251114115655105](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251114115655105.png)



## GV$OB_PLAN_CACHE_PLAN_STAT排查

排查sql的执行计划情况

![image-20251114124503927](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251114124503927.png)

从SQL的执行计划上来看，当前sql生成了多个执行计划，包含type=1/2的，即本地执行计划和远程执行计划

## 问题排查思路

首先，从SQL文本上来看，已经是有比较精确的条件了，进一步排查sql的执行计划，如下：

```

```


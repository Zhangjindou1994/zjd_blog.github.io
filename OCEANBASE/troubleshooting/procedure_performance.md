# OceanBase中存储过程耗时分析
**本文档针对OceanBase中存储过程耗时高的问题进行简单的探讨。不同于ORACLE中对于存储过程的标记，oracle通过sql_id和top_level_sql_id关联存储过程和其内部的SQL,而OceanBase则是通过trace_id和pl_trace_id将两者关联。** 
## 查看gv$ob_processlist  
首先通过会话视图查找存储过程运行中的SQL(根据sql文本或会话id),拿到trace_id  

    select sql_id,trace_id,info,top_info from gv$ob_processlist where info like '%%' (top_info like '%%')  (id='***')
## 查找gv$ob_sql_audit
根据上一步拿到的trace_id去
gv$ob_sql_audit中找到存储过程的pl_trace_id

    select pl_trace_id from gv$ob_sql_audit where trace_id='***'
## 分析相关sql总耗时和平均耗时
    create table sql_audit_20250403_SP_35_CREATEMONTHDYZF_NJ_all as 
    select * from  gv$ob_sql_audit where pl_trace_id='YB420A08E3B1-00062F4AE96D63E9-0-0' and query_sql is not null
    and  usec_to_time(request_time)>date_format('2025-04-03 09:00:00','%Y-%m-%d %H:%i:%f') 
    order by elapsed_time/1000 desc;


    select sql_id,query_sql,sum(elapsed_time/1000),count(*),avg(elapsed_time/1000)
    from sql_audit_20250403_SP_35_CREATEMONTHDYZF_NJ_all  where plan_id>0
    group by sql_id,query_sql order by 3 desc

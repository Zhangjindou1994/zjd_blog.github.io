# 租户合并报错排查

## OCP报错截图

![image-20260121180441524](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20260121180441524.png)

## 合并相关视图排查

```
select * from cdb_ob_major_compaction;
```

![image-20260121180518691](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20260121180518691.png)

报错信息显示checksum_error

进一步排查相关信息

```
select * from CDB_OB_TABLET_CHECKSUM_ERROR_INFO;
select * from CDB_OB_COLUMN_CHECKSUM_ERROR_INFO;
```

![image-20260121180632848](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20260121180632848.png)

视图查看有关索引有报错，其中，data_table_id和index_table_id对应的是dba_objects中的object_id

### 错误对象查看和报错解决

![image-20260121180718420](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20260121180718420.png)

错误索引删除并重建再尝试重新合并

```

select dbms_metadata.get_ddl('INDEX','IDX_CAS_DIG_SIGN_LOG_TIME','SBDEV') from dual;

drop index sbdev.IDX_CAS_DIG_SIGN_LOG_TIME

CREATE /*+ parallel(32) */ INDEX "SBDEV"."IDX_CAS_DIG_SIGN_LOG_TIME" on "SBDEV"."CAS_DIG_SIGN_LOG" (
 "TIME"
) GLOBAL ;

```

## 重新合并

```
alter system clear merge error;
alter system resume merge;
```

问题得到解决
---
number headings: auto, first-level 4, max 6, contents ^toc, start-at 1, _.1.1
dg-publish: false
---


### 文档名称
该文档以 V4.2.5 版本的官方文档为基础；

#### 1 适用版本
V4.2.5 版本

#### 2 问题现象
*move_pq_ywzt_240925#yzw_move_24092505* 租户合并出错，同是该租户所在的 OBServer 也有日志告警：`ERROR [SHARE] check_global_index_column_checksum`，详细告警信息如下：

```log
[2025-05-16 14:35:24.819842] ERROR [SHARE] check_global_index_column_checksum (ob_tablet_replica_checksum_operator.cpp:1143) [118166][T1006_MergeSche][T1006][YB42C00A05E6-0006266540A08DA7-0-0]
  [lt=13][errcode=-4103] Data checksum error(
    msg="data table and global index table column checksum are not equal", ret=-4103, ret="OB_CHECKSUM_ERROR", 
    ckm_error_info={
      tenant_id:1006, frozen_scn:{val:1747332001365290000, v:0}, is_global_index:true, data_table_id:540421, index_table_id:540681, 
      data_tablet_id:{id:0}, index_tablet_id:{id:0}, column_id:16, data_column_checksum:79973309717743830, index_column_checksum:79973313358424444
    }, 
    index_ckm_tablet_cnt=1, index_schema_tablet_cnt=1, data_ckm_tablet_cnt=256, data_schema_tablet_cnt=256
  )
```


#### 3 问题排查过程
##### 3.1 检查租户合并状态
怎么可以判断是否出现了 *checksum error*；

````tab
tab: 1.x - 3.x

```sql
select * from _all_zone;
```

tab: 4.x
查询【业务租户】的合并状态，发现整个租户合并状态为 *COMPACTION*【正在合并】，合并信息为：`CHECKSUM_ERROR`；
```sql
select * from DBA_OB_TENANTS where TENANT_NAME = 'move_pq_ywzt_240925'
    -- 1006 move_pq_ywzt_240925

select * from oceanbase.CDB_OB_MAJOR_COMPACTION where tenant_id = 1006;

select * from oceanbase.CDB_ob_zone_MAJOR_COMPACTION where tenant_id = 1006;
```
````
更多信息：[[查看 OB 合并信息_v4.x 版本]]，；
租户合并报错信息为：*CHECKSUM_ERROR*，说明租户在合并检查 *CHECKSUM* 出错，一般是【主表】【索引表】的 *checksum* 不一致；


##### 3.2 检查 CHECKSUM 信息
具体来说，可以从 `CDB_OB_TABLET_CHECKSUM_ERROR_INFO` 和 `CDB_OB_COLUMN_CHECKSUM_ERROR_INFO` 两个视图中查看存在 *checksum error* 的 *tablet* 或 *table*，然后进一步排查导致 *checksum error* 的原因；

从 V4.0.0 版本开始引入 `CDB_OB_TABLET_CHECKSUM_ERROR_INFO` 视图，展示 Tablet 副本之间出现的数据不一致的信息；该视图的数据来源：`__all_virtual_tablet_replica_checksum` 内部表；
```sql
select * from oceanbase.CDB_OB_TABLET_CHECKSUM_ERROR_INFO;    -- 查询为空
/*
CDB_OB_TABLET_CHECKSUM_ERROR_INFO 视图字段介绍：
	TENANT_ID：bigint(20)，NO，租户 ID；
	TABLET_ID：bigint(20)，NO，出现校验和不一致的 TABLET ID；
*/
```

```sql
select * from __all_virtual_tablet_replica_checksum where tenant_id = 1006 and tablet_id in (217437,1152921504606881078);
```

从 V4.0.0 版本开始引入 `CDB_OB_COLUMN_CHECKSUM_ERROR_INFO` 视图，展示索引表（包括：全局索引和局部索引）与主表之间出现的列校验和不一致的信息；该视图的数据来源：`__ALL_VIRTUAL_COLUMN_CHECKSUM_ERROR_INFO` 内部表；
```sql
-- 
SELECT * FROM oceanbase.CDB_OB_COLUMN_CHECKSUM_ERROR_INFO WHERE TENANT_ID = 1006;
SELECT * FROM oceanbase.CDB_OB_COLUMN_CHECKSUM_ERROR_INFO WHERE TENANT_ID = 1006 AND DATA_TABLE_ID = ;
```

```log
oceanbase.CDB_OB_COLUMN_CHECKSUM_ERROR_INFO 字段介绍：
	TENANT_ID：租户 ID。
	FROZEN_SCN：当前合并的 SCN。
	INDEX_TYPE：索引表类型（0 表示局部索引表，1 表示全局索引表）；
		在 v4.2.4 版本中，该字段为 'LOCAL_INDEX'，但是该索引的类型为 global；
	DATA_TABLE_ID / INDEX_TABLE_ID：主表和索引表的 ID；
	DATA_TABLET_ID / INDEX_TABLET_ID：主表和索引表的 Tablet ID；
	COLUMN_ID：出现校验和错误的列 ID;
	DATA_COLUMN_CHECKSUM：主表对应的列校验和值；
	INDEX_COLUMN_CHECKSUM：索引表对应的列校验和值；
```


##### 3.3 获取表或索引信息
通过上述获取的 *tenant_id*，*table_id* 查询【主表】【索引表】checkersum不一致的表信息，主要是 *database_id* ，在通过 *database_id*  获取数据库名称；

```sql
-- 通过 tenant_id，table_id 确定表名及表所在的数据库
select * from oceanbase.__all_virtual_table where tenant_id = 1006 and table_id = 540421; 

-- 通过 database_id 确认 数据库名称
select * from OCEANBASE.__ALL_VIRTUAL_DATABASE where tenant_id = 1006 and database_id = 500002;
    -- pqzzx
```

通过 *table_id* 查询表【包括索引表】的信息，主要是索引名称；
```sql
-- 通过 table_id 查询表明或索引名称
select * from oceanbase.__all_virtual_table where table_id in (541460,540681,540683,540684);
```


```log
【table_name】：【表名称】或【索引名称】；
		若为索引时，则该字段的值格式为：__idx_ + 表的id + 索引名称；
```


##### 3.4 判断是否卡在对所有 table 进行 checksum 校验阶段
本节介绍判断合并是否卡在对所有 *table* 进行 *checksum* 校验阶段的方法，包括【查看关键日志】和【查看堆栈信息】2 种方法；

#### 4 方法一：查看关键日志

与 *判断是否卡在检查 tablet 版本号阶段中的方法二查看关键日志* 一节中介绍的方法类似，先找到租户 1 号日志流 Leader 所在的机器。然后，登录对应机器，并进入对应日志目录后，执行以下命令，查看是否存在 `exists unverified tables...` 相关日志。若存在，则表明依然存在 table 尚未完成 `checksum` 校验。如图6所示，1004 租户存在 506674 号 table 和 736983 号 table 尚未完成 `checksum` 校验。

```shell
# 将 yyy 替换为对应的时间，将 xxx 替换为对应租户的 ID 号。
grep "exists unverified tables" rootservice.log.yyy | grep Txxx
```

![1678454366931-7a396291-b013-4d95-9a7c-21ae0aeeb8e4.png](https://obbusiness-private.oss-cn-shanghai.aliyuncs.com/doc/img/knowledge-base/database/sql/20240305rssidemergestuck006.png)

图 6，exists unverified tables 日志。

当存在尚未完成 `checksum` 校验的 table 时，需要进一步排查未完成 `checksum` 校验的原因。一方面，可以根据 trace_id 继续搜索相关日志，看是否是由于其它报错引起的。另一方面，需要基于对源码细节的了解，思考是否是 `checksum` 校验的代码逻辑存在漏洞。

#### 5 方法二：查看堆栈信息

若上述方法找不到任何相关的信息，则可以查看堆栈信息，具体方法同 **判断是否卡在检查 tablet 版本号阶段中的方法三查看堆栈信息** 一节。若从堆栈信息中可以看到 `ObChecksumValidatorBase::validate_checksum`，则表明当前处于对所有 table 进行 `checksum` 校验阶段。注意，可反复多查看几次堆栈信息。若长时间处于对所有 table 进行 `checksum` 校验阶段，则需进一步排查原因（例如，是否 `tablet` 数量太多、是否存在慢 `SQL` 等）



#### 6 解决方式
##### 6.1 checksum 概述
*什么是 checksum*：
`data checksum`：一个 SSTable 中所有宏块内存二进制计算出来的 checksum 值。反映了宏块中的数据和数据分布情况。如果宏块中数据一致但是数据分布不一致，计算出来的checksum也不相等；
`column checksum`：SSTable 中所有行中相同列计算出来的 checksum 值，假设这个表共有 3 列，则会有 3 个 column checksum 值与每个列对应；

`checksum 什么时候校验`：在合并之后，RS 会进行 checksum 校验，主要校验 2 种情况：【*副本间 data checksum*】，【*主表，索引表 column checksum*】；

*副本间 data checksum*：校验一个分区多个副本，相同 snapshot 的 Major SSTable 的 data checksum，应该相等；*主表，索引表 column checksum*：校验主表和索引表，相同 snapshot 的 Major SSTable，相同列的 column checksum，应该相等；

如果任一分区出现了checksum不一致的情况，就会报错 *CHECKSUM ERROR*；


##### 6.2 临时处理方法
如果合并长时间未结束，可能出现合并卡住的情况。长时间合并未结束可能由 `checksum error` 或合并暂停导致。在这种情况下，需要解决 `checksum error` 问题或通过执行 `alter system resume merge;` 命令恢复合并；

> [!NOTE] 应急恢复手段：
> 1. 如果是主表，索引表 Checksum 不一致，可以删除索引表；
> 2. 副本间 Checksum 不一致，删少数派分区/删表；

【2025-05-16】【南中心】【合并出错】【*CHECKSUM ERROR*】经过问题排查发现为【主表，索引表】Checksum 不一致，所以将索引删除重建；

````tab
tab:MySQL 模式
```sql
-- 重建索引
-- __idx_540421_ind_arrange_total_fydmOrktsj
    -- KEY `ind_arrange_total_fydmOrktsj` (`FYDM`, `KTSJ`) BLOCK_SIZE 16384 GLOBAL,
    drop index ind_arrange_total_fydmOrktsj on pqzzx.szft_arrange_total;
    create /*+ parallel(16)*/ index ind_arrange_total_fydmOrktsj on pqzzx.szft_arrange_total(FYDM,KTSJ) GLOBAL;
```

tab:Oracle 模式
```sql
DROP INDEX idx_test_tbl1_col2;  -- 删除索引
CREATE INDEX /*+ PARALLEL(4) */ idx_test_tbl1_col2 ON test_tbl2(col2); -- 创建索引
```
````

关于更多删除索引信息：[[OB 索引操作_禁用，删除]]，；

清理租户合并报错，并继续合并；
```sql
-- 清理报错的合并状态，让 RS 继续推进合并
alter system clear merge error;
```


### 参考文档
1. [合并错误，错误代码 4103](https://www.oceanbase.com/knowledge-base/oceanbase-database-20000000083?back=kb)，影响版本：OceanBase 数据库 V2.2.30-BP10 以前的版本；
2. [purge 并发引发的副本 checksum 数据不一致](https://www.oceanbase.com/knowledge-base/oceanbase-database-1000000000008209)，；
3. [checksum error ret=-4103](https://www.oceanbase.com/knowledge-base/oceanbase-database-1000000000209925)，；



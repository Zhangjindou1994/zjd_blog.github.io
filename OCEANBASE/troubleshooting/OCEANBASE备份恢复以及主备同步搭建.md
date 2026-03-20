# OCEANBASE备份恢复/主备同步搭建/租户克隆实践

[TOC]

## 物理备份及逻辑备份

### 物理备份

​	OceanBase 数据库当前版本支持阿里云 OSS、NFS、Azure Blob、AWS S3 以及兼容 S3 协议的对象存储（例如：华为 OBS、Google GCS、腾讯云 COS） 等备份介质，部分备份介质需要满足一些基本要求才能使用。本文档基于NFS进行相关测试工作。

#### 备份前提

##### 日志归档

**&emsp;&emsp;配置归档并发度(可选)**

**&emsp;&emsp;设置方法:**

**&emsp;&emsp;sys租户调整指定租户或所有租户**

```sql
ALTER SYSTEM SET log_archive_concurrency = 10 tenant=[mysql_tenant|all_user||all];
```

**&emsp;&emsp;用户租户调整本租户的日志归档并发度**

```sql
ALTER SYSTEM SET log_archive_concurrency = 10;
```

**&emsp;&emsp;配置归档目的端(以NFS为例)**

&emsp;&emsp;sys**租户为指定租户配置**

```sql
ALTER SYSTEM SET LOG_ARCHIVE_DEST='LOCATION=file:///data/nfs/backup/archive' TENANT = mysql_tenant;
```

**&emsp;&emsp;用户租户为本租户指定**

```sql
ALTER SYSTEM SET LOG_ARCHIVE_DEST='LOCATION=file:///data/nfs/backup/archive';
```

​		**配置归档延迟(可选)**

**&emsp;&emsp;sys租户为指定租户配置归档延迟时间**

```sql
ALTER SYSTEM SET archive_lag_target = '120s' TENANT = mysql_tenant;
```

**&emsp;&emsp;用户租户为本租户配置归档延迟时间**

```sql
ALTER SYSTEM SET archive_lag_target = '120s';
```

&emsp;&emsp;开启归档

```sql
ALTER SYSTEM ARCHIVELOG TENANT = mysql_tenant;
```

#### 数据备份

&emsp;&emsp;设置备份目标端(本文以NFS为例，其余介质下请参考官方文档)

&emsp;&emsp;系统租户为指定租户设置

```sql
ALTER SYSTEM SET DATA_BACKUP_DEST= 'file:///data/nfs/backup/data' TENANT = mysql_tenant;
```

&emsp;&emsp;用户租户设置

```
ALTER SYSTEM SET DATA_BACKUP_DEST='file:///data/nfs/backup/data';
```

&emsp;&emsp;设置备份并发度

```
ALTER SYSTEM SET ha_low_thread_score = 10 TENANT = [all_user|all];
```

&emsp;&emsp;系统租户中执行全量备份

```sql
全部租户:
ALTER SYSTEM BACKUP DATABASE [PLUS ARCHIVELOG] ;
指定租户:
ALTER SYSTEM BACKUP DATABASE  tenant=mysql_tenant;
```

&emsp;&emsp;用户租户发起全量备份

```
ALTER SYSTEM BACKUP DATABASE [PLUS ARCHIVELOG];
```

&emsp;&emsp;系统租户发起增量备份

```sql
全部租户:
ALTER SYSTEM BACKUP INCREMENTAL  DATABASE  ;
指定租户:
ALTER SYSTEM BACKUP INCREMENTAL  tenant=mysql_tenant;
```

&emsp;&emsp;用户租户发起增量备份

```sql
ALTER SYSTEM BACKUP INCREMENTAL DATABASE;
```

### 备份进度及结果查看

```
系统租户查看:
SELECT * FROM oceanbase.CDB_OB_BACKUP_JOBS\G
SELECT * FROM oceanbase.CDB_OB_BACKUP_TASKS\G
业务租户查看:
SELECT * FROM SYS.DBA_OB_BACKUP_JOBS\G
SELECT * FROM SYS.DBA_OB_BACKUP_TASKS\G

结果查看:
系统租户查看:
SELECT * FROM oceanbase.CDB_OB_BACKUP_JOB_HISTORY\G
SELECT * FROM oceanbase.CDB_OB_BACKUP_TASK_HISTORY\G
备份集信息:
SELECT * FROM oceanbase.CDB_OB_BACKUP_SET_FILES WHERE TENANT_ID = 1002 AND BACKUP_SET_ID = 1\G
业务租户查看:
SELECT * FROM SYS.DBA_OB_BACKUP_TASK_HISTORY\G
SELECT * FROM oceanbase.DBA_OB_BACKUP_TASK_HISTORY\G
```



### 逻辑备份

oceanbase使用obdumper/obloader工具进行逻辑备份。



## 备库搭建

### ocp白屏部署

创建备份策略立即执行一次全量备份

![image-20251023180731813](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251023180731813.png)

创建备租户

![image-20251023180835429](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251023180835429.png)

![image-20251023180851628](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251023180851628.png)

该步骤当中可以选择基于网络还是基于归档日志。

### 黑屏命令行部署

#### 前置步骤

&emsp;完成一次主租户或主租户的其他备租户的全量备份

#### 重要参数

archive_log_target 

`archive_lag_target` 用于控制租户日志归档延迟的时间。默认值为120s

- 对于基于日志归档的物理备库场景，可以根据备租户的同步要求，调整配置项 `archive_lag_target` 的值，可以设置为备租户延时要求的一半。
- 对于其他场景，例如基于网络的物理备库场景，建议根据租户的写入压力以及对归档时效性的要求，调整配置项 `archive_lag_target` 的值。在时效性满足的前提下，尽量保证单个日志流在一个 `archive_lag_target` 所设置的周期内，产生不少于一个 64 MB 大小的日志文件。

#### 恢复备租户（使用数据库备份(不含归档日志)以及源端归档日志搭建并基于归档日志同步）

1. 创建unit

   ```
   CREATE RESOURCE UNIT unit1 MAX_CPU 1, MEMORY_SIZE = '5G';
   ```

2. 创建resource pool

   资源池规格尽量与源端租户保持同构

   ```
   CREATE RESOURCE POOL pool_for_standby UNIT = 'unit1', UNIT_NUM = 1, ZONE_LIST = ('zone1','zone2','zone3');
   ```

3. 执行restore命令

   源端备份时，如果没有指定plus archivelog选项，则恢复时需要同时指定数据备份目录和归档目录

   在归档目录为nfs的情况下，源端和目标端目录结构尽量保持一致

   ```
   ALTER SYSTEM RESTORE standby_tenant 
   FROM 'file:///data/1/sh_databackup,file:///data/1/sh_archive' 
   UNTIL TIME='2023-05-26 15:04:23.825558' WITH 'pool_list=pool_for_standby';
   ```

4. 开启日志持续同步

   ```
   ALTER SYSTEM RECOVER STANDBY UNTIL UNLIMITED;
   ```

#### 恢复备租户（使用数据库备份(含归档日志)以及源端归档日志搭建并基于归档日志同步）

1. 创建unit

   ```
   CREATE RESOURCE UNIT unit1 MAX_CPU 1, MEMORY_SIZE = '5G';
   ```

2. 创建resource pool

   资源池规格尽量与源端租户保持同构

   ```
   CREATE RESOURCE POOL pool_for_standby UNIT = 'unit1', UNIT_NUM = 1, ZONE_LIST = ('zone1','zone2','zone3');
   ```

3. 执行restore命令

   源端备份时，如果指定plus archivelog选项，则恢复时指定备份目录即可

   在归档目录为nfs的情况下，源端和目标端目录结构尽量保持一致

   ```
   ALTER SYSTEM RESTORE standby_tenant 
   FROM 'file:///data/1/sh_databackup' 
   UNTIL TIME='2023-05-26 15:04:23.825558' WITH 'pool_list=pool_for_standby';
   ```

4. 开启日志持续同步

   - **基于日志归档的物理备库**

     使用不包含归档日志的数据库备份恢复数据时，已经指定的日志归档目录，此时直接开启日志持续同步即可。

     使用包含归档日志的数据备份恢复数据后，需要设置归档目的端。

     ```
     系统租户设置:
     ALTER SYSTEM SET LOG_RESTORE_SOURCE ='location=file:///data/1/sh_archive' TENANT = standby_tenant;
     业务租户设置:
     ALTER SYSTEM SET LOG_RESTORE_SOURCE ='location=file:///data/1/sh_archive';
     ```

     再开启日志持续同步即可。

   - **基于网络的物理备库**

     和基于日志归档的备库搭建不同，基于网络的备库，将日志恢复源直接指向主租户或源端的备租户，步骤如下:

     1. 创建访问视图的专用用户

        ```
        create user rep_user identified by '***';
        grant standby_replication to rep_user;
        ```

     2. 获取主租户或源端备租户的访问入口信息

        ```
        SELECT * FROM DBA_OB_ACCESS_POINT;
        ```

        

     3. 检查主租户设置

        **ls_gc_delay_time默认为0，当租户开启了日志归档后，该参数不生效。**

        ```
        检查归档:
        select * from dba_ob_archivelog;
        检查ls_gc_delay_time:
        show parameters like '%ls_gc_delay_time%'
        ```

        

     4. 设置日志恢复源

        ```
        备租户系统租户设置:
        ALTER SYSTEM SET LOG_RESTORE_SOURCE ='SERVICE=$ip_list USER=$user_name@$tenant_name PASSWORD=$password' TENANT = standby_tenant_name;
        备租户业务租户设置:
        ALTER SYSTEM SET LOG_RESTORE_SOURCE ='SERVICE=$ip_list USER=$user_name@$tenant_name PASSWORD=$password';
        
        ```

        

     5. 开启日志持续同步

        ```
        ALTER SYSTEM RECOVER STANDBY UNTIL UNLIMITED；
        ```

#### 恢复备租户(执行指定路径的恢复)

添加备份集(包含backup_set和archivelog_piece)

```
ALTER SYSTEM ADD RESTORE SOURCE 'file:///obbackup/qgtcbackup/qgtc/1751349625/tenant_incarnation_1/1002/clog/piece_d1058r38p156';
ALTER SYSTEM ADD RESTORE SOURCE 'file:///obbackup/qgtcbackup/qgtc/1751349625/tenant_incarnation_1/1002/clog/piece_d1058r38p157';
ALTER SYSTEM ADD RESTORE SOURCE 'file:///obbackup/qgtcbackup/qgtc/1751349625/tenant_incarnation_1/1002/data/backup_set_118_full';
ALTER SYSTEM ADD RESTORE SOURCE 'file:///obbackup/qgtcbackup/qgtc/1751349625/tenant_incarnation_1/1002/data/backup_set_119_inc';
```

执行恢复

```

ALTER SYSTEM RESTORE oboracle_dr UNTIL TIME = '2026-03-02 04:00:00' WITH 'pool_list=pool_for_yidong';
```



#### 查看备租户类型(基于网络还是基于归档日志)

系统租户查看

```
SELECT * FROM oceanbase.CDB_OB_LOG_RESTORE_SOURCE;
```

业务租户查看

```
SELECT * FROM SYS.DBA_OB_LOG_RESTORE_SOURCE;
```

字段type值为**location**:基于归档日志

字段type值为**service**:基于网络

#### 两种同步方式的互相切换

- 取消当前同步

  ```
  ALTER SYSTEM RECOVER STANDBY CANCEL;
  ```

- 设置日志恢复源

  ```
  基于归档切换为基于网络:
  ALTER SYSTEM SET LOG_RESTORE_SOURCE ='SERVICE=$ip_list USER=$user_name@$tenant_name PASSWORD=$password';
  示例:
  ALTER SYSTEM SET LOG_RESTORE_SOURCE ='SERVICE=172.22.35.11:2881;172.22.35.3:2881;172.22.35.7:2881 USER=STANDBYRO@oboracle PASSWORD=Wonders123';
  基于网络切换为基于归档:
  ALTER SYSTEM SET LOG_RESTORE_SOURCE ='location=file:///data/1/sh_archive';
  示例：
  ALTER SYSTEM SET LOG_RESTORE_SOURCE ='location=file:///obbackup/qgtc/qgtc_arc';
  
  ```

- 开启同步

  ```
  ALTER SYSTEM RECOVER STANDBY UNTIL UNLIMITED；
  ```

  

## 租户克隆及脚本实践

租户克隆脚本

```sql
#!/bin/bash
###set -euo pipefail

########################################
# OB sys租户连接信息定义
########################################
OB_ADDR=127.0.0.1
OB_PORT=2883
OB_USER=root
OB_PASS=aaAA11@@__
OB_TENANT=sys
OB_SCHEMA=oceanbase
OB_CLUSTER=qgtc_zsc
OB_USER_TENANT=oboracle_zsc
OB_ZSC_USER=sys

########################################
# OB客户端连接参数配置
########################################
OB_CLIENT_PARAMS="-c -A -vv"

########################################
# 文件位置
########################################
LOG_FILE=/root/oboracle_zsc/oboracle_zsc_start.log
LOCK_FILE=/var/run/oboracle_zsc_start.lock

########################################
# 日志函数
########################################
logWrite() {
  echo -e "`date +'%Y-%m-%d %H:%M:%S'` \033[32m[INFO] ${1} \033[0m" >> "${LOG_FILE}"
}

########################################
# 异常退出日志
########################################
##trap 'logWrite "脚本退出，exit code=$?"' EXIT

########################################
# 统一命令执行函数（stdout + stderr 入日志）
########################################
run_cmd() {
    local desc="$1"
    local cmd="$2"

    logWrite "开始执行：$desc"
    logWrite "执行命令：$cmd"

    local tmp_log
    tmp_log=$(mktemp /tmp/obcmd.XXXXXX)

    local rc
    if eval "$cmd" >>"$tmp_log" 2>&1; then
        rc=0
    else
        rc=$?
    fi

    cat "$tmp_log" >>"$LOG_FILE"


    if [ $rc -ne 0 ]; then
        logWrite "$desc 执行失败"
        logWrite "错误信息如下："
        sed 's/^/  /' "$tmp_log" | tail -20 >>"$LOG_FILE"
    else
        logWrite "$desc 执行成功"
    fi

    rm -f "$tmp_log"
    return $rc
}

########################################
# 创建准生产租户
########################################
create_zsc_tenant() {
    cmd="obclient ${OB_CLIENT_PARAMS} \
        -h${OB_ADDR} -P${OB_PORT} \
        -u${OB_USER}@${OB_TENANT}#${OB_CLUSTER} \
        -p${OB_PASS} -D ${OB_SCHEMA} \
        -e \"CREATE TENANT oboracle_zsc FROM cs_oboracle \
             WITH RESOURCE_POOL=zsc_pool, UNIT=unit_zsc;\""

    run_cmd "克隆准生产租户 oboracle_zsc" "$cmd"
    return $?
}

########################################
# 创建租户重试机制（10分钟）
########################################
retry_create_zsc_tenant() {
    local timeout=1800
    local interval=15
    local start_ts
    start_ts=$(date +%s)
    local retry_cnt=0

    logWrite "开始执行准生产租户克隆（带重试机制）"

    while true; do
        ((retry_cnt++))

        if create_zsc_tenant; then
            logWrite "准生产租户克隆成功（第 ${retry_cnt} 次）"
            return 0
        fi

        local now_ts
        now_ts=$(date +%s)
        local elapsed=$((now_ts - start_ts))

        if [ $elapsed -ge $timeout ]; then
            logWrite "准生产租户克隆失败，已重试 ${retry_cnt} 次，超时 ${timeout}s，脚本退出"
            exit 1
        fi

        logWrite "第 ${retry_cnt} 次失败，${interval}s 后重试（已耗时 ${elapsed}s）"
        sleep $interval
    done
}

########################################
# 恢复租户
########################################
recover_zsc_tenant() {
    cmd="obclient ${OB_CLIENT_PARAMS} \
        -h${OB_ADDR} -P${OB_PORT} \
        -u${OB_USER}@${OB_TENANT}#${OB_CLUSTER} \
        -p${OB_PASS} -D ${OB_SCHEMA} \
        -e \"ALTER SYSTEM RECOVER STANDBY TENANT=oboracle_zsc UNTIL UNLIMITED;\""

    run_cmd "恢复准生产租户 oboracle_zsc" "$cmd" || exit 1
}

########################################
# 切换为 primary
########################################
switch_zsc_tenant() {
    cmd="obclient ${OB_CLIENT_PARAMS} \
        -h${OB_ADDR} -P${OB_PORT} \
        -u${OB_USER}@${OB_TENANT}#${OB_CLUSTER} \
        -p${OB_PASS} -D ${OB_SCHEMA} \
        -e \"ALTER SYSTEM SWITCHOVER TO PRIMARY TENANT=oboracle_zsc;\""

    run_cmd "切换准生产租户为 PRIMARY" "$cmd" || exit 1
}

########################################
# 执行开库脚本
########################################
post_switch_zsc_tenant() {
    cmd="obclient ${OB_CLIENT_PARAMS} \
        -h${OB_ADDR} -P${OB_PORT} \
        -u${OB_ZSC_USER}@${OB_USER_TENANT}#${OB_CLUSTER} \
        -p${OB_PASS} \
        -e \"SOURCE /root/oboracle_zsc/post_create_zsc_tenant.sql;\""

    run_cmd "执行准生产租户开库脚本" "$cmd" || exit 1
}

########################################
# 主流程
########################################
main() {
    logWrite "========== oboracle_zsc 启动流程开始 =========="

    retry_create_zsc_tenant

    recover_zsc_tenant

    logWrite "等待 300s，确保租户恢复完成"
    sleep 600

    switch_zsc_tenant

    post_switch_zsc_tenant

    logWrite "========== oboracle_zsc 启动流程完成 =========="
}

########################################
# 防重复执行（锁）
########################################
(
    flock -n 200 || {
        echo "脚本正在运行中，退出"
        exit 1
    }

    main
) 200>"${LOCK_FILE}"


```



## 实测问题

#### 问题现象

 ob 4.3.5.1版本，租户克隆时，会有概率发生CLONE_SYS_WAIT_CREATE_SNAPSHOT_FAIL报错，需要在脚本中添加重试步骤

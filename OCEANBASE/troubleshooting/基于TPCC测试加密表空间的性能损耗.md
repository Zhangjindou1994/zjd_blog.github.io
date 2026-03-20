# 基于TPCC测试加密表空间的性能损耗

## 测试环境

&emsp;&emsp;测试工具:TPCC

&emsp;&emsp;测试环境:

&emsp;&emsp;&emsp;数据库版本:OceanBase 4.3.5.1 

## 测试基准数据

&emsp;&emsp;数据量:1000仓

&emsp;&emsp;测试并发度:150|300|500

## 测试过程

**&emsp;&emsp;生成基准数据**

**&emsp;&emsp;&emsp;建表语句**

```sql
CREATE TABLE bmsql_config (
cfg_name    varchar(30) primary key,
cfg_value   varchar(50)
);

-- drop tablegroup TPCC_GROUP;
CREATE TABLEGROUP TPCC_GROUP binding true partition by hash partitions 9;

CREATE TABLE bmsql_warehouse (
   w_id        integer   not null,
   w_ytd       decimal(12,2),
   w_tax       decimal(4,4),
   w_name      varchar(10),
   w_street_1  varchar(20),
   w_street_2  varchar(20),
   w_city      varchar(20),
   w_state     char(2),
   w_zip       char(9),
   primary key(w_id)
)tablegroup='TPCC_GROUP' partition by hash(w_id) partitions 9;

CREATE TABLE bmsql_district (
   d_w_id       integer       not null,
   d_id         integer       not null,
   d_ytd        decimal(12,2),
   d_tax        decimal(4,4),
   d_next_o_id  integer,
   d_name       varchar(10),
   d_street_1   varchar(20),
   d_street_2   varchar(20),
   d_city       varchar(20),
   d_state      char(2),
   d_zip        char(9),
   PRIMARY KEY (d_w_id, d_id)
)tablegroup='TPCC_GROUP' partition by hash(d_w_id) partitions 9;

CREATE TABLE bmsql_customer (
   c_w_id         integer        not null,
   c_d_id         integer        not null,
   c_id           integer        not null,
   c_discount     decimal(4,4),
   c_credit       char(2),
   c_last         varchar(16),
   c_first        varchar(16),
   c_credit_lim   decimal(12,2),
   c_balance      decimal(12,2),
   c_ytd_payment  decimal(12,2),
   c_payment_cnt  integer,
   c_delivery_cnt integer,
   c_street_1     varchar(20),
   c_street_2     varchar(20),
   c_city         varchar(20),
   c_state        char(2),
   c_zip          char(9),
   c_phone        char(16),
   c_since        timestamp,
   c_middle       char(2),
   c_data         varchar(500),
   PRIMARY KEY (c_w_id, c_d_id, c_id)
)tablegroup='TPCC_GROUP' partition by hash(c_w_id) partitions 9;


CREATE TABLE bmsql_history (
   hist_id  integer,
   h_c_id   integer,
   h_c_d_id integer,
   h_c_w_id integer,
   h_d_id   integer,
   h_w_id   integer,
   h_date   timestamp,
   h_amount decimal(6,2),
   h_data   varchar(24)
)tablegroup='TPCC_GROUP' partition by hash(h_w_id) partitions 9;

CREATE TABLE bmsql_new_order (
   no_w_id  integer   not null ,
   no_d_id  integer   not null,
   no_o_id  integer   not null,
   PRIMARY KEY (no_w_id, no_d_id, no_o_id)
)tablegroup='TPCC_GROUP' partition by hash(no_w_id) partitions 9;

CREATE TABLE bmsql_oorder (
   o_w_id       integer      not null,
   o_d_id       integer      not null,
   o_id         integer      not null,
   o_c_id       integer,
   o_carrier_id integer,
   o_ol_cnt     integer,
   o_all_local  integer,
   o_entry_d    timestamp,
   PRIMARY KEY (o_w_id, o_d_id, o_id)
)tablegroup='TPCC_GROUP' partition by hash(o_w_id) partitions 9;

CREATE TABLE bmsql_order_line (
   ol_w_id         integer   not null,
   ol_d_id         integer   not null,
   ol_o_id         integer   not null,
   ol_number       integer   not null,
   ol_i_id         integer   not null,
   ol_delivery_d   timestamp,
   ol_amount       decimal(6,2),
   ol_supply_w_id  integer,
   ol_quantity     integer,
   ol_dist_info    char(24),
   PRIMARY KEY (ol_w_id, ol_d_id, ol_o_id, ol_number)
)tablegroup='TPCC_GROUP' partition by hash(ol_w_id) partitions 9;

CREATE TABLE bmsql_item (
   i_id     integer      not null,
   i_name   varchar(24),
   i_price  decimal(5,2),
   i_data   varchar(50),
   i_im_id  integer,
   PRIMARY KEY (i_id)
);

CREATE TABLE bmsql_stock (
   s_w_id       integer       not null,
   s_i_id       integer       not null,
   s_quantity   integer,
   s_ytd        integer,
   s_order_cnt  integer,
   s_remote_cnt integer,
   s_data       varchar(50),
   s_dist_01    char(24),
   s_dist_02    char(24),
   s_dist_03    char(24),
   s_dist_04    char(24),
   s_dist_05    char(24),
   s_dist_06    char(24),
   s_dist_07    char(24),
   s_dist_08    char(24),
   s_dist_09    char(24),
   s_dist_10    char(24),
   PRIMARY KEY (s_w_id, s_i_id)
)tablegroup='TPCC_GROUP' use_bloom_filter=true partition by hash(s_w_id) partitions 9;
```

**&emsp;&emsp;&emsp;tpcc参数文件**

```
db=oceanbase
driver=com.oceanbase.jdbc.Driver
conn=jdbc:oceanbase://***:12883/tpcc?rewriteBatchedStatements=true&useLobLocatorV2=false&useOraclePrepareExecute=true&cachePrepStmts=true&useServerPrepStmts=true&usePieceData=true&allowMultiQueries=true&useLocalSessionState=true&useUnicode=true&characterEncoding=utf-8&socketTimeout=3000000&useSSL=false
user=TPCC@oboracle#poc
password=*** 

warehouses=1000
loadWorkers=48

terminals=48
//To run specified transactions per terminal- runMins must equal zero
runTxnsPerTerminal=0
//To run for specified minutes- runTxnsPerTerminal must equal zero
runMins=15
//Number of total transactions per minute
limitTxnsPerMin=0

//Set to true to run in 4.x compatible mode. Set to false to use the
//entire configured database evenly.
terminalWarehouseFixed=true

//The following five values must add up to 100
//The default percentages of 45, 43, 4, 4 & 4 match the TPC-C spec
newOrderWeight=45
paymentWeight=43
orderStatusWeight=4
deliveryWeight=4
stockLevelWeight=4

// Directory name to create for collecting detailed result data.
// Comment this out to suppress.
resultDirectory=my_result_%tY-%tm-%td_%tH%tM%tS
osCollectorScript=./misc/os_collector_linux.py
osCollectorInterval=1
//osCollectorSSHAddr=user@dbhost
osCollectorDevices=net_eth0 blk_sda
```

------

### 非加密表空间下测试

- **150并发测试结果:**

![](D:\项目文件\金保\信创测试\加密表空间性能测试\非加密表空间-150并发.png)

- **300并发测试结果:**

![](D:\项目文件\金保\信创测试\加密表空间性能测试\非加密表空间-300并发.png)

- **500并发测试结果**

![](D:\项目文件\金保\信创测试\加密表空间性能测试\非加密表空间-500并发.png)

- **1000并发测试结果**![](D:\项目文件\金保\信创测试\加密表空间性能测试\非加密表空间-1000并发.png)

### 设置加密表空间

配置internal方式的透明加密



1. 开启透明加密

   ```
   ALTER SYSTEM SET tde_method='internal';
   ```

2. 创建keystore

   ```
   ADMINISTER KEY MANAGEMENT CREATE KEYSTORE sectest1 IDENTIFIED BY Wonders123;
   ```

3. 开启keystore

   ```
   ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY Wonders123;
   ```

4. 生成主密钥

   ```
   ADMINISTER KEY MANAGEMENT SET KEY IDENTIFIED BY Wonders123;
   ```

5. 创建表空间并指定加密算法

   ```
   CREATE TABLESPACE sectest_ts1 ENCRYPTION USING 'sm4-gcm';
   ```

   

6. 将表放入加密表空间

   ```
   alter table TPCC.BMSQL_CONFIG TABLESPACE sectest_ts1;     
   alter table TPCC.BMSQL_CUSTOMER TABLESPACE sectest_ts1;   
   alter table TPCC.BMSQL_DISTRICT TABLESPACE sectest_ts1;   
   alter table TPCC.BMSQL_HISTORY TABLESPACE sectest_ts1;    
   alter table TPCC.BMSQL_ITEM TABLESPACE sectest_ts1;       
   alter table TPCC.BMSQL_NEW_ORDER TABLESPACE sectest_ts1;  
   alter table TPCC.BMSQL_OORDER TABLESPACE sectest_ts1;     
   alter table TPCC.BMSQL_ORDER_LINE TABLESPACE sectest_ts1; 
   alter table TPCC.BMSQL_STOCK TABLESPACE sectest_ts1;      
   alter table TPCC.BMSQL_WAREHOUSE TABLESPACE sectest_ts1;  
   ```

   

7. 执行合并

   ```
   alter system major freeze;
   ```

8. 确认加密完成

   ```
   SELECT * FROM V$OB_ENCRYPTED_TABLES;
   ```

   blocks_decrypted字段为0，表示所有宏块加密完成

   ![](D:\项目文件\金保\信创测试\加密表空间性能测试\表加密情况.png)

### 加密表空间下进行测试

- 150并发![](D:\项目文件\金保\信创测试\加密表空间性能测试\加密表空间-150并发.png)
- 300并发![](D:\项目文件\金保\信创测试\加密表空间性能测试\加密表空间-300并发.png)
- 500并发![](D:\项目文件\金保\信创测试\加密表空间性能测试\加密表空间-500并发.png)

- 1000并发![](D:\项目文件\金保\信创测试\加密表空间性能测试\加密表空间-1000并发.png)

## 测试结果

| 并发数 | 非加密表空间/tpmC | 加密表空间/tpmC | 差异   |
| :----- | ----------------- | --------------- | ------ |
| 150    | 126954            | 123229          | -2.93% |
| 300    | 208724            | 193292          | -7.39% |
| 500    | 224081            | 207626          | -7.34% |
| 1000   | 298086            | 245710          | -17.5% |


## 待完成内容
编写sql执行计划绑定的脚本，根据sql_id和可用的plan_id绑定sql的执行计划
## 问题描述
	
## 问题原因
## 解决办法
### 绑定执行计划
	通过绑定执行计划，使得执行计划不稳定或执行计划跳变的sql固定下来，通常由两种方式: 
	通过sql_text
	通过sql_id
	
 ** 使用sql_text创建 **
```
CREATE [OR REPLACE] OUTLINE outline_name ON stmt [ TO target_stmt ];

```
* 指定or replace后，可以对已经存在的outline进行替换
* stmt 一般为一个带有hint和原始参数的DML语句
* 如果不指定to target_stmt,则表示数据库接受的SQL参数化后与stmt去掉HINT参数化文本相同，则将该SQL绑定stmt中HINT生成执行计划
* 如果期望对含有HINT的语句进行固定计划，则需要to target_stmt来指明原始的SQL

** 使用SQL_ID 创建 **
```
CREATE OUTLINE outline_name ON sql_id USING HINT hint;
```

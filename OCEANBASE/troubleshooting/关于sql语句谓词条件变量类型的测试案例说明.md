# 关于sql语句谓词条件变量值是否应该显式增加to_number或to_char函数的测试案例说明

## 测试表结构

![](D:\knowledge\database\oracle\oracle学习\实战案例\谓词变量类型一例\e3915d94-855f-46c2-b5c4-05ae1f0f18e3.png)

表索引:

![image-20251126102252502](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251126102252502.png)

## 测试sql语句

--字段类型是number,变量值类型为字符串

```
select * from test_case where object_id_num='56';
select * from test_case where object_id_num=to_char('56');
```

执行计划如下:

![image-20251126101451084](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251126101451084.png)

变量类型强制通过to_char指定为字符串类型时，oracle会隐式转换变量类型。

![image-20251126101600908](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251126101600908.png)

--字段类型是varchar2,变量值类型为number

```
select * from test_case where object_id_char=56;
select * from test_case where object_id_char=to_number('56');	
```

oracle会隐式转换字段类型，从而无法使用到字段索引，sql性能严重降低

![](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251126101753040.png)

![image-20251126101904011](C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20251126101904011.png)
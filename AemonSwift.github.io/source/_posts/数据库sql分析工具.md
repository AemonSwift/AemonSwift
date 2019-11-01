---
title: 数据库SQL分析工具
date: 2019-11-01 16:03:43
tags: 
- 数据库
- 分析工具
categories:
- 数据库
- 分析工具
---
主要是用explain工具来分析sql命中的情况

# explain之type字段
官网描述为连接类型(the join type)。它描述了找到所需数据使用的$\color{red}{扫描方式。}$
常见的扫描方式如下：
- system：系统表，少量数据，往往不需要进行磁盘IO；
- const：常量连接；
- eq_ref：主键索引(primary key)或者非空唯一索引(unique not null)等值扫描；
- ref：非主键非唯一索引等值扫描；
- range：范围扫描；
- index：索引树扫描；
- ALL：全表扫描(full table scan)；
上面扫描方式由快到慢顺序：
system > const > eq_ref > ref > range > index > ALL

## system ——不需要走io
从系统库mysql的系统表time_zone里查询数据，扫码类型为system，这些数据已经加载到内存里，不需要进行磁盘IO。 `explain select * from mysql.time_zone;`
内层嵌套(const)返回了一个临时表，外层嵌套从临时表查询，其扫描类型也是system，也不需要走磁盘IO，速度超快。 `explain select * from (select * from user where id=1) tmp;`

## const
成为const扫描的条件：且关系
1. 命中主键(primary key)或者唯一(unique)索引；
2. 被连接的部分是一个常量(const)值；
例如：`explain select * from user where id=1;`id是PK，连接部分是常量1。
这类扫描效率极高，返回数据量少，速度非常快。

## eq_ref
eq_ref扫描的条件为：对于前表的每一行(row)，后表只有一行被扫描。
再细化一点：
1. join查询；
2. 命中主键(primary key)或者非空唯一(unique not null)索引；
3. 等值连接；
例如：  `explain select * from user,user_ex where user.id=user_ex.id;`

## ref
eq_ref案例中的主键索引，改为普通非唯一(non unique)索引。`explain select * from user,user_ex where user.id=user_ex.id;`就由eq_ref降级为了ref，此时对于前表的每一行(row)，后表可能有多于一行的数据被扫描。
`explain select * from user where id=1;`当id改为普通非唯一索引后，常量的连接查询，也由const降级为了ref，因为也可能有多于一行的数据被扫描。
ref扫描，可能出现在join里，也可能出现在单表普通索引里，每一次匹配可能有多行数据返回，虽然它比eq_ref要慢，但它仍然是一个很快的join类型。

## range 
range扫描就比较好理解了，它是索引上的范围查询，它会在索引上扫码特定范围内的值。
```sql
explain select * from user where id between 1 and 4;
explain select * from user where idin(1,2,3);
explain select * from user where id>3;
```
像上例中的between，in，>都是典型的范围(range)查询。

## index
index类型，需要扫描索引上的全部数据。
`explain count (*) from user;`id是主键，该count查询需要通过扫描索引上的全部数据来计数。

## all
`explain select * from user,user_ex where user.id=user_ex.id;`如果id上不建索引，对于前表的每一行(row)，后表都要被全表扫描。

# 注意事项

## “列类型”与“where值类型”不符，不能命中索引，会导致全表扫描(full table scan)。
```sql
create table t1 (
cell varchar(3) primary key
)engine=innodb default charset=latin1;

insert into t1(cell) values ('111'),('222'),('333');
```

`explain select * from t1 where cell=111;`强制类型转换，不能命中索引，需要全表扫描，即3条记录(即explain中的rows字段)；
`explain select * from t1 where cell='111';`类型相同，命中索引，1条记录；(即explain中的rows字段)

## 相join的两个表的字符编码不同，不能命中索引，会导致笛卡尔积的循环计算（nested loop）。
```sql
create table t1 (
cell varchar(3) primary key
)engine=innodb default charset=utf8;

insert into t1(cell) values ('111'),('222'),('333');

create table t2 (
cell varchar(3) primary key
)engine=innodb default charset=latin1;

insert into t2(cell) values ('111'),('222'),('333'),('444'),('555'),('666');

create table t3 (
cell varchar(3) primary key
)engine=innodb default charset=utf8;

insert into t3(cell) values ('111'),('222'),('333'),('444'),('555'),('666');
```
1. t2和t1字符集不同，插入6条测试数据；
2. t3和t1字符集相同，也插入6条测试数据；
3. 除此之外，t1，t2，t3表结构完全相同；
`explain select * from t1,t2 where t1.cell=t2.cell;`第一个join，连表t1和t2（字符集不同），关联属性是cell；

`explain select * from t1,t3 where t1.cell=t3.cell;`第一个join，连表t1和t3（字符集相同），关联属性是cell；
测试结果：
1. t1和t2字符集不同，存储空间不同；
2. t1和t2相join时，遍历了t1的所有记录3条，t1的每一条记录又要遍历t2的所有记录6条，实际进行了笛卡尔积循环计算(nested loop)，索引无效；
3. ）t1和t3相join时，遍历了t1的所有记录3条，t1的每一条记录使用t3索引，即扫描1行记录；
# 参考文献
[两类非常隐蔽的全表扫描，不能命中索引](https://mp.weixin.qq.com/s/1Sowt2TcjMGDv55OQOe2rQ)
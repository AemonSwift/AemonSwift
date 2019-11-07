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

# id字段
1. id相同，执行顺序由上至下
2. id不同，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行
3. id相同不同，同时存在

# select_type字段
分别用来表示查询的类型，主要是用于区别普通查询、联合查询、子查询等的复杂查询。
- SIMPLE 简单的select查询，查询中不包含子查询或者UNION
- PRIMARY 查询中若包含任何复杂的子部分，最外层查询则被标记为PRIMARY
- PRIMARY 查询中若包含任何复杂的子部分，最外层查询则被标记为PRIMARY
- DERIVED 在FROM列表中包含的子查询被标记为DERIVED（衍生），MySQL会递归执行这些子查询，把结果放在临时表中
- UNION 若第二个SELECT出现在UNION之后，则被标记为UNION：若UNION包含在FROM子句的子查询中，外层SELECT将被标记为：DERIVED
- UNION RESULT  从UNION表获取结果的SELECT

# table字段
指的就是当前执行的表

# possible_keys 和 key
possible_keys 显示可能应用在这张表中的索引，一个或多个。查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询实际使用。
key表示实际使用的索引，如果为NULL，则没有使用索引。（可能原因包括没有建立索引或索引失效）

# key_len
表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度，在不损失精确性的情况下，长度越短越好。key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的。

# ref
显示索引的那一列被使用了，如果可能的话，最好是一个常数。哪些列或常量被用于查找索引列上的值。

# rows
根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数，也就是说，用的越少越好

# extra
包含不适合在其他列中显式但十分重要的额外信息

##  Using filesort（九死一生）
说明mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。MySQL中无法利用索引完成的排序操作称为“文件排序”。

## Using temporary（十死无生）
使用了用临时表保存中间结果，MySQL在对查询结果排序时使用临时表。常见于排序order by和分组查询group by。

## Using index（发财了）
表示相应的select操作中使用了覆盖索引（Covering Index），避免访问了表的数据行，效率不错。如果同时出现using where，表明索引被用来执行索引键值的查找；如果没有同时出现using where，表明索引用来读取数据而非执行查找动作。

## Using where
表明使用了where过滤

## Using join buffer
表明使用了连接缓存,比如说在查询的时候，多表join的次数非常多，那么将配置文件中的缓冲区的join buffer调大一些。

## impossible where
where子句的值总是false，不能用来获取任何元组

##  select tables optimized away
在没有GROUPBY子句的情况下，基于索引优化MIN/MAX操作或者对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化。

## distinct
优化distinct操作，在找到第一匹配的元组后即停止找同样值的动作

## 实例分析

1. 执行顺序1：select_type为UNION，说明第四个select是UNION里的第二个select，最先执行【select name,id from t2】
2. 执行顺序2：id为3，是整个查询中第三个select的一部分。因查询包含在from中，所以为DERIVED【select id,name from t1 where other_column=”】
3. 执行顺序3：select列表中的子查询select_type为subquery,为整个查询中的第二个select【select id from t3】
4. 执行顺序4：id列为1，表示是UNION里的第一个select，select_type列的primary表示该查询为外层查询，table列被标记为<derived3>,表示查询结果来自一个衍生表，其中derived3中的3代表该查询衍生自第三个select查询，即id为3的select。【select d1.name …..】
5. 执行顺序5：代表从UNION的临时表中读取行的阶段，table列的< union1,4 >表示用第一个和第四个select的结果进行UNION操作。【两个结果union操作】




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
[EXPLAIN用法和结果分析](https://blog.csdn.net/why15732625998/article/details/80388236)
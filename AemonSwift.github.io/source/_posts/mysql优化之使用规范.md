---
title: mysql优化之使用规范
date: 2019-11-03 21:05:23
tags:
- 数据库
categories:
- 数据库
---
# 基础规范

## 1. 使用InnoDB存储引擎
支持事务、行级锁、并发性能更好、CPU及内存缓存页优化使得资源利用率更高

## 2. 推荐使用utf8mb4字符集 
无需转码，无乱码风险, 支持emoji表情以及部分不常见汉字

## 3. 表、字段必须加注释
 方便他人理解字段意思，在后期维护中非常非常有用，不用去瞎猜这个字段是干嘛的。

## 4. 不在数据库做计算
禁止使用存储过程、视图、触发器、Event。在并发量大的情况下，这些功能很可能将数据库拖跨，业务逻辑放到服务层具备更好的扩展性，能够轻易实现“增机器就加性能”。

## 5. 禁止存储文件
文件存储在文件系统，数据库里存URI

## 6. 控制单表数据量
单表记录控制在千万级，超过了可以进行分库分表

## 7. 不要在数据库中进行排序。
特别是大数据量的排序，可考虑在程序中设计排序；

## 8. 不要对数据做真正意义的物理删除(DELETE…)。
可考虑逻辑删除，即在表中设计一个is_deleted字段标记该字段是否删除，防止毁灭性事件的发生；

## 9. 避免在数据库中做计算，减轻数据库压力

## 10. 避免JOIN查询，请尽可能的使用单表查询，减少查询复杂度，减轻数据库压力

## 11. 禁止在数据库中使用视图、存储过程、函数、触发器、事件
# 命名规范

## 1. 库名、表名、字段名、索引名：小写，下划线风格
非唯一索引按照”ix字段名称[字段名称]”进行命名，如ix_uid_name;
唯一索引按照”uk字段名称[字段名称]”进行命名，如uk_uid_name;
## 2. 表必须有主键，例如自增主键
a. 主键递增，数据行写入可以提高插入性能; 
b. 主键要选择较短的数据类型，Innodb引擎普通索引都会保存主键的值，较短的数据类型可以有效的减少索引的磁盘空间，提高索引的缓存效率; 
c. 保证实体的完整性，唯一性。
## 3. 不要使用外键。
如果有外键约束，用应用程序控制。外键会导致表与表之间耦合，update与delete操作都会涉及相关联的表，十分影响sql 的性能，甚至会造成死锁。高并发情况下容易造成数据库性能下降，大数据高并发业务场景数据库使用以性能优先。

# 字段设计规范
## 1. 把字段定义为NOT NULL并且提供默认值。
- 数值类型使用：NOT NULL DEFAULT 0
- 字符类型使用：NOT NULL DEFAULT ""
- $\color{red}{timestamp类型不指定默认值的话，MariaDB 会默认给0；多于一个timestamp字段没有指定默认值，会自动给一个timestamp默认值为 CURRENT_TIMESTAMP ON UPDATE CURRENT\\_TIMESTAMP，其他为0。}$

主要原因在于：

a. null的列使索引/索引统计/值比较都更加复杂，对MySQL来说更难优化; 
b. null 这种类型MySQL内部需要进行特殊处理，增加数据库处理记录的复杂性；同等条件下，表中有较多空字段的时候，数据库的处理性能会降低很多; 
c. null值需要更多的存储空间，无论是表还是索引中每行中的null的列都需要额外的空间来标识; 
d. 对null 的处理时候，只能采用is null或is not null，而不能采用=、in、<、<>、!=、not in这些操作符号。如：where name!=’zhangsan’，如果存在name为null值的记录，查询结果就不会包含name为null值的记录。

## 2. 不要使用TEXT、BLOB类型
会浪费更多的磁盘和内存空间，非必要的大量的大字段查询会淘汰掉热数据，导致内存命中率急剧降低，影响数据库性能,如果必须要使用则独立出来一张表，用主键来对应，避免影响其它字段索引效率。

## 3. 避免使用浮点数类型
计算机处理整型比浮点型快N倍，如果必须使用，请将浮点型扩大N倍后转为整型；建议使用整数，小数容易导致钱对不上。即通常做法为：涉及精确金额相关用途的字段类型，强烈建议扩大N倍后转换成整型存储（例如金额中的分扩大百倍后存储成整型），避免浮点数加减出现不准确的问题，也强烈建议比实际需求多保留一位，便于后续财务方面对账更加准确

## 4. 必须使用varchar存储手机号
手机号会去做数学运算么？

## 5. 为提高效率可以牺牲范式设计，冗余数据
冗余字段选择要求：
a. 不是频繁修改的字段; 
b. 不是 varchar 超长字段，更不能是 text 字段。

## 6. 所有表都必须要有主键
主键类型必须为：INT/BIGINT  unsigned NOT NULL AUTO_INCREMENT，提高顺序insert效率，强烈建议该列与业务没有联系，并且不建议使用组合主键，仅仅作为自增主键id使用。
INT／BIGINT如何选择？
a. 当表的预估数据量在42亿条以内，请使用INT UNSIGNED；b. 当表的预估数据量超过42亿条，请使用BIGINT UNSIGNED;
为什么选择自增id作为主键？
a. 主键自增，数据行写入可以提高插入性能，可以避免page分裂，减少表碎片提升空间和内存的使用
b. 自增型主键设计(int,bigint)可以降低二级索引的空间，提升二级索引的内存命中率；
c. 主键要选择较短的数据类型， Innodb引擎普通索引都会保存主键的值，较短的数据类型可以有效的减少索引的磁盘空间，提高索引的缓存效率;
d. 无主键的表删除，在row模式的主从架构，会导致备库夯住。

## 7. 所有表必须携带ctime(创建时间),mtime(最后修改时间)这两个字段，便于数据分析以及故障排查；
```SQL
#两个字段的类型如下，只需要在建表时建立即可，不需要开发人员再往其中插入时间值，前提是INSERT INTO语句显示的字段名称：
ctime TIMESATMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT ‘创建时间’;
mtime TIMESATMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT ‘最后修改时间’
```

## 9. 对于CHAR(N)/VARCHAR(N)类型，在满足够用的前提下，尽可能小的选择N的大小，并且建议N<255，用于节省磁盘空间和内存空间；
## 10. 使用TINYINT代替ENUM类型，新增ENUM类型需要在DDL操作，对于TINYINT类型在数据库字段COMMENT和程序代码中做好备注信息，避免混淆
```SQL
num enum('0','1','2','3') comment 'enum枚举类型' );
`num` tinyint(4) NOT NULL DEFAULT '0' COMMENT 'TINY枚举类型：0-不通过，1-通过'
```
## 10. JOIN查询时，用于JOIN的字段定义必须完全相同(避免隐式转换)，并且建立索引。
## 11. 存储单个IP时，必须使用整型INT UNSIGNED类型，不允许使用字符型VARCHAR()存储单个IP。
## 时间类型，首选使用整型INT、INT UNSIGNED类型，其次使用timestamp类型。
- INT: 存储范围：-2147483648 to 2147483647 对应的时间范围: 1970/1/1 8:00:00 – 2038/1/19 11:14:07
- INT UNSIGNED: 存储范围：0 to 4294967295 对应的时间范围：1970/1/1 8:00:00 – 2106/2/7 14:28:15
## 12. 日期类型，请使用date类型。

# 索引设计规范
## 1. 禁止在更新十分频繁、区分度不高的属性上建立索引
a. 更新会变更B+树，更新频繁的字段建立索引会大大降低数据库性能; 
b. “性别”这种区分度不大的属性，建立索引是没有什么意义的，故找区分度大的字段作为索引。
## 2. 建立组合索引，必须把区分度高的字段放在最左边
因为使用单字段查询时会使用组合索引的左边字段而不使用右边的字段，如果 where a=? and b=? ， a 列的几乎接近于唯一值，那么只需要单建 idx_a 索引即可。

## 3. 页面搜索严禁左模糊或者全模糊
索引文件具有 B-Tree 的最左前缀匹配特性，如果左边的值未确定，那么无法使用此索引, 如果需要请走搜索引擎来解决。

# SQL使用规范
## 1. 禁止使用SELECT *，只获取必要的字段，需要显示说明列属性
a. 消耗cpu，io，内存，带宽; 
b. 不能有效的利用覆盖索引; 
c. 使用SELECT *容易在增加或者删除字段后出现程序BUG, 不具有扩展性。
d. 没有避免新增字段对程序应用逻辑的影响

## 2. 使用INSERT INTO t_xxx VALUES(xxx)，必须显示指定插入的列属性
容易在增加或者删除字段后出现程序BUG

## 3. 务必请使用“同类型”进行比较，否则可能全表扫面
SELECT name FROM t_user WHERE phone=1888888888 会导致全表扫描.

## 4. 禁止在WHERE条件的上使用函数或者计算
解读：SELECT naem FROM tuser WHERE date(createdatatime)='2017-12-29' 会导致全表扫描
推荐的写法是：SELECT name FROM tuser WHERE createdatatime>= '2017-12-29' and create_datatime < '2017-12-30'

## 5. 禁止负向查询，以及%开头的模糊查询
a. 负向查询条件：NOT、!=、<>、!<、!>、NOT IN、NOT LIKE等，会导致全表扫描。b. %开头的模糊查询，会导致全表扫描。

## 6. 不要大表使用JOIN查询，禁止大表使用子查询
会产生临时表，消耗较多内存与CPU，极大影响数据库性能。

## 7. OR改写为IN()或者UNION
原因很简单or不会走索引。

## 8. 简单的事务
事务就像程序中的锁一样粒度尽可能要小。

## 9. 不要一次更新大量数据
数据更新会对行或者表加锁，应该分为多次更新。

## 10. 生产环境中，表一旦设计好，字段只允许增加(ADD COLUMN)，禁止减少(DROP COLUMN)，禁止改名称(CHANGE/MODIFY COLUMN);

## 11. 禁止使用UPDATE ... LIMIT ...和DELETE ... LIMIT ...操作
因为你无法得知自己究竟更新或者删除了哪些数据，请务必添加ORDER BY进行排序
```SQL
# 这是错误的语法示例
UPDATE tb SET col1=value1 LIMIT n;
# 这是错误的语法示例
DELETE FROM tb LIMIT n;
# 这是正确的语法示例
UPDATE tb SET col1=value1 ORDER BY id LIMIT n;
# 这是正确的语法示例
DELETE FROM tb ORDER BY id LIMIT n;
```
## 12. 禁止超过2张表的JOIN查询

## 13. 禁止使用子查询

## 14. 禁止出现冗余索引，如索引(a),索引(a,b)，此时索引(a)为冗余索引;

## 15. 禁止使用ORDER BY RAND()排序，性能极其低下。 

## 16. 禁止使用外键，外键的逻辑应当由程序去控制
外键会导致表与表之间耦合，UPDATE与DELETE操作都会涉及相关联的表，十分影响SQL 的性能，甚至会造成死锁。高并发情况下容易造成数据库性能，大数据高并发业务场景数据库使用以性能优先。

## 17. 禁止回退表的DDL操作

# 例子
## 1. 创建表
```SQL
# 这是正确的语法示范
CREATE TABLE `test` (
  `c1` int(11) NOT NULL AUTO_INCREMENT COMMENT '自增主键ID',
  `c2` int(11) NOT NULL DEFAULT '0' COMMENT '无符号数值型字段',
  `c3` int(11) NOT NULL DEFAULT '0' COMMENT '有符号数值型字段',
  `c4` varchar(16) NOT NULL DEFAULT '0' COMMENT '变长字符型字段',
  `ctime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间类型字段',
  `mtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间类型字段',
  `c7` tinyint(4) NOT NULL DEFAULT '0' COMMENT '枚举类型字段：0-xxx,1-xxx,2-xxx',
  PRIMARY KEY (`c1`),
  UNIQUE KEY `uk_c2` (`c2`),
  KEY `ix_c3` (`c3`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='测试表'
```
## 更改表
### 添加字段
```SQL
# 这是正确的语法示范
ALTER TABLE test ADD COLUMN c8 int(11) NOT NULL DEFAULT 0 COMMENT '添加字段测试';
```
添加字段时禁止使用after/before属性，避免数据偏移。

### 变更字段
```SQL
# 这是正确的语法示范
# MODIFY只修改字段定义（优先使用）
ALTER TABLE test MODIFY COLUMN c8 varchar(16) NOT NULL DEFAULT 0 COMMENT 'MODIFY修改字段定义';
# CHANGE修改字段名称
ALTER TABLE test CHANGE COLUMN c7 c8 varchar(16) NOT NULL DEFAULT 0 COMMENT 'CHANGE修改字段名称';
```
### 删除字段
```SQL
# 这是正确的语法示范
ALTER TABLE test DROP COLUMN c8;
```

### 添加主键
```SQL
# 这是正确的语法示范
ALTER TABLE  test ADD PRIMARY KEY(c1);
```

### 删除主键
```SQL
# 这是正确的语法示范
ALTER TABLE test DROP PRIMARY KEY;
```

### 添加普通索引
```SQL
# 这是正确的语法示范
alter table test add INDEX ix_c3(c3);
```
如果创建的是联合索引，筛选度高的列靠左

### 添加唯一索引
```SQL
# 这是正确的语法示范
alter table test add UNIQUE INDEX uk_c2(c2);
```

### 删除普通索引
```SQL
# 这是正确的语法示范
alter table test DROP INDEX ix_c3;
```

### 删除唯一索引
```SQL
# 这是正确的语法示范
alter table test DROP INDEX uk_c2;
```
同一张表的修改语句请合并，避免多次重建表影响读写
```SQL
alter table test add col1 int(11) NOT NULL DEFAULT 0 COMMENT 'col1字段用途',add index idx_col2(col2),modify col3 varchar(16) NOT NULL DEFAULT '' COMMENT 'col3字段用途'
```
## 插入语句
```SQL
# INSERT INTO语句的正确语法示例
INSERT INTO tb(col1,col2) VALUES(value1,values2);
```
INSERT INTO语句需要显示指明字段名称;
对于多次单条INSERT INTO语句，务必使用批量INSERT INTO语句，提高INSERT INTO语句效率，如：
```SQL
# 多次单条INSERT INTO，这是错误的语法示例
INSERT INTO test(col1,col2) VALUES(value1,values2);
INSERT INTO test(col1,col2) VALUES(value3,values4);
INSERT INTO test(col1,col2) VALUES(value5,values6);
# 批量INSERT INTO语句，这是正确的语法示例
INSERT INTO test(col1,col2) VALUES(value1,values2),(value3,values4),(value5,values6);
```

## 更新语句
```SQL
# UPDATE语句的正确语法示例
UPDATE tb SET col1=value1,col2=value2,col3=value3 WHERE col0=value0 AND col5=value5;
```
SET后接的并列字段分隔符为”逗号(,)”，而不是常见的”AND”，使用”AND”也能将UPDATE语句执行成功，但意义完全不一样。原来在UPDATE … SET后接分隔符为”AND”的语句，由于AND的优先级较高，所以先处理“AND”，再处理“＝”，于是“＝”后面的值只有逻辑运算的结果true(1) / false(0)。例如：update t set c1=11 and c2='AA' where id=1; 在这条语句中MySQL将c1=11 and c2='AA'解析成了c1=(11 and c2='AA')
强烈建议UPDATE语句后携带WHERE条件，防止灾难性事件的发生；
使用UPDATE修改大量数据时，该语句极易引起主从复制延迟；
禁止使用UPDATE ... LIMIT ...语法

## 查询语句
```sql
# 这是错误的语法示范
SELECT * FROM tb WHERE col1=value1;
# 这是正确的语法示范
SELECT col1,col2 FROM tb WHERE col1=value1;
```
禁止使用SELECT * FROM语句，SELECT只获取需要的字段，既防止了新增字段对程序应用逻辑的影响，又减少了对程序和数据库的性能影响；
合理的使用数据类型，避免出现隐式转换，隐式转换无法使用索引且效率低，如：SELECT name FROM tb WHERE id='1'; 此时id为int类型，此时出现隐式转换以及全表扫描。
不建议使用％前缀模糊查询，导致查询无法使用索引，如：SELECT id FROM tb WHERE name LIKE '%test';
对于LIMIT操作，强烈建议使先ORDER BY 再LIMIT，即ORDER BY c1 LIMIT n；

## 删除语句
```SQL
# DELETE语句的正确语法示例
DELETE FROM tb WHERE col0=value0 AND col1=value1;
```
强烈建议DELETE语句后携带WHERE条件，防止灾难性事件的发生；
使用DELETE修改大量数据时，该语句极易引起主从复制延迟；
禁止使用DELETE ... LIMIT ...语法

## 其它书写规范
1. 禁止再where后面给字段使用mysql中的函数
```SQL
# 这是错误的语法示范
SELECT col1,col2 FROM test WHERE unix_timestamp(col1)=value1;
# 这是正确的语法示范
SELECT col1,col2 FROM test WHERE col1=unix_timestamp(value1);
```

## 强烈建议字段放在操作符左边
```SQL
# 这是错误的语法示范
SELECT col1,col2 FROM tb WHERE value1=col1l
# 这是正确的语法示范
SELECT col1,col2 FROM tb WHERE col1=value1;
```

## 禁止将字符类型传入到整型类型字段中，也禁止整形类型传入到字段类型中，存在隐式转换的问题；
```SQL
# 这是错误的语法示范
# var_col字段为VARCHAR类型
SELECT col1,col2 FROM test WHERE var_col=123;
# int_col字段为INT类型
SELECT col1,col2 FROM test WHERE int_col='123';
# 这是正确的语法示范
# var_col字段为VARCHAR类型
SELECT col1,col2 FROM test WHERE var_col='123';
# int_col字段为INT类型
SELECT col1,col2 FROM test WHERE int_col=123;
```

# 数据库配置规范
如果应用使用的是长连接，应用必须具有自动重连的机制，但请避免每执行一个SQL去检查一次DB可用性；
如果应用使用的是长连接，应用应该具有连接的TIMEOUT检查机制，及时回收长时间没有使用的连接，TIMEOUT时间一般建议为2小时；
程序访问数据库连接的字符集请设置为utf8mb4；
程序中禁止一切DDL操作。
禁止使用应用程序配置文件内的帐号手工访问线上数据库，大部分配置文件内的数据库配置的是主库，你无法预知你的一条SQL会不会导致MySQL崩溃；
突发性大量操作数据库等操作时，需进行流量评估，避免数据库出现瓶颈；
批量清洗数据，应避开业务高峰期时段执行，并在执行过程中观察服务状态；
禁止在主库上执行后台管理和统计类的功能查询，这种复杂类的SQL会造成CPU的升高，进而会影响业务。

# 常用字段数据类型范围
数值|取值范围
:-:|:-:
TINYINT(4)	|-128 ~ 127
TINYINT(4) UNSIGNED|	0 ~ 255
SMALLINT(6)	|-32768 ~ 32767
SMALLINT(6) UNSIGNED|	0 ~ 65535
MEDIUMINT(8)|	-8388608 ~ 8388607
MEDIUMINT(8) UNSIGNED	|0 ~ 16777215
INT(11)|	-2147483648 ~ 2147483647
INT(11) UNSIGNED	|0 ~ 4294967295
BIGINT(20)|	-9223372036854775808 ~ 9223372036854775807
BIGINT(20) UNSIGNED|	0 ~ 18446744073709551615

VARCHAR(N)：在MySQL数据库中，VARCHAR(N)中的N代表N个字符，不管你是中文字符还是英文字符，VARCHAR(N)能存储最大为N个中文字符/英文字符。
TIMESTAMP: 1970-01-01 00:00:01 UTC ~2038-01-19 03:14:07 UTC
DATETIME: 1000-01-0100:00:00 ~ 9999-12-31 23:59:59
# 参考文献
[为什么InnoDB表要建议用自增列做主键](https://imysql.com/2014/09/14/mysql-faq-why-innodb-table-using-autoinc-int-as-pk.shtml)
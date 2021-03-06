# 为什么查询速度会慢（问题)

查询性能低下最基础的原因是访问的数据大多。

1. 请求了不需要的数据。
2. mysql是否在扫描额外的记录。

一个查询由多个子任务组成。每个子任务都会消耗一定时间，优化查询，实际上要优化起子任务，要么`消除其中一些子任务，要么减少子任务的执行次数`。



----



# 慢查询基础：优化访问（目标）

最简单衡量查询开销的指标：

	* 响应时间
	* 扫描的行数
	* 返回的行数





----



# 查询执行的基础

mysql执行一个查询的过程



![img](https://raw.githubusercontent.com/pujie147/nouk-doc/master/mysql/images/20191204191659238.png)

## 1.client/server 通信方式

* TCP/IP：
  是一套常见的通信协议，被用于连接主机和互联网，但是同样也可以通过它来用于本地连接，可以用于所有的操作系统。

* Unix socket file：
  该协议允许在同样的主机上连接彼此，它依靠物理文件“socket”来实现，该协议可以很直接轻易的交换数据。
  在unix/linux上是最好的本地连接选择。

* Shared memory：
  Windows创造一块可被信任读写的内存区域，程序可以很直接的连接内存而不是通过操作系统，这将变得十分高效。
  默认是不可用，如果需要使用，需在启动服务的时候加上“--shared-memory”选项。

* Named pipes：
  该协议类似共享内存，让进程通过内存区域与其他进程交换信息，不像共享内存的是，命名管道通过Local Area Network

## 2.查询缓存

缓存存放在一个引用表中，通过一个哈希值（sql产生）引用，这个哈希值包括了如下因素：

查询语句、当前要查询的数据库、客户端协议的版本等一些其他可能会影响返回结果的信息。

手动操作缓存：

```sql
RESET QUERY CACHE；#从查询缓存中移除所有查询
```

查询缓存系统配置：

```sql
mysql> SHOW GLOBAL VARIABLES LIKE '%query_cache%';
+------------------------------+----------+
| Variable_name                | Value    |
+------------------------------+----------+
| have_query_cache             | YES     |      
| query_cache_limit            | 1048576  |-- 查询结果大于这个值，则不会被缓存。单位Bytes
| query_cache_min_res_unit     | 4096     |
| query_cache_size             | 16777216 |
| query_cache_strip_comments   | OFF      |
| query_cache_type             | ON       |-- ON|OFF|DEMAND
| query_cache_wlock_invalidate | OFF      |
+------------------------------+----------+]
```

与缓存相关的状态变量:

```sql
mysql> SHOW  GLOBAL STATUS  LIKE  'Qcache%';
+-------------------------+----------+
| Variable_name            | Value   |
+-------------------------+----------+
| Qcache_free_blocks       | 1       | #查询缓存中的空闲块
| Qcache_free_memory       | 16759656| #查询缓存中尚未使用的空闲内存空间
| Qcache_hits              | 16      | #缓存命中次数
| Qcache_inserts           | 71      | #向查询缓存中添加缓存记录的条数
| Qcache_lowmem_prunes     | 0       | #表示因缓存满了而不得不清理部分缓存以存储新的缓存，这样操作的次数。若此数值过大，则表示缓存空间太小了。
| Qcache_not_cached        | 57      | #没能被缓存的次数
| Qcache_queries_in_cache  | 0       | #此时仍留在查询缓存的缓存个数
| Qcache_total_blocks      | 1       | #共分配出去的块数
+-------------------------+----------+
```

**命中率：Qcache_hits/(Qcache_hits+Com_select)**



## 3.解析器

1.词法分析

扫描字符流，根据构词规则识别单个单词。生成多个token，分别为关键字和非关键字。

2.语法分析

在词法分析的基础上将单词序列组成语法短语，最后生成语法树。



## 4.预处理器



根据一些mysql规则进一步检查解析树是否合法。如检查查询的表名、列名是否正确，是否有表的权限等。

![img](https://raw.githubusercontent.com/pujie147/nouk-doc/master/mysql/images/1385831-20190219101803343-1749289173.png)

```sql
CREATE TABLE students (
    id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    age INT(10) 
) ENGINE=InnoDB;
```



```sql
--声明带占位符的预处理
prepare select_student from 'select * from students where age between ? and ?';
--定义两个变量
set @min = 1;
set @max = 18;
--使用两个变量代替占位符执行SQL指令
execute select_student using @min,@max;
drop prepare select_student;
```



## 5.查询优化器

```sql
select count(*) from students;
show status like 'last_query_cost'; -- 查询最后一次的查询成本
```

mysql会更具：每个表或者索引页面个数、索引基数、索引和数据行的常熟、索引分布情况得出查询的成本。

然后把成本最低的执行sql传给执行引擎。



## 6.执行引擎

执行引擎会通过执行计划逐条执行，执行过程中大量会用到存储引擎（存储引擎提供了一套接口`handler api`）

### 链接

```sql
mysql> SELECT tbl1. col1, tbl2. col2 
	   FROM tbl1 LEFT OUTER JOIN tbl2 USING( col3) 
       WHERE tbl1.col1 IN( 5, 6);
```

```sql
outer_iter = iterator over tbl1 where col1 IN( 5, 6) 
while outer_row = outer_iter.next 
	inner_iter = iterator over tbl2 where col3 = outer_row.col3 
	inner_row = inner_iter.next 
	if inner_row
    	while inner_row 
    		output [outer_row.col1,inner_row.col2]
    		inner_ row = inner_ iter.next
        end 
    else 
    	output[outer_row.col1,NULL] 
    end 
    outer_row = outer_iter.next 
end

```

![img](https://raw.githubusercontent.com/pujie147/nouk-doc/master/mysql/images/format,png)

### 关联排序

	* ORDER BY 子句中的所有字段都来自于第一张表，mysql会在关联处理第一张表时进行文件排序。explain extra字段（Using filesort）。
	* ORDER BY 子句中的字段非第一张表，则mysql会所有关联结束后，再进行文件排序。explain extra字段（Using temporary;Using filesort）。需要临时表排序 limit 数据非常大。5.6版本之后对该策略有做新优化。



# Mysql的join算法

## 1. Nested-Loop Join

在Mysql中，使用Nested-Loop Join的算法思想去优化join，Nested-Loop Join翻译成中文则是“嵌套循环连接”。

> 举个例子：
> select * from t1 inner join t2 on t1.id=t2.tid
> （1）t1称为外层表，也可称为驱动表。
> （2）t2称为内层表，也可称为被驱动表。
>
> ```sql
> //伪代码表示：
> List<Row> result = new ArrayList<>();
> for(Row r1 in List<Row> t1){
> 	for(Row r2 in List<Row> t2){
> 		if(r1.id = r2.tid){
> 			result.add(r1.join(r2));
> 		}
> 	}
> }
> ```



**在Mysql的实现中，Nested-Loop Join有3种实现的算法：**

- Simple Nested-Loop Join：SNLJ，简单嵌套循环连接
- Index Nested-Loop Join：INLJ，索引嵌套循环连接
- Block Nested-Loop Join：BNLJ，缓存块嵌套循环连接

在选择Join算法时，会有优先级，理论上会优先判断能否使用INLJ、BNLJ：
**Index Nested-LoopJoin > Block Nested-Loop Join > Simple Nested-Loop Join**



## 2. Simple Nested-Loop

![img](D:\git-pjs\nouk-doc\mysql\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NDEyOTY=,size_16,color_FFFFFF,t_70)

简单嵌套循环连接实际上就是简单粗暴的嵌套循环，如果table1有1万条数据，table2有1万条数据，那么数据比较的次数=1万 * 1万 =1亿次，这种查询效率会非常慢。



## 3. Index Nested-LoopJoin

![img](D:\git-pjs\nouk-doc\mysql\images\index_nested-loopjoin)

索引嵌套循环连接是基于索引进行连接的算法，索引是基于内层表的，通过外层表匹配条件直接与内层表索引进行匹配，避免和内层表的每条记录进行比较， 从而利用索引的查询减少了对内层表的匹配次数，优势极大的提升了 join的性能：

> 原来的匹配次数 = 外层表行数 * 内层表行数
> 优化后的匹配次数= 外层表的行数 * 内层表索引的高度



## 4. Block Nested-Loop Join

![img](D:\git-pjs\nouk-doc\mysql\images\block_nested-loopjoin)

缓存块嵌套循环连接通过一次性缓存多条数据，把参与查询的列缓存到Join Buffer 里，然后拿join buffer里的数据批量与内层表的数据进行匹配，从而减少了内层循环的次数（遍历一次内层表就可以批量匹配一次Join Buffer里面的外层表数据）





# Mysql 查询优化器局限性

## 1.关联子查询

mysql5.6之前子查询实现的非常糟糕，最糟糕的一类查询是where条件中包括in的子查询。

```sql
explain select * from course where cid in (select course_id from score where student_id = 7 and num > 85);
```

mysql会将相关的外层表压倒子表中，会将查询改写成下面样子：

```sql
explain select * from course where exists (select course_id from score where student_id = 7 and num > 85 and course_id = cid);
```

使用不上course表的主键索引。

| id   | select_type        | table  | type | possible_keys                    |
| ---- | ------------------ | ------ | ---- | -------------------------------- |
| 1    | PRIMARY            | course | ALL  | NULL                             |
| 2    | DEPENDENT SUBQUERY | score  | ref  | fk_score_student,fk_score_course |

> mysql5.7做了优化会把子查询的数据做个临时表（子查询物化：MATERIALIZED）
>
> ```sql
> +----+--------------+-------------+------------+--------+----------------------------------+------------------+---------+-----------------------+------+----------+-------------+
> | id | select_type  | table       | partitions | type   | possible_keys                    | key              | key_len | ref                   | rows | filtered | Extra       |
> +----+--------------+-------------+------------+--------+----------------------------------+------------------+---------+-----------------------+------+----------+-------------+
> |  1 | SIMPLE       | <subquery2> | NULL       | ALL    | NULL                             | NULL             | NULL    | NULL                  | NULL |   100.00 | NULL        |
> |  1 | SIMPLE       | course      | NULL       | eq_ref | PRIMARY                          | PRIMARY          | 4       | <subquery2>.course_id |    1 |   100.00 | NULL        |
> |  2 | MATERIALIZED | score       | NULL       | ref    | fk_score_student,fk_score_course | fk_score_student | 4       | const                 |    4 |    33.33 | Using where |
> +----+--------------+-------------+------------+--------+----------------------------------+------------------+---------+-----------------------+------+----------+-------------+
> ```

从以上例子就可以知道了mysql的每个版本都有可能推翻以前优化的思路，所以优化好方案还是在生产相同的环境下多多调试。



####  关联子查询的劣势例子

下面两条sql看执行计划差别不大

```sql
explain select * from student_1 where not exists (select 1 from class where class_id = class.cid);
```

但是第二条sql 的explain中Extra里包括`not exists (可以理解为提前终端)`

```sql
explain select * from student_1 left outer join class on (class_id = class.cid) where class.cid is null;

student_1_iter = iterator over student_1 
while student_1_row = student_1_iter.next 
	class_iter = iterator over class
	while class_row = class_iter.next 
    	if student_1_row.class_id = class_row.id
    		break;
    	end
    end 
    output [outer_row.col1,inner_row.col2]
end
```

> mysql在8.0之后
>
> ```sql
> mysql> show warnings;
> +-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
> | Level | Code | Message
>                                                                                                                  |
> +-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
> | Note  | 1276 | Field or reference 'test1.student_1.class_id' of SELECT #2 was resolved in SELECT #1
>                                                                                                                  |
> | Note  | 1003 | /* select#1 */ select `test1`.`student_1`.`sid` AS `sid`,`test1`.`student_1`.`gender` AS `gender`,`test1`.`student_1`.`class_id` AS `class_id`,`test1`.`student_1`.`sname` AS `sname` from `test1`.`student_1` anti join (`test1`.`class`) on((`<subquery2>`.`cid` = `test1`.`student_1`.`class_id`)) where true |
> +-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
> 2 rows in set (0.00 sec)
> ```
>
> 可以看到mysql直接优化成anti join了
>
> ```sql
> mysql> explain analyze select * from student_1 where not exists (select 1 from class where class_id = class.cid);
> +----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
> | EXPLAIN
> 
>                                                                                      |
> +----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
> | -> Nested loop antijoin  (actual time=0.302..0.307 rows=1 loops=1)
>     -> Table scan on student_1  (cost=1.85 rows=16) (actual time=0.036..0.060 rows=16 loops=1)
>     -> Single-row index lookup on <subquery2> using <auto_distinct_key> (cid=student_1.class_id)  (actual time=0.003..0.003 rows=1 loops=16)
>         -> Materialize with deduplication  (actual time=0.011..0.011 rows=1 loops=16)
>             -> Index scan on class using PRIMARY  (cost=0.65 rows=4) (actual time=0.009..0.018 rows=4 loops=1)
>  |
> +----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
> 1 row in set, 1 warning (0.00 sec)
> ```
>
> 可以使用8.0新工具analyze查询更具体的情况



#### 关联子查询的优势例子



```sql
explain select distinct cid from class inner join student_1 on (class_id = cid);
```

在使用distinct和group by 时，通常需要产生临时中间表。



```sql
explain select cid from class where exists ( select 1 from student_1 where class_id = cid);
```

而第二条sql没有产生零食表的过程，而且只要存在就返回。



## 2.union

在union时mysql 会对结果集，生产一张临时表。所以尽可能让这个临时表的数据小是一个比较好的优化方向。因为union之后的结果集一般还有可能在外层添加

`group by` 、`order by` 等操作。

> ##### 优化前
>
> ```sql
> SELECT sname,class_id FROM 
> (
> 	(
> 		SELECT
> 			sname,
> 			class_id
> 		FROM
> 			student
> 	)
> 	UNION ALL
> 	(
> 		SELECT
> 			sname,
> 			class_id
> 		FROM
> 			student_1
> 	)
> ) a
> ORDER BY sname
> limit 10
> ```
>
> ##### 优化后
>
> ```sql
> SELECT sname,class_id FROM 
> (
> 	(
> 		SELECT
> 			sname,
> 			class_id
> 		FROM
> 			student
>         ORDER BY sname
>         limit 10
> 	)
> 	UNION ALL
> 	(
> 		SELECT
> 			sname,
> 			class_id
> 		FROM
> 			student_1
>         ORDER BY sname
>         limit 10
> 	)
> ) a
> ORDER BY sname
> limit 10
> ```
>





## 3.索引合并

交集，获取两个索引然后获取交集数据。使用到了2个索引

#### intersect

```sql

EXPLAIN
SELECT
	*
FROM
	score
WHERE
	student_id = 1
AND course_id = 1



+----+-------------+-------+------------+-------------+----------------------------------+----------------------------------+---------+------+------+----------+----------------------------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys                    | key                              | key_len | ref  | rows | filtered | Extra                                                          |
+----+-------------+-------+------------+-------------+----------------------------------+----------------------------------+---------+------+------+----------+----------------------------------------------------------------+
|  1 | SIMPLE      | score | NULL       | index_merge | fk_score_student,fk_score_course | fk_score_student,fk_score_course | 4,4     | NULL |    1 |      100 | Using intersect(fk_score_student,fk_score_course); Using where |
+----+-------------+-------+------------+-------------+----------------------------------+----------------------------------+---------+------+------+----------+----------------------------------------------------------------+
```

并集，通过2个索引等到数据并合并。

#### union

```sql
EXPLAIN
SELECT
	*
FROM
	score
WHERE
	student_id = 1
OR course_id = 1



+----+-------------+-------+------------+-------------+----------------------------------+----------------------------------+---------+------+------+----------+------------------------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys                    | key                              | key_len | ref  | rows | filtered | Extra                                                      |
+----+-------------+-------+------------+-------------+----------------------------------+----------------------------------+---------+------+------+----------+------------------------------------------------------------+
|  1 | SIMPLE      | score | NULL       | index_merge | fk_score_student,fk_score_course | fk_score_student,fk_score_course | 4,4     | NULL |   15 |      100 | Using union(fk_score_student,fk_score_course); Using where |
+----+-------------+-------+------------+-------------+----------------------------------+----------------------------------+---------+------+------+----------+------------------------------------------------------------+
```

和union的区别就是添加了sort

#### sort_union

```sql
EXPLAIN
SELECT
	*
FROM
	score
WHERE
	student_id < 2
OR course_id > 7

+----+-------------+-------+------------+-------------+----------------------------------+----------------------------------+---------+------+------+----------+-----------------------------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys                    | key                              | key_len | ref  | rows | filtered | Extra                                                           |
+----+-------------+-------+------------+-------------+----------------------------------+----------------------------------+---------+------+------+----------+-----------------------------------------------------------------+
|  1 | SIMPLE      | score | NULL       | index_merge | fk_score_student,fk_score_course | fk_score_student,fk_score_course | 4,4     | NULL |    4 |      100 | Using sort_union(fk_score_student,fk_score_course); Using where |
+----+-------------+-------+------------+-------------+----------------------------------+----------------------------------+---------+------+------+----------+-----------------------------------------------------------------+
```



## 4.等值传递

Mysql优化器会将In()列表复制应用到关联的各个表中。



## 5.并行执行

mysql 无法并行执行SQL查询。



## 6. 哈希关联

innodb 不支持哈希索引



## 7.松散索引扫描（TODO）



## 8.最大值和最小值优化

在需要取最大/最小值的字段上创建索引，然后在查询语句中加入“use index”语句强制使用索引，当MySQL读到第一条满足条件的记录的时候就是我们需要找的最大/最小值了。

优化示例：

优化前：

 ![img](D:\GitHub\nouk-doc\mysql\images\747151-20170213210355254-864113715.png)

优化后：

![img](D:\GitHub\nouk-doc\mysql\images\747151-20170213210455425-860352610.png)





# 查询优化器的提示（hint）



##   1. HIGH_PRIORITY 、 LOW_PRIORITY

 HIGH_PRIORITY: 提示mysql该语句优先执行
 LOW_PRIORITY： 提示mysql该语句处于等待执行， 有可能出现一直等待状态
 这两个提示针对表级锁有用， 不要使用在Innodb中， 由于优先级排序操作， 禁用并发， 严重影响性能。
 这两个提示只是控制mysql访问数据的队列顺序， 不会影响具体的请求对资源的控制



## 2. DELAYED

当DELAYED插入操作到达的时候， 服务器把数据行放入一个队列中，并立即给客户端返回一个状态信息，这样客户端就可以在数据表被真正地插入记录之前继续进行操作了。

问题：

- 不能使用LAST_INSERT_ID()来获取AUTO_INCREMENT值。AUTO_INCREMENT值可能由语句生成。

- 对于SELECT语句，DELAYED行不可见，直到这些行确实被插入了为止。

- DELAYED会有丢失数据的风险，因为不是直接写入表中。可能因为mysql意外终止。
- DELAYED在从属复制服务器中被忽略了，因为DELAYED不会在从属服务器中产生与主服务器不一样的数据。



## 3.STRAIGHT JOIN

STRAIGHT_JOIN就是在内连接中使用，而强制使用左表来当驱动表，所以这个特性可以用于一些调优，强制改变mysql的优化器选择的执行计划。



## 4.SQL_SMALL_RESULT 、 SQL_BIG_RESULT

 SQL_SMALL_RESULT: 提示优化器结果集比较小， 可以将结果放在内存中的索引临时表， 以避免排序和IO操作。
 SQL_BIG_RESULT：提示优化器结果集比较大， 建议使用磁盘临时表做排序操作。



## 5.SQL_CACHE 、 SQL_NO_CACHE

 查询结果是否放入查询缓存



## 6.FOR UPDATE 、 LOCK IN SHARE MODE

- 共享锁（S）：select * from table_name where ... lock in share mode;
- 排他锁（S）：select * from table_name where ... for update;

> for update是在数据库中上锁用的，可以为数据库中的行上一个排它锁。当一个事务的操作未完成时候，其他事务可以读取但是不能写入或更新。



## 7.USE INDEX 、 IGNORE INDEX 、 FORCE INDEX

提示优化器使用不使用索引
USE INDEX 、 FORCE INDEX 使用基本一致，FORCE INDEX 更加强调全表扫描代价更大。



# 总结

1. mysql整体结构和查询的工作路程。
2. join时mysql是怎么去实现的。
3. 优化器存在的缺陷。
4. 优化器查询提示的一些用法。



> 判断思路：
>
> 1、数据量大时要保证单表能瞒住觉大部分的过滤条件。如果单表不能行也要保证驱动表的大小控制。
>
> 2、数据量大时order by 、group by 等一定要命中[索引](https://www.cnblogs.com/dongguacai/p/7241860.html)，或者减少查询的数量。
>
> 3、min、max 一定要中索引。
>
> 4、sum 可以在外部计算（每次insert 和 update时在redis或者别的地方计算`记得要保证数据的可恢复性`）。
>
> 4、数据量大时如果业务页面上的内容显示过多子集数据一定要在业务上避免。
>
> 5、数据量大时如果查询的数据要计算，可以在插入或修改时前计算完成。
>
> 6、使用join时可以用[inner join](https://blog.csdn.net/LJFPHP/article/details/88635755)(之前有说过mysql join的链接方式)。
>
> 7、访问平凡的数据可以缓存、最好时能做到静态化数据。
>
> 8、一张表的索引数量和服务器配置有关系，大概5个比较正常。包括 create_date和外键索引组成。
>
> 9、在业务条件筛选多的时候是不是都要添加索引，可以在条件筛选时查询出具体的外键辅助查询。





# 实例分析



```sql
SELECT
	a.id AS "id",
	a.message AS "message",
	a. STATUS AS "status",
	a.create_date AS "createDate",
	a.send_type AS "sendType",
	a.real_time AS "realTime",
	a. NO AS "no",
	a.serial_number AS "serialNumber",
	g.id AS "gift.id",
	g. NO AS "gift.no",
	g. NAME AS "gift.name",
	g.icon_path AS "gift.iconPath",
	g.claer_photo_path AS "gift.claerPhotoPath",
	g.is_invalid AS "gift.isInvalid",
	g.price AS "gift.price",
	s.id AS "sender.id",
	s. NO AS "sender.no",
	s.login_name AS "sender.loginName",
	s.sex AS "sender.sex",
	s.birthday AS "sender.birthday",
	s.country AS "sender.country",
	s.province AS "sender.province",
	s.city AS "sender.city",
	s.eng_name AS "sender.engName",
	s. STATUS AS "sender.status",
	s.icon_path AS "sender.iconPath",
	s. ONLINE AS "sender.online",
	t.id AS "target.id",
	t. NO AS "target.no",
	t. ONLINE AS "target.online",
	t.login_name AS "target.loginName",
	t.sex AS "target.sex",
	t.birthday AS "target.birthday",
	t.country AS "target.country",
	t.province AS "target.province",
	t.city AS "target.city",
	t.eng_name AS "target.engName",
	t. STATUS AS "target.status",
	t.icon_path AS "target.iconPath",
	sp.id AS "sender.publisher.id",
	sp. NAME AS "sender.publisher.name",
	sp. NO AS "sender.publisher.no",
	sp.country AS "sender.publisher.country",
	sp.province AS "sender.publisher.province",
	sp.city AS "sender.publisher.city",
	sp.address AS "sender.publisher.address",
	sp.country_code AS "sender.publisher.countryCode",
	sp. STATUS AS "sender.publisher.status",
	su. NO AS "sender.followUser.no",
	su. NAME AS "sender.followUser.name"
FROM
	am_gift_hist a
LEFT JOIN am_member s ON a.sender_id = s.id
LEFT JOIN am_member t ON a.target_id = t.id
LEFT JOIN am_publisher_info sp ON s.publisher_id = sp.id
LEFT JOIN am_gift g ON a.gift_id = g.id
LEFT JOIN sys_user su ON su.id = s.follow_user_id
WHERE
	1 = 1
AND s.del_flag = '0'
AND t.del_flag = '0'
AND a.del_flag = '0'
AND su.id = 'b845cabd1b07475e982e149a8a7c2e2e'
AND s.sex = '2'
AND a.create_date >= '2020-11-13 00:00:00'
AND a.create_date <= '2020-11-13 23:59:59'
ORDER BY
	a.create_date DESC
LIMIT 0,
 10
```



```sql
SELECT
	f.no as "female.no",
	f.eng_name as "female.engName",
	f.icon_path as "female.iconPath",
	f.rect_icon_path as "female.rectIconPath",	
	a.id,
	a.serial,		
	a.timing_id AS "timing.id",
	a.sender_id AS "sender.id",
	a.male_id AS "male.id",
	a.female_id AS "female.id",
	a.consumption_type,
	a.`type`,
	a.content,
	a.status,
	a.create_date,
	a.create_timestamp,
	a.update_date,
	a.del_flag
FROM
	am_chat_hist a
	left join am_member f on f.id = a.female_id
	left join am_member m ON m.id = a.male_id
WHERE
	a.sender_id = a.female_id
	and a.`del_flag` = #{delFlag}
	AND m.id = #{male.id}
	AND a.`type` = #{type}
	AND a.`status` = #{status}
	AND a.create_date &gt;=#{createDate}
ORDER BY a.create_timestamp DESC
```


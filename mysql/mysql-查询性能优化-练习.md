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



#### 关联子查询的优势例子



```sql
explain select distinct cid from class inner join student_1 on (class_id = cid);
```

在使用distinct和group by 时，通常需要产生临时中间表。



```sql
explain select cid from class where exists ( select 1 from student_1 where class_id = cid);
```

而第二条sql没有产生零食表的过程，而且只要存在就返回。





# 练习

1.找到日常开发中的sql使用explain分析，并使用伪代码描述执行过程。










# 构造数据

```sql
create TABLE `user` (
  `id` int(11) NOT NULL,
  `last_name` varchar(45) DEFAULT NULL,
  `first_name` varchar(45) DEFAULT NULL,
  `sex` set('M','F')  DEFAULT NULL,
  `age` tinyint(1) DEFAULT NULL,
  `phone` varchar(11) DEFAULT NULL,
  `address` varchar(45) DEFAULT NULL,
  `password` varchar(45) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_last_first_name_age` (`last_name`,`first_name`,`age`) USING BTREE,
  KEY `idx_phone` (`phone`) USING BTREE,
  KEY `idx_create_time` (`create_time`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```

```sql
drop PROCEDURE test_insert;
DELIMITER $$
CREATE PROCEDURE test_insert()
 begin 
 declare num int;  
set num=0;
        while num < 1 do
            insert into user(last_name, first_name, sex, age, create_time) values("last_name"+num,"first_name"+num,FLOOR(RAND() * 6)+1,FLOOR(RAND() * 100)+1,now());
            set num=num+1;
        end while;
END $$
   
call test_insert();
```

# 为什么查询速度会慢

一个查询由多个子任务组成。每个子任务都会消耗一定时间，优化查询，实际上要优化起子任务，要么消除其中一些子任务，要么减少子任务的执行次数。



# 慢查询基础：优化访问

查询性能低下最基础的原因是访问的数据大多。

1. 请求了不需要的数据。
2. mysql是否在扫描额外的记录。

最简单衡量查询开销的指标：

	* 响应时间
	* 扫描的行数
	* 返回的行数

# 查询执行的基础

mysql执行一个查询的过程



![img](https://img-blog.csdnimg.cn/20191204191659238.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpZ2h0ZXI2MTM=,size_16,color_FFFFFF,t_70)

## client/server 通信方式

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

## 查询缓存

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



## 解析器

1.词法分析

扫描字符流，根据构词规则识别单个单词。生成多个token，分别为关键字和非关键字。

2.语法分析

在词法分析的基础上将单词序列组成语法短语，最后生成语法树。



## 预处理器



根据一些mysql规则进一步检查解析树是否合法。如检查查询的表名、列名是否正确，是否有表的权限等。

![img](https://img2018.cnblogs.com/blog/1385831/201902/1385831-20190219101803343-1749289173.png)

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

## 查询优化器

```sql
select count(*) from students;
show status like 'last_query_cost'; -- 查询最后一次的查询成本
```

mysql会更具：每个表或者索引页面个数、索引基数、索引和数据行的常熟、索引分布情况得出查询的成本。

然后把成本最低的执行sql传给执行引擎。














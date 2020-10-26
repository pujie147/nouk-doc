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


CREATE TABLE `locks` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增ID',
  `uid` bigint(20) unsigned NOT NULL  COMMENT '学号ID',
  `name` varchar(64) NOT NULL DEFAULT '' COMMENT '名字',
    `phone` varchar(16) NOT NULL DEFAULT '13000000000' COMMENT '手机号',
  `sex` tinyint(2) NOT NULL DEFAULT 0 COMMENT '性别',
  
  PRIMARY KEY (`id`),
  UNIQUE KEY `uid` (`uid`),
  KEY `name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COMMENT='锁测试表';


insert into locks values (1,1,"Tom",1,0);
insert into locks values (5,5,"Tom",1,0);
insert into locks values (10,10,"Tom",1,0);
insert into locks values (15,15,"Tom",1,0);
insert into locks values (20,20,"Tom",1,0);


select @@tx_isolation;
SET AUTOCOMMIT=0;


+----+-----+------+-------+-----+
| id | uid | name | phone | sex |
+----+-----+------+-------+-----+
|  1 |   1 | Tom  | 1     |   0 |
|  5 |   5 | Tom  | 1     |   0 |
|  6 |   6 | Tom  | 1     |   0 |
| 10 |  10 | Tom  | 1     |   0 |
| 15 |  15 | Tom  | 1     |   0 |
| 16 |   2 | Tom  | 1     |   0 |
| 17 |   3 | Tom  | 1     |   0 |
| 20 |  20 | Tom  | 1     |   0 |
+----+-----+------+-------+-----+


1. 主键

A事务
select * from locks where id = 5 for update;  

锁定的情况：
select * from locks where id = 5 for update;
select * from locks where id < 5 for update;
select * from locks where id != 5 for update;

未锁定情况:
select * from locks where id = 1 for update;
insert into locks values (4,4,"Tom",1,0);

--------------------------------------------------------


A事务
select * from locks where id > 1 and id <= 5 for update;

锁定情况：
select * from locks where id = 4 for update; 
select * from locks where id = 5 for update;
select * from locks where id = 6 for update; 
select * from locks where id < 10 for update; 
select * from locks where id != 10 for update;
insert into locks values (4,4,"Tom",1,0);
insert into locks values (6,6,"Tom",1,0);


未锁定情况:
select * from locks where id = 10 for update;
select * from locks where id = 1 for update; 

---------------------------------------------

insert into locks values (12,12,"Tom",1,0);

锁定情况
insert into locks values (12,12,"Tom",1,0);
select * from locks where id < 15 for update;
select * from locks where id >= 12 for update;

未锁定情况
select * from locks where id = 10 for update;
select * from locks where id = 15 for update;
select * from locks where id >= 15 for update;
select * from locks where id > 15 for update;
select * from locks where id > 12 for update;
insert into locks values (13,13,"Tom",1,0);


2. 唯一索引
3. 非唯一索引
4. 非索引
select * from locks where id = 1 for update;



唯一索引 --  等值查询 -- 行锁
唯一索引 --  范围查询 -- 间隙锁
唯一索引 --  插入操作 -- 行锁

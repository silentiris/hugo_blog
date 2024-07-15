+++
title = "关于mysql索引的一些探索"
date = "2024-03-12T10:59:48+08:00"
tags = []
slug = "some-exploration-of-MySQL-indexes"
+++

在复习索引的时候看了小林coding的博客，造了一些数据数据，方法如下：

```mysql
CREATE TABLE person(
    id int(10) NOT NULL AUTO_INCREMENT PRIMARY KEY comment '主键',
    person_id tinyint not null comment '用户id',
    person_name VARCHAR(200) comment '用户名称',
    gmt_create datetime comment '创建时间',
    gmt_modified datetime comment '修改时间'
) comment '人员信息表';

insert into person values(1, 1,'user_1', NOW(), now());

select (@i:=@i+1) as rownum, person_name from person, (select @i:=100) as init; 
set @i=1;

insert into person(id, person_id, person_name, gmt_create, gmt_modified)
select @i:=@i+1,
left(rand()*10,10) as person_id,
concat('user_',@i%2048),
date_add(gmt_create,interval + @i*cast(rand()*100 as signed) SECOND),
date_add(date_add(gmt_modified,interval +@i*cast(rand()*100 as signed) SECOND), interval + cast(rand()*1000000 as signed) SECOND)
from person;

# 可以成倍的增加数据，用这种方法制造了200w行的记录
mysql> select count(*) from person ;
+----------+
| count(*) |
+----------+
|  2097152 |
+----------+
```

**问题一：explain是如何判断rows：**

```mysql
mysql> explain select count(*) from person ;
+----+-------------+--------+------------+-------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+--------+------------+-------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | index | NULL          | name | 803     | NULL | 3581169 |   100.00 | Using index |
+----+-------------+--------+------------+-------+---------------+------+---------+------+---------+----------+-------------+
```

既然只有209w行记录，那为什么会预估需要扫358w行呢

这个问题很快得到了解答：使用 `ANALYZE TABLE `让优化器重新分析一下表就可以了。重新分析过后很快得到了正确答案。

**问题二：关于索引失效的问题**

```mysql
mysql> show keys from person\G;
*************************** 1. row ***************************
        Table: person
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 3258015
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
      Visible: YES
   Expression: NULL
*************************** 2. row ***************************
        Table: person
   Non_unique: 1
     Key_name: name
 Seq_in_index: 1
  Column_name: person_name
    Collation: A
  Cardinality: 2049
     Sub_part: NULL
       Packed: NULL
         Null: YES
   Index_type: BTREE
      Comment: 
Index_comment: 
      Visible: YES
   Expression: NULL
```

在name列上建了一个索引。

- 建索引前：

```mysql
mysql> select * from person where person_name like 'user_1915%';
1024 rows in set (1.66 sec)

mysql> select * from person where person_name like '%user_1915%';
1024 rows in set (1.76 sec)

mysql> select * from person where person_name = 'user_1915';
1024 rows in set (1.55 sec)


#可以看得出，四种查询耗时几乎一样，也符合理解：此时是全表扫描，大部分耗时用在了扫描身上，所以也能看出来尽量减少扫描是很重要的
mysql> explain select * from person where person_name = 'user_1915';
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 3581169 |    10.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
```

- 建索引后：

```mysql
#在person_name字段建立索引
mysql> create index name on person (person_name);

#重复上面的实验
mysql> select * from person where person_name like 'user_1915%';
1024 rows in set (1.61 sec)
mysql> explain select * from person where person_name like 'user_1915%';
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ALL  | name          | NULL | NULL    | NULL | 3581169 |    50.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+

mysql> select * from person where person_name like '%user_1915%';
1024 rows in set (1.89 sec)
mysql> explain select * from person where person_name like '%user_1915%';
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 3581169 |    11.11 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+

mysql> select * from person where person_name = 'user_1915';
1024 rows in set (0.01 sec)
mysql> explain select * from person where person_name = 'user_1915';
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | person | NULL       | ref  | name          | name | 803     | const | 1024 |   100.00 | NULL  |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-------+

#可以看得出，只有第三句使用了索引，ref指使用非唯一索引，这才是正常的。可是问题在于为什么第一句不会走索引呢。理论来讲第一个不应该使用name索引吗？

番外：
mysql> select * from person where person_name like 'user_1915%';
➜  ~ docker exec -it mysql bash
Error response from daemon: container 467bcb6435a831ac52c9b0c72391e5a45971a81c29151a8e4628c15f9b240180 is not running
不知道为什么，测试的时候查着查着mysql就g了
更新：破案了，是内存莫名其妙占满了，reboot就行了，但具体问题还没找出来
```

在提出上面的问题后问了许多学长，更奇怪的是使用了force index后速度也很慢，所以这并不完全是优化器的锅，更深层的原因需要figuare out为什么遍历索引的时候会如此慢。

这个表主要有这样的特征：所有的name数据都是以`user_`开头，所以一个猜测是可能因为有共同的前缀导致以上问题，但其实并不能完全说通，干扰优化器可以理解，但force之后还这么慢就不能理解了（force之后仅仅快了300-400ms左右）

其实衍生出来的还有一些问题：

- ` select * from person where person_name like 'user_1915';`为什么不会直接被优化成`=`呢。

    随后换了一张表进行测试，600w数据，完全随机数据。

    得到的结果是这样的：

    ```mysql
    mysql> explain select * from test where name like 'Zhu Lan';
    +----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
    | id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                 |
    +----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
    |  1 | SIMPLE      | test  | NULL       | range | name          | name | 258     | NULL | 1046 |   100.00 | Using index condition |
    +----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
    1 row in set, 1 warning (0.01 sec)
    
    mysql> explain select * from test where name = 'Zhu Lan';
    +----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
    | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
    +----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
    |  1 | SIMPLE      | test  | NULL       | ref  | name          | name | 258     | const | 1046 |   100.00 | NULL  |
    +----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)
    ```

    发现正常情况下也会不一样，like还是走的range索引，而且不会有索引下推。

在有上面的猜想之后，新建了一个600w完全随机数据的表，重复了上面的实验，完全符合预期。所以目前上面猜测可能性还是很大的，目前唯一的问题是强制使用索引后为什么依然速度很差。希望有一天能update解决这个问题，先写一下实验经过记录一下。

---

分割线，2024-07-15更新

在看[一个关于mysql慢查询博客](https://mp.weixin.qq.com/s/sPO-6ULwIfUexLY3V4acBg)的时候突然想到了这个问题，突然发现之前没有考虑回表查询的损耗。因为查询的是`*`，而且区分度不明显，可能会产生大量回表查询，所以不如直接全表扫描。
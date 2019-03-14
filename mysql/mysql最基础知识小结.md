title: mysql最基础知识小结
tag: mysql
---

本文介绍关于数据库的最最最最基本的一些语法知识，如果这些都不熟悉，建议多多练习，因为后续的文章会比较深入原理。
<!-- more -->

一、DDL语句

1、创建数据库：`create database dbname;`

2、删除数据库：`drop database dbname;`

3、创建表：`create table tname;`

4、删除表：`drop table tname;`

5、修改表：略，懒得看

---


二、DML语句

- 插入：
```sql
insert into table(字段1，字段2，...) values (value1,value2,...) , (value3,value4,..)
```


- 更新：
```sql
update table set 字段=value where ...
```


- 删除：
```sql
delete from table where ...
```
这里要注意下`delete`和`truncate`以及`drop`三者的区别，下篇文章详解。

- 单表查询：
```sql
select 字段 from table
```


- 连表查询方式1：
```sql
select 别名1.字段,别名2.字段 from table1 别名1,table2 别名2 where ...
```


- 连表查询方式2：
```sql
select 别名1.字段,别名2.字段 from table1 别名1 join table2 别名2 on ...
```

这是全连接，这里就要了解一下笛卡儿积，简单来说，最后行数是左边表的函数乘以右边表的行数。详细的可以自行google.

- 查询不重复的记录：
```sql
select distinct 字段 from table ...
```


- 排序：默认是升序 
```sql
select 字段 from table where ... order by ... asc/desc
```


- limit：主要用于分页 
```sql
select * from table where ... order by ... asc/desc limit 起始偏移位置，显示条数
```


- 聚合：
```sql
select count(*)/avg(..)/sum(...)/max(...)/min(...) from table group by ... having ....
```
注意这里的`having`和`where`的区别：`where`是对表结果进行筛选，`having` 是对查询结果进行筛选，与`group by` 合用

- 左连接和右连接

左连接意思就是左表中的记录都在，右表没有匹配项就以null显示。记录数等于左表的行数。

右连接与之同理，尽量转为左连接做。


- 子查询：
```sql
select * from table where ... in (select ....)
```
所谓子查询就是根据另一个`select`的结果再进行筛选，常用的是in,not in,=,!=,exits,not exits


- union
主要用于两张表中的数据的合并：
```sql
select 字段 from table1 union all select 字段 from table2
```
要想不重复用`union`






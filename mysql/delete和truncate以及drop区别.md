title: delete和truncate以及drop区别
tag: mysql
---

这个题目我自己也被问过，这里简单整理一下。
<!-- more -->

先来个总结：
- `drop`直接删掉表；
- `truncate`删除的是表中的数据，再插入数据时自增长的数据id又重新从1开始；
- `delete`删除表中数据，可以在后面添加`where`字句。

## 日志是否记录

`DELETE`语句执行删除操作的过程是每次从表中删除一行，并且同时将该行的删除操作作为事务记录在日志中保存以便进行进行回滚操作。

`TRUNCATE TABLE` 则一次性地从表中删除所有的数据并不把单独的删除操作记录记入日志保存，删除行是不能恢复的。并且在删除的过程中不会激活与表有关的删除触发器。执行速度快。


## 是否可以回滚

 `delete` 这个操作会被放到 `rollback segment` 中,事务提交后才生效。`truncate`、`drop`是DLL（data define language),操作立即生效，原数据不放到 `rollback segment` 中，不能回滚.
 
 所以在没有备份情况下，谨慎使用 `drop` 与 `truncate`。
 
 

## 表和索引占的空间

当表被 `TRUNCATE` 后，这个表和索引所占用的空间会恢复到初始大小。

而 `DELETE` 操作不会减少表或索引所占用的空间。

`drop`语句将表所占用的空间全释放掉。

`TRUNCATE` 和 `DELETE` 只删除数据，而 `DROP` 则删除整个表（结构和数据）

所以从干净程度，`drop` > `truncate` > `delete`

ok，差不多了。
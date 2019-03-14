title: MySQL调优
tag: mysql
---

本文介绍最基本的sql调优手段。
<!-- more -->

## 根据慢日志定位慢查询sql

```sql
<!--这里是用模糊查询查出关于查询的一些配置项-->
show variables like '%query%'
```

我们关注`slow_query_log`：`OFF`，表示慢查询处于关闭状态。关注`long_query_time`：超出这个时间就是慢查询，记录到`slow_query_log_file`文件中。

```sql
show status like '%slow_queries%'
```

这一句作用是统计慢查询的数量。

如何打开慢查询呢？

```sql
<!--打开慢查询-->
set global slow_query_log = on;
<!--慢查询的标准是1秒-->
set global long_query_time = 1;
```

注意要重启一下客户端。或者在配置文件中设置，重启服务端就永久保留了。

## explain分析慢日志

上一步时打开慢查询日志。下面要进行分析。

```sql
explain select ...
```

对这个命令进行分析。有两个关键字段：

`type`：表示mysql找到数据行的方式，下面的顺序是由快到慢：

`system`>`const`> `eq_ref`>`ref`>`fulltext`>`ref_or_null`>`index_merge`>`unique_subquery`>`index_subquery`>`range`>`index`>`all`

**其中`index`和`all`为全表扫描。说明需要优化。**

`extra`：
- `using_filesort`：表示MySQL会对结果使用一个外部索引排序，而不是从表里按索引次序读到相关内容。可能在内存或者磁盘上进行排序。MqSQL中无法利用索引完成的排序操作称为“文件排序”
- `using temporary`：表示MySQL在对查询结果排序时使用临时表。常见于排序order by 和分组查询 group by。

**当`extra`中出现以上两项意味着MYSQL根本不能使用索引，效率会受到重大影响，应尽可能对此进行优化。**

## 修改sql或者尽量让sql走索引

上一步分析完之后，就要采取一定的措施来修正。

如果是没有加索引，可以对其加上索引。`extra`就会变成`using index`，表示走了索引。

## 索引是越多越好吗

- 数据量小的表不需要建立索引，建立会增加额外的索引开销
- 数据变更需要维护索引，因此更多的索引意味着更多的维护成本
- 更多的索引意味着也需要更多的空间

可以理解为，一个几页的宣传手册

- 对于几页的宣传手册我们还需要建立一个目录吗？
- 变更这个小的宣传手册里面的章节还要修改目录不是更烦吗？
- 一个宣传手册内容就两页，结果你的目录就占了一页，这合理吗？
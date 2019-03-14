title: mysql面试高频理论知识
tag: mysql
---

整理一些面试题，简单看看。
<!-- more -->

目录

1. 数据库三范式
2. 事务
3. mysql数据库默认最大连接数
4. 分页
5. 触发器
6. 存储过程
7. 用jdbc怎么调用存储过程？
8. 对jdbc的理解
9. 写一个简单的jdbc的程序。写一个访问oracle数据的jdbc程序
10. JDBC中的PreparedStatement相比Statement的好处
11. 数据库连接池作用
12. 选择合适的存储引擎
13. 数据库优化-索引
14. 数据库优化-分表
15. 数据库优化-读写分离
16. 数据库优化-缓存
17. 数据库优化-sql语句优化的技巧
18. jdbc批量插入几百万数据怎么实现
19. 聚簇索引和非聚簇索引
20. sql注入问题
21. mysql悲观锁和乐观锁

## 1. 数据库三范式

### 1.1 范式是什么

范式就是规范，要满足第二范式必须先满足第一范式，要满足第三范式，必须要先满足第二范式。

- 1NF(第一范式)：列数据不可分割，即一列不能有多个值
- 2NF(第二范式)：主键(每一行都有唯一标识)
- 3NF(第三范式)：外键(表中不包含已在其他表中包含的非主关键信息)

### 1.2 反三范式

反三范式：有时为了效率，可以设置重复或者推导出的字段，例如：订单总价格订单项的单价，这个订单总价虽然可以由订单项计算出来，但是当订单数目庞大时，效率比较低，所以订单的总价这个字段是必要的。

## 2. 事务

### 2.1 含义

事务时并发控制的单位，是用户定义的一个操作序列，要么都做，要么都不做，是不可分割的工作单位。


## 3. mysql数据库默认最大连接数

### 3.1 为什么需要最大连接数

特定服务器上的数据库只能支持一定数目同时连接，这时需要我们设置最大连接数（最多同时服务多少连接）。在数据库安装时会有一个默认的最大连接数。

> `my.ini`中`max_connections=100`

## 4. 分页

### 4.1 为什么需要分页？

在很多数据时，不可能完全显示数据。进行分段显示.

### 4.2 mysql如何分页


```sql
String sql = 
"select * from students order by id limit " + pageSize*(pageNumber-1) + "," + pageSize;
```

### 4.3 oracle分页

是使用了三层嵌套查询。
```swl
String sql = 
	 "select * from " +  
	 (select *,rownum rid from (select * from students order by postime desc) where rid<=" + pagesize*pagenumber + ") as t" + 
	 "where t>" + pageSize*(pageNumber-1);
```

## 5. 触发器

略。

## 6. 存储过程

### 6.1 数据库存储过程具有如下优点：

* 1、存储过程只在创建时进行编译，以后每次执行存储过程都不需再重新编译，而一般 SQL 语句每执行一次就编译一次，因此使用存储过程可以大大提高数据库执行速度。

* 2、通常，复杂的业务逻辑需要多条 SQL 语句。这些语句要分别地从客户机发送到服务器，当客户机和服务器之间的操作很多时，将产生大量的网络传输。如果将这些操作放在一个存储过程中，那么客户机和服务器之间的网络传输就会大大减少，降低了网络负载。

* 3、存储过程创建一次便可以重复使用，从而可以减少数据库开发人员的工作量。

* 4、安全性高，存储过程可以屏蔽对底层数据库对象的直接访问，使用 EXECUTE 权限调用存储过程，无需拥有访问底层数据库对象的显式权限。

### 6.2 定义存储过程:


```sql
create procedure insert_Student (_name varchar(50),_age int ,out _id int)
begin
	insert into student value(null,_name,_age);
	select max(stuId) into _id from student;
end;

call insert_Student('wfz',23,@id);
select @id;
```

## 7. 用jdbc怎么调用存储过程？

> 贾琏欲执事
- 加载驱动
- 获取连接
- 设置参数
- 执行
- 释放连接
    

```java
import java.sql.CallableStatement;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.sql.Types;

public class JdbcTest {

	/**
	 * @param args
	 */
	public static void main(String[] args) {
		// TODO Auto-generated method stub
		Connection cn = null;
		CallableStatement cstmt = null;		
		try {
			//这里最好不要这么干，因为驱动名写死在程序中了
			Class.forName("com.mysql.jdbc.Driver");
			//实际项目中，这里应用DataSource数据，如果用框架，
			//这个数据源不需要我们编码创建，我们只需Datasource ds = context.lookup()
			//cn = ds.getConnection();			
			cn = DriverManager.getConnection("jdbc:mysql:///test","root","root");
			cstmt = cn.prepareCall("{call insert_Student(?,?,?)}");
			cstmt.registerOutParameter(3,Types.INTEGER);
			cstmt.setString(1, "wangwu");
			cstmt.setInt(2, 25);
			cstmt.execute();
			//get第几个，不同的数据库不一样，建议不写
			System.out.println(cstmt.getString(3));
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		finally
		{

			/*try{cstmt.close();}catch(Exception e){}
			try{cn.close();}catch(Exception e){}*/
			try {
				if(cstmt != null)
					cstmt.close();
				if(cn != null)				
					cn.close();
			} catch (SQLException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
	}
```

## 8. 对jdbc的理解

`Java database connection` java数据库连接.数据库管理系统(`mysql` `oracle`等)是很多，每个数据库管理系统支持的命令是不一样的。

Java只定义接口，让数据库厂商自己实现接口，对于我们者而言。只需要导入对应厂商开发的实现即可。然后以接口方式进行调用.(`mysql` + `mysql`驱动（实现）+`jdbc`)

## 9. 写一个简单的jdbc的程序。写一个访问oracle数据的jdbc程序

> 贾琏欲执事

1. 加载驱动(`com.mysql.jdbc.Driver,oracle.jdbc.driver.OracleDriver`)
2. 取连接(`DriverManager.getConnection(url,usernam,passord)`)
3. 设置参数  `Statement PreparedStatement `
            `cstmt.setXXX(index, value);`
4. 执行   `executeQuery executeUpdate`
5. 释放连接(是否连接要从小到大，必须放到`finnaly`)

## 10. JDBC中的PreparedStatement相比Statement的好处

**大多数我们都使用`PreparedStatement`代替`Statement`**

- 1：`PreparedStatement`是预编译的，比`Statement`速度快 
- 2：代码的可读性和可维护性

虽然用`PreparedStatement`来代替`Statement`会使代码多出几行,但这样的代码无论从可读性还是可维护性上来说.都比直接用`Statement`的代码高很多档次：


```java
stmt.executeUpdate("insert into tb_name (col1,col2,col2,col4) values
('"+var1+"','"+var2+"',"+var3+",'"+var4+"')"); 

perstmt = con.prepareStatement("insert into tb_name (col1,col2,col2,col4) values (?,?,?,?)");
perstmt.setString(1,var1);
perstmt.setString(2,var2);
perstmt.setString(3,var3);
perstmt.setString(4,var4);
perstmt.executeUpdate();
```

- 3：安全性

`PreparedStatement`可以防止`SQL`注入攻击，而`Statement`却不能。

比如说：
> String sql = "select * from tb_name where name= '"+varname+"' and passwd='"+varpasswd+"'";

如果我们把`[' or '1' = '1]`作为varpasswd传入进来.用户名随意,看看会成为什么?

> select * from tb_name = '随意' and passwd = '' or '1' = '1';

因为`'1'='1'`肯定成立，所以可以任何通过验证。

更有甚者：把`[';drop table tb_name;]`作为`varpasswd`传入进来,则：

> select * from tb_name = '随意' and passwd = '';drop table tb_name;

有些数据库是不会让你成功的，但也有很多数据库就可以使这些语句得到执行。

而如果你使用预编译语句你传入的任何内容就不会和原来的语句发生任何匹配的关系，只要全使用预编译语句你就用不着对传入的数据做任何过虑。而如果使用普通的`statement`,有可能要对`drop`等做费尽心机的判断和过虑。

## 11. 数据库连接池作用

* 1、限定数据库的个数，不会导致由于数据库连接过多导致系统运行缓慢或崩溃
* 2、数据库连接不需要每次都去创建或销毁，节约了资源
* 3、数据库连接不需要每次都去创建，响应时间更快。

## 12. 选择合适的存储引擎

在开发中，我们经常使用的存储引擎 `myisam` / `innodb`/ `memory`

> MyISAM存储引擎

如果表对事务要求不高，同时是以查询和添加为主的，我们考虑使用myisam存储引擎. 比如 bbs 中的 发帖表，回复表.

> INNODB存储引擎: 

对事务要求高，保存的数据都是重要数据，我们建议使用`INNODB`,比如订单表，账号表.

> Memory 存储

我们数据变化频繁，不需要入库，同时又频繁的查询和修改，我们考虑使用`memory`, 速度极快.

`MyISAM` 和 `INNODB`的区别(主要)

1. 事务安全 `myisam`不支持事务而`innodb`支持
2. 查询和添加速度 `myisam`不用支持事务就不用考虑同步锁，查找和添加和添加的速度快
3. 支持全文索引 `myisam`支持`innodb`不支持
4. 锁机制 `myisam`支持表锁而`innodb`支持行锁(事务)
5. 外键 `MyISAM` 不支持外键， `INNODB`支持外键. (通常不设置外键，通常是在程序中保证数据的一致)



---

# 下面是数据库的优化手段，但是只是表面，需要以后再好好探究

在项目自验项目转测试之前，在启动`mysql`数据库时开启慢查询，并且把执行慢的语句写到日志中，在运行一定时间后。通过查看日志找到慢查询语句。

```sql
show variables like '%slow%';   #查看MySQL慢查询是否开启

set global slow_query_log=ON;   #开启MySQL慢查询功能

show variables like "long_query_time";  #查看MySQL慢查询时间设置，默认10秒

set global long_query_time=5;  #修改为记录5秒内的查询

select sleep(6);  #测试MySQL慢查询

show variables like "%slow%";  #查看MySQL慢查询日志路径

show global status like '%slow%';  #查看MySQL慢查询状态

或者

vi  /etc/my.cnf  #编辑，在[mysqld]段添加以下代码

slow-query-log = on  #开启MySQL慢查询功能

slow_query_log_file =  /var/run/mysqld/mysqld-slow.log #设置MySQL慢查询日志路径

long_query_time = 5  #修改为记录5秒内的查询，默认不设置此参数为记录10秒内的查询

log-queries-not-using-indexes = on  #记录未使用索引的查询

:wq! #保存退出

service mysqld restart #重启MySQL服务
```


## 13. 数据库优化-索引

### 13.1 索引的概念

索引（`Index`）是帮助`DBMS`高效获取数据的数据结构。

### 13.2 索引有哪些

> 分类：普通索引/唯一索引/主键索引/全文索引

* 普通索引:允许重复的值出现

* 唯一索引:除了不能有重复的记录外，其它和普通索引一样(用户名、用户身份证、email,tel)

* 主键索引：是随着设定主键而创建的，也就是把某个列设为主键的时候，数据库就会給改列创建索引。这就是主键索引.唯一且没有null值

* 全文索引:用来对表中的文本域(`char`，`varchar`，`text`)进行索引， 全文索引针对`MyIsam`
`explain select * from articles where match(title,body) against(‘database’);`【会使用全文索引】


### 13.3 使用索引的注意事项

> 索引弊端
1. 占用磁盘空间。
2. 对`dml`(插入、修改、删除)操作有影响，变慢。

> 使用场景：
1. 肯定在`where`条件经常使用,如果不做查询就没有意义
2. 该字段的内容不是唯一的几个值(sex) 
3. 字段内容不是频繁变化.

> 注意事项

1. 对于创建的多列索引（复合索引），不是使用的第一部分就不会使用索引。

```sql
alter table dept add index my_ind (dname,loc); // dname 左边的列,loc就是右边的列
explain select * from dept where dname='aaa'\G 会使用到索引
explain select * from dept where loc='aaa'\G 就不会使用到索引
```


2. 对于使用`like`的查询，查询如果是`%aaa`不会使用到索引而`aaa%`会使用到索引。

```sql
explain select * from dept where dname like '%aaa'\G不能使用索引
explain select * from dept where dname like 'aaa%'\G使用索引.
```

所以在`like`查询时，‘关键字’的最前面不能使用`%` 或者 `_`这样的字符，如果一定要前面有变化的值，则考虑使用 全文索引->sphinx.

3. 索引列排序

`MySQL`查询只使用一个索引，因此如果`where`子句中已经使用了索引的话，那么`order by`中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作；尽量不要包含多个列的排序，如果需要最好给这些列创建复合索引。
   

4. 如果列类型是字符串，那一定要在条件中将数据使用引号引用起来。否则不使用索引。

```sql
expain select * from dept where dname=’111’;
expain select * from dept where dname=111;（数值自动转字符串）
expain select * from dept where dname=qqq;报错
```

也就是，如果列是字符串类型，无论是不是字符串数字就一定要用 ‘’ 把它包括起来.

5. 如果`mysql`估计使用全表扫描要比使用索引快，则不使用索引。
   表里面只有一条记录

6. 索引不会包含有`NULL`值的列

只要列中包含有`NULL`值都将不会被包含在`MySQL`索引中，复合索引中只要有一列含有`NULL`值，那么这一列对于此复合索引就是无效的。所以我们在数据库设计时不要让字段的默认值为`NULL`。

7. 使用短索引

对串列进行索引，如果可能应该指定一个前缀长度。例如，如果有一个`CHAR(255)`的列，如果在前10个或20个字符内，多数值是惟一的，那么就不要对整个列进行索引。短索引不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。

8. 不要在列上进行运算，不使用`NOT IN`和`<>`操作，不支持正则表达式。


## 14. 数据库优化-分表

分表分为水平(按行)分表和垂直(按列)分表

**水平分表情形：**

根据经验，`Mysql`表数据一般达到百万级别，查询效率会很低，容易造成表锁，甚至堆积很多连接，直接挂掉；水平分表能够很大程度较少这些压力。

**垂直分表情形：**

如果一张表中某个字段值非常多(长文本、二进制等)，而且只有在很少的情况下会查询。这时候就可以把字段多个单独放到一个表，通过外键关联起来。考试详情，一般我们只关注分数，不关注详情。

**水平分表策略：**

> 1.按时间分表

这种分表方式有一定的局限性，当数据有较强的实效性，如微博发送记录、微信消息记录等，这种数据很少有用户会查询几个月前的数据，如需要就可以按月分表。

> 2.按区间范围分表

一般在有严格的自增id需求上，如按照`user_id`水平分表：

```sql
table_1  user_id从1~100w  
table_2  user_id从101~200w 
table_3  user_id从201~300w
```

> 3.hash分表

通过一个原始目标的ID或者名称通过一定的`hash`算法计算出数据存储表的表名，然后访问相应的表。


## 15. 数据库优化-读写分离

一台数据库支持的最大并发连接数是有限的，如果用户并发访问太多。一台服务器满足不要要求是就可以集群处理。Mysql的集群处理技术最常用的就是读写分离。	

**主从同步**

数据库最终会把数据持久化到磁盘，如果集群必须确保每个数据库服务器的数据是一直的。**能改变数据库数据的操作都往主数据库去写，而其他的数据库从主数据库上同步数据。**

**读写分离**
   
使用负载均衡来实现写的操作都往主数据去，而读的操作往从服务器去。


## 16. 数据库优化-缓存

**什么是缓存**

在持久层(`dao`)和数据库(`db`)之间添加一个缓存层，如果用户访问的数据已经缓存起来时，在用户访问时直接从缓存中获取，不用访问数据库。而缓存是在操作内存级，访问速度快。

**作用**

减少数据库服务器压力，减少访问时间。

**Java中常用的缓存有**
   
1. `hibernate`的二级缓存。该缓存不能完成分布式缓存。
2. 可以使用`redis`(`memcahe`等)来作为中央缓存。对缓存的数据进行集中处理



## 17. 数据库优化-sql语句优化的技巧

### DDL优化

1. 通过禁用索引来提供导入数据性能，这个操作主要针对现有数据库的表追加数据

```sql
//去除键
alter table test3 DISABLE keys;
//批量插入数据
insert into test3 ***
//恢复键
alter table test3 ENABLE keys;
```

2. 关闭唯一校验

```sql
set unique_checks=0  关闭
set unique_checks=1  开启
```


3. 修改事务提交方式(导入)（变多次提交为一次）

```sql
set autocommit=0   关闭
//批量插入
set autocommit=1   开启
```



### DML优化（变多次提交为一次）

```sql
insert into test values(1,2);
insert into test values(1,3);
insert into test values(1,4);
//合并多条为一条
insert into test values(1,2),(1,3),(1,4)
```


### DQL优化

> Order by优化

1. 多用索引排序
2. 普通结果排序（非索引排序）Filesort
   
> group by优化
      
使用order by null,取消默认排序


等等等等...

## 18. jdbc批量插入几百万数据怎么实现

1、变多次提交为一次
2、使用批量操作
3、像这样的批量插入操作能不使用代码操作就不使用，可以使用存储过程来实现

![image](http://bloghello.oursnail.cn/jdbc%E6%8F%92%E5%85%A5%E7%99%BE%E4%B8%87%E6%95%B0%E6%8D%AE%E4%BC%98%E5%8C%96.png)

mysql优化手段介绍到这里。
---


## 19. 聚簇索引和非聚簇索引

索引分为聚簇索引和非聚簇索引。

**“聚簇索引”**

以一本英文课本为例，要找第8课，直接翻书，若先翻到第5课，则往后翻，再翻到第10课，则又往前翻。这本书本身就是一个索引，即“聚簇索引”。

**“非聚簇索引”**

如果要找"fire”这个单词，会翻到书后面的附录，这个附录是按字母排序的，找到F字母那一块，再找到"fire”，对应的会是它在第几课。这个附录，为“非聚簇索引”。

由此可见，聚簇索引，索引的顺序就是数据存放的顺序，所以，很容易理解，一张数据表只能有一个聚簇索引。

聚簇索引要比非聚簇索引查询效率高很多，特别是范围查询的时候。所以，至于聚簇索引到底应该为主键，还是其他字段，这个可以再讨论。

### 1、MYSQL的索引

mysql中，不同的存储引擎对索引的实现方式不同，大致说下`MyISAM`和`InnoDB`两种存储引擎。

**MyISAM存储引擎的索引实现**

`MyISAM`的`B+Tree`的叶子节点上的`data`，并不是数据本身，而是数据存放的地址。主索引和辅助索引没啥区别，只是主索引中的key一定得是唯一的。这里的索引都是非聚簇索引。
MyISAM还采用压缩机制存储索引，比如，第一个索引为“her”，第二个索引为“here”，那么第二个索引会被存储为“3,e”，这样的缺点是同一个节点中的索引只能采用顺序查找。

**InnoDB存储引擎的索引实现**

`InnoDB` 的数据文件本身就是索引文件，`B+Tree`的叶子节点上的`data`就是数据本身，`key`为主键，这是聚簇索引。非聚簇索引，叶子节点上的data是主键 (所以聚簇索引的`key`，不能过长)。为什么存放的主键，而不是记录所在地址呢，理由相当简单，因为记录所在地址并不能保证一定不会变，但主键可以保证。
至于为什么主键通常建议使用自增id呢？

### 2.聚簇索引

聚簇索引的数据的物理存放顺序与索引顺序是一致的，即：只要索引是相邻的，那么对应的数据一定也是相邻地存放在磁盘上的。如果主键不是自增id，那么可以想 象，它会干些什么，不断地调整数据的物理地址、分页，当然也有其他一些措施来减少这些操作，但却无法彻底避免。但，如果是自增的，那就简单了，它只需要一 页一页地写，索引结构相对紧凑，磁盘碎片少，效率也高。

聚簇索引不但在检索上可以大大滴提高效率，在数据读取上也一样。比如：需要查询f~t的所有单词。

一个使用`MyISAM`的主索引，一个使用`InnoDB`的聚簇索引。两种索引的`B+Tree`检索时间一样，但读取时却有了差异。

**因为`MyISAM`的主索引并非聚簇索引**，那么他的数据的物理地址必然是凌乱的，拿到这些物理地址，按照合适的算法进行I/O读取，于是开始不停的寻道不停的旋转。聚簇索引则只需一次I/O。

**不过，如果涉及到大数据量的排序、全表扫描、count之类的操作的话，还是`MyISAM`占优势些，因为索引所占空间小，这些操作是需要在内存中完成的**。

鉴于聚簇索引的范围查询效率，很多人认为使用主键作为聚簇索引太多浪费，毕竟几乎不会使用主键进行范围查询。但若再考虑到聚簇索引的存储，就不好定论了。

## 20. sql注入问题

### 20.1 什么是sql注入

sql注入大家都不陌生，是一种常见的攻击方式，攻击者在界面的表单信息或url上输入一些奇怪的sql片段，例如“or ‘1’=’1’”这样的语句，有可能入侵参数校验不足的应用程序。所以在我们的应用中需要做一些工作，来防备这样的攻击方式。在一些安全性很高的应用中，比如银行软件，经常使用将sql语句全部替换为存储过程这样的方式，来防止sql注入，这当然是一种很安全的方式，但我们平时开发中，可能不需要这种死板的方式。

### 20.2 PrepareStatement解决SQL注入的问题

在使用`JDBC`的过程中，可以使用`PrepareStatement`进行预处理，预处理的优势就是预防绝大多数的SQL注入；而且针对多次操作数据库的情况，可以极大的提高访问数据库的效率。

那为什么它这样处理就能预防SQL注入提高安全性呢？其实是因为SQL语句在程序运行前已经进行了预编译。在程序运行时第一次操作数据库之前，SQL语句已经被数据库分析，编译和优化，对应的执行计划也会缓存下来并允许数据库以参数化的形式进行查询。当运行时动态地把参数传给`PreprareStatement`时，即使参数里有敏感字符如 or '1=1'，数据库也会作为一个参数一个字段的属性值来处理而不会作为一个SQL指令。如此，就起到了SQL注入的作用了！

### 20.3 MyBatis如何防止sql注入

`mybatis`框架作为一款半自动化的持久层框架，其sql语句都要我们自己来手动编写，这个时候当然需要防止sql注入。其实`Mybatis`的sql是一个具有“输入+输出”功能，类似于函数的结构，如下：


```sql
<select id=“getBlogById“ resultType=“Blog“ parameterType=”int”>
       select id,title,author,content 
　　　　from blog 
　　　　where id=#{id} 
</select>
```

这里，`parameterType`标示了输入的参数类型，`resultType`标示了输出的参数类型。回应上文，如果我们想防止sql注入，理所当然地要在输入参数上下功夫。上面代码中“#{id}”即输入参数在sql中拼接的部分，传入参数后，打印出执行的sql语句，会看到sql是这样的：

> select id,title,author,content from blog where id = ?

不管输入什么参数，打印出的sql都是这样的。这是因为`mybatis`启用了预编译功能，在sql执行前，会先将上面的sql发送给数据库进行编译，执行时，直接使用编译好的sql，替换占位符“？”就可以了。因为sql注入只能对编译过程起作用，所以这样的方式就很好地避免了sql注入的问题。

`mybatis`是如何做到sql预编译的呢？其实在框架底层，是`jdbc`中的`PreparedStatement`类在起作用，`PreparedStatement`是我们很熟悉的`Statement`的子类，它的对象包含了编译好的sql语句。这种“准备好”的方式不仅能提高安全性，而且在多次执行一个sql时，能够提高效率，原因是sql已编译好，再次执行时无需再编译。

补充


```sql
<select id=“orderBlog“ resultType=“Blog“ parameterType=”map”>
 
       select id,title,author,content from blog order by ${orderParam}
 
</select>
```
仔细观察，内联参数的格式由“#{xxx}”变为了${xxx}。如果我们给参数“orderParam”赋值为”id”,将sql打印出来，是这样的：

> select id,title,author,content from blog order by id

显然，这样是无法阻止sql注入的。在mybatis中，”${xxx}”这样格式的参数会直接参与sql编译，从而不能避免注入攻击。**但涉及到动态表名和列名时，只能使用“${xxx}”这样的参数格式**，所以，这样的参数需要我们在代码中手工进行处理来防止注入。

## 21. mysql悲观锁和乐观锁

## 21.1 悲观锁

悲观锁（`Pessimistic Lock`），顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。

悲观锁：假定会发生并发冲突，屏蔽一切可能违反数据完整性的操作。

`Java synchronized` 就属于悲观锁的一种实现，每次线程要修改数据时都先获得锁，保证同一时刻只有一个线程能操作数据，其他线程则会被block。

## 21.2 乐观锁

乐观锁（`Optimistic Lock`），顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在提交更新的时候会判断一下在此期间别人有没有去更新这个数据。乐观锁适用于读多写少的应用场景，这样可以提高吞吐量。

乐观锁：假设不会发生并发冲突，只在提交操作时检查是否违反数据完整性。

乐观锁一般来说有以下2种方式：

* 使用数据版本（`Version`）记录机制实现，这是乐观锁最常用的一种实现方式。何谓数据版本？即为数据增加一个版本标识，一般是通过为数据库表增加一个数字类型的 `version` 字段来实现。当读取数据时，将`version`字段的值一同读出，数据每更新一次，对此`version`值加一。当我们提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的`version`值进行比对，如果数据库表当前版本号与第一次取出来的`version`值相等，则予以更新，否则认为是过期数据。
* 使用时间戳（`timestamp`）。乐观锁定的第二种实现方式和第一种差不多，同样是在需要乐观锁控制的table中增加一个字段，名称无所谓，字段类型使用时间戳（`timestamp`）, 和上面的`version`类似，也是在更新提交的时候检查当前数据库中数据的时间戳和自己更新前取到的时间戳进行对比，如果一致则OK，否则就是版本冲突。
`Java JUC`中的`atomic`包就是乐观锁的一种实现，`AtomicInteger` 通过`CAS`（`Compare And Set`）操作实现线程安全的自增。

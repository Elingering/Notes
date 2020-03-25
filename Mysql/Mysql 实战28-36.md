# 28 | 读写分离有哪些坑
## 处理主备延迟导致的读写分离问题。
- 强制走主库方案；
- sleep方案；
- 判断主备无延迟方案；
- 配合semi-sync方案；
- 等主库位点方案；
- 等GTID方案。

# 29 | 如何判断一个数据库是不是出问题了
## 并发连接和并发查询
你在show processlist的结果里，看到的几千个连接，指的就是并发连接。而“当前正在执行”的语句，才是我们所说的并发查询。

通常情况下，我们建议把innodb_thread_concurrency设置为64~128之间的值。

并发连接数达到几千个影响并不大，就是多占一些内存而已。我们应该关注的是并发查询，因为并发查询太高才是CPU杀手。

==在线程进入锁等待以后，并发线程的计数会减一==

## 判断方法
- select 1判断
- 查表判断
- 更新判断
为了让主备之间的更新不产生冲突，我们可以在mysql.health_check表上存入多行数据，并用A、B的server_id做主键。
```sql
mysql> CREATE TABLE `health_check` (
  `id` int(11) NOT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

/* 检测命令 */
insert into mysql.health_check(id, t_modified) values (@@server_id, now()) on duplicate key update t_modified=now();
```
- 内部统计
MySQL 5.6版本以后提供的performance_schema库，就在file_summary_by_event_name表里统计了每次IO请求的时间。

	我的测试结果是，如果打开所有的performance_schema项，性能大概会下降10%左右。所以，我建议你只打开自己需要的项进行统计。你可以通过下面的方法打开或者关闭某个具体项的统计。

	如果要打开redo log的时间监控，你可以执行这个语句：
```sql
mysql> update setup_instruments set ENABLED='YES', Timed='YES' where name like '%wait/io/file/innodb/innodb_log_file%';
```
你可以通过MAX_TIMER的值来判断数据库是否出问题了。比如，你可以设定阈值，单次IO请求时间超过200毫秒属于异常，然后使用类似下面这条语句作为检测逻辑。
```sql
mysql> select event_name,MAX_TIMER_WAIT  FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;
```
发现异常后，取到你需要的信息，再通过下面这条语句：
```sql
mysql> truncate table performance_schema.file_summary_by_event_name;
```
把之前的统计信息清空。这样如果后面的监控中，再次出现这个异常，就可以加入监控累积值了。

## 小结
我个人比较倾向的方案，是优先考虑update系统表，然后再配合增加检测performance_schema的信息。

# 30 | 答疑文章（二）：用动态的观点看加锁

# 31 | 误删数据后除了跑路，还能怎么办
业务开发同学，你可以用show grants命令查看账户的权限，如果权限过大，可以建议DBA同学给你分配权限低一些的账号；你也可以评估业务的重要性，和DBA商量备份的周期、是否有必要创建延迟复制的备库等等。

# 32 | 为什么还有kill不掉的语句
“kill不掉”的情况，其实是因为发送kill命令的客户端，并没有强行停止目标线程的执行，而只是设置了个状态，并唤醒对应的线程。而被kill的线程，需要执行到判断状态的“埋点”，才会开始进入终止逻辑阶段。并且，终止逻辑本身也是需要耗费时间的。

所以，如果你发现一个线程处于Killed状态，你可以做的事情就是，通过影响系统环境，让这个Killed状态尽快结束。

比如，如果是第一个例子里InnoDB并发度的问题，你就可以临时调大innodb_thread_concurrency的值，或者停掉别的线程，让出位子给这个线程执行。

# 33 | 我查这么多数据，会不会把数据库内存打爆
## 全表扫描对server层的影响
MySQL是“边读边发的”
查询的结果是分段发给客户端的，因此扫描全表，查询返回大量的数据，并不会把内存打爆。

## 全表扫描对InnoDB的影响
内存的数据页是在Buffer Pool (BP)中管理的，在WAL里Buffer Pool 起到了加速更新的作用。而实际上，Buffer Pool 还有一个更重要的作用，就是加速查询。

而Buffer Pool对查询的加速效果，依赖于一个重要的指标，即：内存命中率。

你可以在show engine innodb status结果中，查看一个系统当前的BP命中率。一般情况下，一个稳定服务的线上系统，要保证响应时间符合要求的话，内存命中率要在99%以上。

而对于InnoDB引擎内部，由于有淘汰策略，大查询也不会导致内存暴涨。并且，由于InnoDB对LRU算法做了改进（5/8），冷数据的全表扫描，对Buffer Pool的影响也能做到可控。

# 34 | 到底可不可以使用join
## Index Nested-Loop Join
==在这个join语句执行过程中，**驱动表是走全表扫描**，而**被驱动表是走树搜索**。==

## Simple Nested-Loop Join
被驱动表无索引可用，扫描行数是一个笛卡尔积。
这个算法不被采用

## Block Nested-Loop Join

## 小结
==什么叫作“小表”==。

我们前面的例子是没有加条件的。如果我在语句的where条件加上 t2.id<=50这个限定条件，再来看下这两条语句：
```sql
select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=50;
select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=50;
```
注意，为了让两条语句的被驱动表都用不上索引，所以join字段都使用了没有索引的字段b。

但如果是用第二个语句的话，join_buffer只需要放入t2的前50行，显然是更好的。所以这里，“t2的前50行”是那个相对小的表，也就是“小表”。

我们再来看另外一组例子：
```sql
select t1.b,t2.* from  t1  straight_join t2 on (t1.b=t2.b) where t2.id<=100;
select t1.b,t2.* from  t2  straight_join t1 on (t1.b=t2.b) where t2.id<=100;
```
这个例子里，表t1 和 t2都是只有100行参加join。但是，这两条语句每次查询放入join_buffer中的数据是不一样的：

表t1只查字段b，因此如果把t1放到join_buffer中，则join_buffer中只需要放入b的值；
表t2需要查所有的字段，因此如果把表t2放到join_buffer中的话，就需要放入三个字段id、a和b。
这里，我们应该选择表t1作为驱动表。也就是说在这个例子里，“只需要一列参与join的表t1”是那个相对小的表。

==在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与join的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。==
1. 如果可以使用被驱动表的索引，join语句还是有其优势的；
2. 不能使用被驱动表的索引，只能使用Block Nested-Loop Join算法，这样的语句就尽量不要使用；
3. 在使用join的时候，应该让小表做驱动表。

# 35 | join语句怎么优化
## Multi-Range Read优化
这个优化的主要目的是尽量使用顺序读盘。

**因为大多数的数据都是按照主键递增顺序插入得到的，所以我们可以认为，如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能。**

## Batched Key Access
大表join操作虽然对IO有影响，但是在语句执行结束后，对IO的影响也就结束了。但是，对Buffer Pool的影响就是持续性的，需要依靠后续的查询请求慢慢恢复内存命中率。

## BNL转BKA
一些情况下，我们可以直接在被驱动表上建索引，这时就可以直接转成BKA算法了。
另一些不适合建立所引的，就要用到临时表，加索引。

不论是在原表上加索引，还是用有索引的临时表，我们的思路都是让join语句能够用上被驱动表上的索引，来触发BKA算法，提升查询性能。

## 扩展-hash join
看到这里你可能发现了，其实上面计算10亿次那个操作，看上去有点儿傻。如果join_buffer里面维护的不是一个无序数组，而是一个哈希表的话，那么就不是10亿次判断，而是100万次hash查找。

实际上，这个优化思路，我们可以自己实现在业务端。实现流程大致如下：
1. select * from t1;取得表t1的全部1000行数据，在业务端存入一个hash结构，比如C++里的set、PHP的dict这样的数据结构。
2. select * from t2 where b>=1 and b<=2000; 获取表t2中满足条件的2000行数据。
3. 把这2000行数据，一行一行地取到业务端，到hash结构的数据表中寻找匹配的数据。满足匹配的条件的这行数据，就作为结果集的一行。

理论上，这个过程会比临时表方案的执行速度还要快一些。

# 36 | 为什么临时表可以重名
- 内存表，指的是使用Memory引擎的表，建表语法是create table … engine=memory。这种表的数据都保存在内存里，系统重启的时候会被清空，但是表结构还在。除了这两个特性看上去比较“奇怪”外，从其他的特征上看，它就是一个正常的表。

- 而临时表，可以使用各种引擎类型 。如果是使用InnoDB引擎或者MyISAM引擎的临时表，写数据的时候是写到磁盘上的。当然，临时表也可以使用Memory引擎。




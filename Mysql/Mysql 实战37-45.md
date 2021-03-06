# 37 | 什么时候会使用内部临时表
## union 执行流程
1. 创建一个内存临时表，这个临时表只有一个整型字段f，并且f是主键字段。
2. 执行第一个子查询，得到1000这个值，并存入临时表中。
3. 执行第二个子查询：
	- 拿到第一行id=1000，试图插入临时表中。但由于1000这个值已经存在于临时表了，违反了唯一性约束，所以插入失败，然后继续执行；
	- 取到第二行id=999，插入临时表成功。
4. 从临时表中按行取出数据，返回结果，并删除临时表，结果中包含两行数据分别是1000和999。

## group by 执行流程
1. 创建内存临时表，表里有两个字段m和c，主键是m；
2. 扫描表t1的索引a，依次取出叶子节点上的id值，计算id%10的结果，记为x；
	- 如果临时表中没有主键为x的行，就插入一个记录(x,1);
	- 如果表中有主键为x的行，就将x这一行的c值加1；
3. 遍历完成后，再根据字段m做排序，得到结果集返回给客户端。

## group by 优化方法 --索引
group by的语义逻辑，是统计不同的值出现的个数。但是，由于每一行的id%100的结果是无序的，所以我们就需要有一个临时表，来记录并统计结果。

那么，如果扫描过程中可以保证出现的数据是有序的，是不是就简单了呢？

在MySQL 5.7版本支持了generated column机制，用来实现列数据的关联更新。你可以用下面的方法创建一个列z，然后在z列上创建一个索引（如果是MySQL 5.6及之前的版本，你也可以创建普通列和索引，来解决这个问题）。
```sql
alter table t1 add column z int generated always as(id % 100), add index(z);
```

## group by优化方法 --直接排序
在group by语句中加入SQL_BIG_RESULT这个提示（hint），就可以告诉优化器：这个语句涉及的数据量很大，请直接用磁盘临时表。

MySQL的优化器一看，磁盘临时表是B+树存储，存储效率不如数组来得高。所以，既然你告诉我数据量很大，那从磁盘空间考虑，还是直接用数组来存吧。

**MySQL什么时候会使用内部临时表？**
1. 如果语句执行过程可以一边读数据，一边直接得到结果，是不需要额外内存的，否则就需要额外的内存，来保存中间结果；
2. join_buffer是无序数组，sort_buffer是有序数组，临时表是二维表结构；
3. 如果执行逻辑需要用到二维表特性，就会优先考虑使用临时表。比如我们的例子中，union需要用到唯一索引约束， group by还需要用到另外一个字段来存累积计数。

## 小结
1. 如果对group by语句的结果没有排序要求，要在语句后面加 order by null；
2. 尽量让group by过程用上表的索引，确认方法是explain结果里没有Using temporary 和 Using filesort；
3. 如果group by需要统计的数据量不大，尽量只使用内存临时表；也可以通过适当调大tmp_table_size参数，来避免用到磁盘临时表；
4. 如果数据量实在太大，使用SQL_BIG_RESULT这个提示，来告诉优化器直接使用排序算法得到group by的结果。

# 38 | 都说InnoDB好，那还要不要使用Memory引擎
## 内存表的数据组织结构
- InnoDB引擎把数据放在主键索引上，其他索引上保存的是主键id。这种方式，我们称之为==索引组织表==（Index Organizied Table）。
- 而Memory引擎采用的是把数据单独存放，索引上保存数据位置的数据组织形式，我们称之为==堆组织表==（Heap Organizied Table）。

## hash索引和B-Tree索引
实际上，内存表也是支B-Tree索引的。
不建议你在生产环境上使用内存表。这里的原因主要包括两个方面：
1. 锁粒度问题；
2. 数据持久化问题。

## 内存表的锁
内存表不支持行锁，只支持表锁。

## 数据持久性问题
数据放在内存中，是内存表的优势，但也是一个劣势。因为，数据库重启的时候，所有的内存表都会被清空。
在数据量可控，不会耗费过多内存的情况下，你可以考虑使用内存表。

内存临时表刚好可以无视内存表的两个不足，主要是下面的三个原因：
1. 临时表不会被其他线程访问，没有并发性的问题；
2. 临时表重启后也是需要删除的，清空数据这个问题不存在；
3. 备库的临时表也不会影响主库的用户线程。

# 39 | 自增主键为什么不是连续的
自增主键不能保证连续递增。
## 自增值保存在哪儿？
1. MyISAM引擎的自增值保存在数据文件中。
2. InnoDB引擎的自增值，其实是保存在了内存里。
- MySQL重启可能会修改一个表的AUTO_INCREMENT的值。
- 在MySQL 8.0版本，将自增值的变更记录在了redo log中，重启的时候依靠redo log恢复重启之前的值。

## 自增值修改机制
自增值不能回退。
- 唯一键冲突是导致自增主键id不连续的第一种原因。
- 事务回滚也会产生类似的现象，这就是第二种原因。
- 批量插入申请的id没用完。

## 自增锁的优化
在生产上，尤其是有insert … select这种批量插入数据的场景时，从并发插入数据性能的角度考虑，我建议你这样设置：innodb_autoinc_lock_mode=2（申请到自增id以后就立马释放自增锁），并且 binlog_format=row.这样做，既能提升并发性，又不会出现数据一致性问题。

需要注意的是，我这里说的==批量插入==数据，包含的语句类型是==insert … select、replace … select和load data语句==。

对于批量插入数据的语句，MySQL有一个批量申请自增id的策略：
1. 语句执行过程中，第一次申请自增id，会分配1个；
2. 1个用完以后，这个语句第二次申请自增id，会分配2个；
3. 2个用完以后，还是这个语句，第三次申请自增id，会分配4个；
4. 依此类推，同一个语句去申请自增id，每次申请到的自增id个数都是上一次的两倍。

# 40 | insert语句的锁为什么这么多
有些insert语句是属于“特殊情况”的，在执行过程中需要给其他资源加锁，或者无法在申请到自增id以后就立马释放自增锁。

## insert … select 语句
在可重复读隔离级别下，binlog_format=statement时：
```sql
insert into t2(c,d) select c,d from t;
```
需要对表t的所有行和间隙加锁，防止主备不一致。

## insert 循环写入
当然了，执行insert … select 的时候，对目标表也不是锁全表，而是只锁住需要访问的资源（limit）。

## 小结
insert … select 是很常见的在两个表之间拷贝数据的方法。你需要注意，在可重复读隔离级别下，这个语句会给select的表里扫描到的记录和间隙加读锁。

而如果insert和select的对象是同一个表，则有可能会造成循环写入。这种情况下，我们需要引入用户临时表来做优化。

insert 语句如果出现唯一键冲突，会在冲突的唯一值上加共享的next-key lock(S锁)。因此，碰到由于唯一键约束导致报错后，要尽快提交或回滚事务，避免加锁时间过长。

# 41 | 怎么最快地复制一张表
## mysqldump方法
一种方法是，使用mysqldump命令将数据导出成一组INSERT语句。你可以使用下面的命令：
```sql
mysqldump -h$host -P$port -u$user --add-locks --no-create-info --single-transaction  --set-gtid-purged=OFF db1 t --where="a>900" --result-file=/client_tmp/t.sql
```

## 导出CSV文件
另一种方法是直接将结果导出成.csv文件。MySQL提供了下面的语法，用来将查询结果导出到服务端本地目录：
```sql
select * from db1.t where a>900 into outfile '/server_tmp/t.csv';
```

## 物理拷贝方法
在MySQL 5.6版本引入了可传输表空间(transportable tablespace)的方法，可以通过导出+导入表空间的方式，实现物理拷贝表的功能。

## 小结
今天这篇文章，我和你介绍了三种将一个表的数据导入到另外一个表中的方法。

我们来对比一下这三种方法的优缺点。

1. 物理拷贝的方式速度最快，尤其对于大表拷贝来说是最快的方法。如果出现误删表的情况，用备份恢复出误删之前的临时库，然后再把临时库中的表拷贝到生产库上，是恢复数据最快的方法。但是，这种方法的使用也有一定的局限性：
	- 必须是全表拷贝，不能只拷贝部分数据；
	- 需要到服务器上拷贝数据，在用户无法登录数据库主机的场景下无法使用；
	- 由于是通过拷贝物理文件实现的，源表和目标表都是使用InnoDB引擎时才能使用。
2. 用mysqldump生成包含INSERT语句文件的方法，可以在where参数增加过滤条件，来实现只导出部分数据。这个方式的不足之一是，不能使用join这种比较复杂的where条件写法。
3. 用select … into outfile的方法是最灵活的，支持所有的SQL写法。但，这个方法的缺点之一就是，每次只能导出一张表的数据，而且表结构也需要另外的语句单独备份。

后两种方式都是逻辑备份方式，是可以跨引擎使用的。

# 42 | grant之后要跟着flushprivileges吗
## 全局权限
==一般在生产环境上要合理控制用户权限的范围。==我们上面用到的这个grant语句就是一个典型的错误示范。如果一个用户有所有权限，一般就不应该设置为所有IP地址都可以访问。

## db权限
grant修改db权限的时候，是同时对磁盘和内存生效的。

## 表权限和列权限
跟db权限类似，这两个权限每次grant的时候都会修改数据表，也会同步修改内存中的hash结构。因此，对这两类权限的操作，也会马上影响到已经存在的连接。

==正常情况下，grant命令之后，没有必要跟着执行flush privileges命令。==

## flush privileges使用场景
当数据表中的权限数据跟内存中的权限数据不一致的时候，flush privileges语句可以用来重建内存数据，达到一致状态。

这种不一致往往是由不规范的操作导致的，比如直接用DML语句操作系统权限表。

# 43 | 要不要使用分区表
## 分区表是什么
- 对于引擎层来说，这是4个表；
- 对于Server层来说，这是1个表。

## 分区表的引擎层行为
分区表和手工分表，一个是由server层来决定使用哪个分区，一个是由应用层代码来决定使用哪个分表。因此，从引擎层看，这两种方式也是没有差别的。

其实这两个方案的区别，主要是在server层上。从server层看，我们就不得不提到分区表一个被广为诟病的问题：打开表的行为。

## 分区策略
MyISAM分区表使用的分区策略，我们称为==通用分区策略==（generic partitioning），每次访问分区都由server层控制。(弃用)

从MySQL 5.7.9开始，InnoDB引擎引入了==本地分区策略==（native partitioning）。

## 分区表的server层行为
如果从server层看的话，一个分区表就只是一个表。

1. MySQL在第一次打开分区表的时候，需要访问所有的分区；
2. 在server层，认为这是同一张表，因此所有分区共用同一个MDL锁；
3. 在引擎层，认为这是不同的表，因此MDL锁之后的执行过程，会根据分区表规则，只访问必要的分区。

## 分区表的应用场景
分区表的一个显而易见的优势是对业务透明，相对于用户分表来说，使用分区表的业务代码更简洁。还有，分区表可以很方便的清理历史数据。

如果一项业务跑的时间足够长，往往就会有根据时间删除历史数据的需求。这时候，按照时间分区的分区表，就可以直接通过alter table t drop partition …这个语法删掉分区，从而删掉过期的历史数据。

这个alter table t drop partition …操作是直接删除分区文件，效果跟drop普通表类似。与使用delete语句删除数据相比，优势是速度快、对系统影响小。

## 小结
分区表跟用户分表比起来，有两个绕不开的问题：一个是第一次访问的时候需要访问所有分区，另一个是共用MDL锁。

这里有两个问题需要注意：
- 分区并不是越细越好。实际上，单表或者单分区的数据一千万行，只要没有特别大的索引，对于现在的硬件能力来说都已经是小表了。
- 分区也不要提前预留太多，在使用之前预先创建即可。比如，如果是按月分区，每年年底时再把下一年度的12个新分区创建上即可。对于没有数据的历史分区，要及时的drop掉。

# 44 | 答疑文章（三）：说一说这些好问题
## join的写法
在MySQL里，NULL跟任何值执行等值判断和不等值判断的结果，都是NULL。这里包括， select NULL = NULL 的结果，也是返回NULL。

==使用left join时，左边的表不一定是驱动表。==

==如果需要left join的语义，就不能把被驱动表的字段放在where条件里面做等值判断或不等值判断，必须都写在on里面。==

# 45 | 自增 id 用完怎么办？
1.  表的自增 id 达到上限后，再申请时它的值就不会改变，进而导致继续插入数据时报主键冲突
的错误。
2. row_id 达到上限后，则会归 0 再重新递增，如果出现相同的 row_id ，后写的数据会覆盖之前
的数据。
3. Xid 只需要不在同一个 binlog 文件中出现重复值即可。虽然理论上会出现重复值，但是概率极
小，可以忽略不计。
4. InnoDB 的 max_trx_id  递增值每次 MySQL 重启都会被保存起来，所以我们文章中提到的脏读
的例子就是一个必现的 bug ，好在留给我们的时间还很充裕。
5. thread_id 是我们使用中最常见的，而且也是处理得最好的一个自增 id 逻辑了。

当然，在 MySQL 里还有别的自增 id ，比如 table_id 、 binlog 文件序号等，就留给你去验证和探索
了。
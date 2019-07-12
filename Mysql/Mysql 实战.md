# 基础架构：一条SQL查询语句是如何执行的?

![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/12/1562894956552-1562894956587.png?token=AFRM336B6ASVJK6EZQGJTJK5E7RKY)
MySQL 的逻辑架构图

## 连接器
```sql
mysql -h$ip -P$port -u$user -p
```
- 使用 show processlist 命令查看连接状态
- 自动关闭 sleep 连接，wait_timeout 默认8小时
- 也有长连接和短连接。长连接耗内存，解决办法：
1. 定期断开长连接。使用一段时间，或者程序里面判断执行过一个占用内存的大查询后，断开连接，之后要查询再重连。如果你用的是。
2. MySQL 5.7 或更新版本，可以在每次执行一个比较大的操作后，通过执行 mysql_reset_connection 来重新初始化连接资源。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态。

## 查询缓存
MySQL 拿到一个查询请求后，会先到查询缓存看看，之前是不是执行过这条语句。之前执行过的语句及其结果可能会以 key-value 对的形式，被直接缓存在内存中。key 是查询的语句，value 是查询的结果。如果你的查询能够直接在这个缓存中找到 key，那么这个 value 就会被直接返回给客户端。如果语句不在查询缓存中，就会继续后面的执行阶段。

查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。因此很可能你费劲地把结果存起来，还没使用呢，就被一个更新全清空了。

将参数 query_cache_type 设置成 DEMAND，这样对于默认的 SQL 语句都不使用查询缓存。
要使用查询缓存的语句，可以用 SQL_CACHE 显式指定：
```sql
mysql> select SQL_CACHE * from T where ID=10；
```
==MySQL 8.0 删除了查询缓存==

## 分析器
”词法分析“->”语法分析“

## 优化器
优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序。

## 执行器
开始执行的时候，要先判断一下你对这个表 T 有没有执行查询的权限，如果没有，就会返回没有权限的错误，如下所示 (在工程实现上，如果命中查询缓存，会在查询缓存返回结果的时候，做权限验证。查询也会在优化器之前调用 precheck 验证权限)。

在有些场景下，执行器调用一次，在引擎内部则扫描了多行，因此**引擎扫描行数跟 rows_examined 并不是完全相同的**。

# 日志系统：一条SQL更新语句是如何执行的？
查询语句的那一套流程，更新语句也是同样会走一遍。

## 重要的日志模块：redo log （InnoDB 引擎特有的日志）
WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘。
当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log （粉板）里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面。

InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么这块“粉板”总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写：
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/12/1562897221514-1562897221521.png?token=AFRM3364FNVDRLEDCXG6LUC5E7VYI)
write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。（这个用LSN来记录，后面文章会提到）

write pos 和 checkpoint 之间的是“粉板”上还空着的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，表示“粉板”满了，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe

## 重要的日志模块：binlog （server 层的日志）
两种日志的区别：
- redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
- redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
- redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

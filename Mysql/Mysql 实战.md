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
”词法分析“-》
## 优化器

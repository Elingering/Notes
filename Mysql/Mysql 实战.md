# 基础架构：一条SQL查询语句是如何执行的?

![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/12/1562894956552-1562894956587.png?token=AFRM336B6ASVJK6EZQGJTJK5E7RKY)
Mysql 的逻辑架构图

## 连接器
```mysql
mysql -h$ip -P$port -u$user -p
```
- 使用 show processlist 命令查看连接状态
- 自动关闭 sleep 连接，wait_timeout 默认8小时
- 也有长连接和短连接
1. 定期断开长连接。使用一段时间，或者程序里面判断执行过一个占用内存的大查询后，断开连接，之后要查询再重连。如果你用的是 MySQL 5.7 或更新版本，可以在每次执行一个比较大的操作后，通过执行 mysql_reset_connection 来重新初始化连接资源。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态。
# 持久化方式
快照：MySQL Dump；RDB
写日志：MySQL Binlog；AOF

# RDB
以二进制形式，完整保存redis内容

触发机制：
1. save
	- 保存内容多会阻塞
	- 新文件替换老文件
	- O(n)
2. bgsave
	- 异步几乎不阻塞
	- 异步需要消耗内存
3. 自动


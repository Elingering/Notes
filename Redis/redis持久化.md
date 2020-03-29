# 持久化方式
快照：MySQL Dump；RDB
写日志：MySQL Binlog；AOF

# RDB
以二进制形式，完整保存redis内容

## 触发机制：
1. save
	- 保存内容多会阻塞
	- 新文件替换老文件
	- O(n)
2. bgsave
	- 异步几乎不阻塞
	- 异步需要消耗内存
3. 自动
	- 满足条件触发bgsave
	- 如：60秒更新1000次

## 最佳配置
dbfilename dump-${port}.rdb
dir /bigdiskpath
stop-writes-on-bgsave-error yes
rdbcompression yes #压缩
rdbchecksum yes #

## 触发方式
1. 全量复制（主从复制）
2. debug reload
3. shutdown

# AOF
RDB缺点：
1. 耗时、耗性能
	- O(n)
	- fork()耗内存
	- 写盘I/O
2. 不可控、丢失数据


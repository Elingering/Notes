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

## 策略
always:每条命令都fsync到硬盘
everysec:每秒fsync到硬盘
no：系统决定

##　AOF重写
减少硬盘占用量
加速恢复速度

##　AOF重写的两种方式
bgrewriteaof
AOF重写配置

## fork操作
1. 同步操作
2. 于内存量息息相关
3. info：latest_fork_usec

## 改善fork
1. 优先使用物理机或者高效支持fork操作的虚拟化技术
2. 控制redis实例最大可用内存：maxmemory
3. 合理配置Linux内存分配策略：vm.overcommit_memory=1
4. 降低fork频率：例如放宽AOF重写自动触发时机，不必要的全量复制

## 子进程开销和优化
1. cpu
	- 开销：RDB和AOF文件生成，属于cpu密集型
	- 优化：不做cpu绑定，不和cpu密集型部署
2. 内存
	- 开销：fork内存开销，copy-on-write
	- 优化：echo never > /sys/kernel/mm/transparent_hugepage/enabled
3. 硬盘
	- 开销：AOF和RDB文件写入，可以结合iostat，iotop分析
	- 优化：不要和高硬盘负载服务部署：存储服务，消息队列。。。；no-appendfsync-on-rewrite=yes；使用ssd；单机多实例持久化文件目录可以考虑分盘

## AOF追加阻塞
#image#
每秒刷盘策略可能不止丢失一秒的数据，也可能是两秒。优化硬盘I/O


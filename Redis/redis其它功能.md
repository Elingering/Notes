# 慢查询
先进先出队列
固定长度，slowlog-max-len（128，建议1000）
保存在内存里

slowlog-log-slower-than（微妙：10000，建议1000）

slowlog list
slowlog get [n]
slowlog len
slowlog reset

# pipeline
合并操作命令，减少网络请求，注意每次携带数据量
是非原子操作
只能运行在一个节点

# 发布订阅
publish channel message #返回订阅者个数
subscribe [channel ...]
unsubscribe [channel ...]

# Bitmap
位图：字符串对应的二进制

setbit key offset value
getbit key offset
bitcount key [start end] #获取位图指定范围（字节）位置为1的个数
bitop op destkey key [key]

# HyperLogLog

# GEO
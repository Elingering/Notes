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
bitop op destkey key [key] #做多个Bitmap的and（交）、or（并）、not（非）、xor（异或）操作，并将结果保存在destkey中
bitops key targetBit [start] [end] #计算位图指定范围（字节）第一个偏移量对应的值等于他让个人Bit的位置

场景：独立数量统计

# HyperLogLog
1. 基于HyperLogLog算法：极小空间完成独立数量统计
2. 本质还是字符串
3. 不重复

pfadd key element [element...]
pfcount key [key..] #计算独立总数
pfmerge destkey sourcekey [sourcekey...] #合并

缺点：
1. count 错误率0.81%
2. 无法取出单条数据

# GEO
地理位置信息：存储经纬度，计算两地距离，范围计算等

## API
geoadd key longitude latitude member
geopos key member
geodist key member1 member2 [unit] #m（米）、km（千米）、mi（英里）、ft（尺）
georadius #范围查询
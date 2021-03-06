## 通用命令
- keys [pattern] #遍历key
- dbsize #计算key的总数
- exists key #检查key是否存在 0 1
- del key [key...] #删除指定key-value 0 1
- expire key seconds #key在seconds秒后过期（也有其它单位的命令）
- ttl key #查看key的剩余过期时间 -1永不过期 0刚好过期 -2过期已删除
- persist key #去掉key的过期时间
- type key #返回key的类型

## 数据结构和内部编码
![title](https://raw.githubusercontent.com/Elingering/note-images/master/gitnote/2020/03/26/Snipaste_2020-03-26_17-22-15-1585214568363.png)


## 单线程
为什么快：
1. 纯内存
2. 非阻塞IO
3. 避免了线程切换和竞态消耗

## 字符串
### 场景
- 缓存
- 计数器
- 分布式锁

### API
get, set, del
incr, decr, incrby, decrby
set, setnx(新增), set key value xx(更新)
mget, mset
getset, append, strlen(中文utf8，两字节)
incrbyfloat, getrange key start end, setrange key index value

## 哈希
### API
hget, hset, hdel
hexists key field, hlen key(返回field个数)
hincrby key field value
hget all, hvals, hkeys

## 列表
有序，可以重复，左右两边插入弹出
### API
lpush, rpush
linsert key before|after value newValue
lpop key #从左边坛醋一个元素
rpop key #返回弹出的元素
lrem key count value #count>0,从左到右;count=0删除所有
ltrim key start end #保留start至end间的元素 []
lrange key start end #获取start至end间的元素 []
lindex key index
llen key
lset key index newValue
brpop key timeout #rpop阻塞版本，timeout是阻塞超时时间，为0永远不阻塞

### TIPS
lpush + lpop = stack
lpush + rpop = queue
lpush + ltrim = capped collection（固定数量的集合）
lpush + brpop = message queue

## 集合
无序，不能重复，集合间操作

###场景
点赞，抽奖，标签，共同关注

### API
sadd, srem
scard, sismember, srandmember, spop, smembers
sinter（交）， sdiff（差）， sunion（并） + store destkey

### TIPS
sadd = tagging
spop/srandmember = rnadom item
sadd + sinter = social graph

## 有序集合
### API
zadd key score element, zrem
zscore key element
zincrby key increScore element
zrank key element #0->1
zrange key start end [withscores] #获取某范围排名的值，升序
zrangebyscore key minScore maxScore [withscores] #获取分值围排名的值，升序
zcount key minScore maxScore
zremrangebyrank key start end
zremrangebyscore key minScore maxScore



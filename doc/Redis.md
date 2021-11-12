## lazy free
* UNLINK
* FLUSHALL/FLUSHDB ASYNC
* 配置策略
> lazyfree-lazy-eviction；lazyfree-lazy-expire；
lazyfree-lazy-server-del（rename命令，当目标键已存在,redis会先删除目标键，如果这些目标键是一个big key,那就会引入阻塞删除的性能问题）；
slave-lazy-flush（针对slave进行全量数据同步，slave在加载master的RDB文件前，会运行flushall来清理自己的数据）

## 内存淘汰策略
* no-eviction
* volatile-random
* volatile-ttl
* volatile-lru
* volatile-lru
* allkeys-random
* allkeys-lru

## 过期键的删除策略
* 定时过期：每个设置过期时间的key都需要创建一个定时器，到过期时间就会立即清除。
* 惰性过期
* 定期过期：每隔一定的时间，会扫描一定数量的数据库的expires字典中一定数量的key，并清除其中已过期的key。

## 主从复制
* min-slaves-max-lag
* min-slaves-to-write

## 集群（shard）数据再均衡
* redis-cli --cluster rebalance

## 数据结构
> Redis还会对ziplist进行压缩存储，使用LZF算法压缩。
* string 用 sds和raw
* zset 和 hash 用ziplist
* list 用 快速列表（早期用ziplist和双向链表）
* zset 用 跳表（score）
* Redis5.0 引入listpack

## dict（字典）rehash
---
layout: post
title:  "Redis"
date:   2020-07-11 13:57:31
categories: Redis
tags: Java 数据库 缓存 
---

* content
{:toc}

Redis是应用的最多的缓存中间件，也是使用最多的非关系型数据库，其重要性不可言喻。





## Redis
### 过期策略
对于已经过期的数据，Redis 将使用两种策略来删除这些过期键，它们分别是惰性删除和定期删除。

#### 惰性删除
指 Redis 服务器不主动删除过期的键值，而是当访问键值时，再检查当前的键值是否过期，如果过期则执行删除并返回 null 给客户端；如果没过期则正常返回值信息给客户端。
```java
int expireIfNeeded(redisDb *db, robj *key) {
    // 判断键是否过期
    if (!keyIsExpired(db,key)) return 0;
    if (server.masterhost != NULL) return 1;
    /* 删除过期键 */
    // 增加过期键个数
    server.stat_expiredkeys++;
    // 传播键过期的消息
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    // server.lazyfree_lazy_expire 为 1 表示异步删除，否则则为同步删除
    return server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) :
                                         dbSyncDelete(db,key);
}
// 判断键是否过期
int keyIsExpired(redisDb *db, robj *key) {
    mstime_t when = getExpire(db,key);
    if (when < 0) return 0; 
    if (server.loading) return 0;
    mstime_t now = server.lua_caller ? server.lua_time_start : mstime();
    return now > when;
}
// 获取键的过期时间
long long getExpire(redisDb *db, robj *key) {
    dictEntry *de;
    if (dictSize(db->expires) == 0 ||
       (de = dictFind(db->expires,key->ptr)) == NULL) return -1;
    serverAssertWithInfo(NULL,key,dictFind(db->dict,key->ptr) != NULL);
    return dictGetSignedIntegerVal(de);
}
```

#### 定期删除
![](/images/redis.png)

### 内存淘汰策略
- noeviction：不淘汰任何数据，当内存不足时，执行缓存新增操作会报错，它是 Redis 默认内存淘汰策略。
- volatile-random：随机淘汰设置了过期时间的任意键值。
- allkeys-lru：淘汰整个键值中最久未使用的键值。
- allkeys-random：随机淘汰任意键值。
- volatile-lru：淘汰所有设置了过期时间的键值中最久未使用的键值。
- volatile-ttl：优先淘汰更早过期的键值。
- volatile-lfu，淘汰所有设置了过期时间的键值中最少使用的键值；
- allkeys-lfu，淘汰整个键值中最少使用的键值。

### 内存淘汰算法
LRU和LFU

### 分布式锁
我们前面所讲的[在程序中使用的锁](https://xiaoy-jojo.github.io/2020/07/03/Lock/)都是单机锁，单机锁只能在单台服务器上生效，无法应用在分布式系统中。分布式锁的实现方式主要为四种：
- 基于 MySQL 的悲观锁来实现分布式锁
- 基于 Memcached 实现分布式锁
- 基于 Redis 实现分布式锁
- 基于 ZooKeeper 实现分布式锁

#### 使用 Redis 实现分布式锁
使用 Redis 实现分布式锁主要需要使用 setnx 方法
```java
127.0.0.1:6379> setnx lock true
(integer) 1 //返回为1时表示创建成功
#逻辑业务处理...
127.0.0.1:6379> del lock
(integer) 1 #释放锁

//该命令判断锁是否存在并将超时时间存入键值对
127.0.0.1:6379> set lock true ex 30 nx

```

### 实现消息队列
Redis一共有四种方式实现：
1. 使用 List 实现消息队列；
2. 使用 ZSet 实现消息队列；
3. 使用发布订阅者模式实现消息队列（不支持持久化）；
4. 使用 Stream 实现消息队列（支持消费者确认）。

### Redis的高可用
实现高可用的手段主要有四种：
1. 数据持久化，保证了系统在发生宕机或者重启之后数据不会丢失，增加了系统的可靠性；
2. 主从数据同步，可以将数据存储至多台服务器；
3. 哨兵模式，用于发生故障之后自动切换服务器；
4. 集群，用于提供性能更好的 Redis 服务，并且它自身拥有故障自动切换的能力。

#### 数据持久化
4.0之前数据持久化方式有AOF和RDB两种，之后推出了混合持久化。
- RDB（Redis DataBase，快照方式）是将某一个时刻的内存数据，以二进制的方式写入磁盘，占用的空间更小、并且与 AOF 相比，RDB 具备更快的重启恢复能力；
- AOF（Append Only File，文件追加方式）是指将所有的操作命令，以文本的形式追加到文件中，存储频率更高，因此丢失数据的风险就越低，缺点是占用空间大，重启之后的数据恢复速度比较慢；
- 混合持久化：开始的数据以 RDB 的格式进行存储，因此只会占用少量的空间，并且之后的命令会以 AOF 的方式进行数据追加，这样就可以减低数据丢失的风险，同时可以提高数据恢复的速度。

#### 主从模式
- 把多个 Redis 节点组成一个 Redis 集群，在这个集群当中有一个主节点用来进行数据的操作，其他从节点用于同步主节点的内容。
- 分为主从模式和从从模式（一级从节点下还可以拥有其他从节点。
- 可以实现数据读写分离，主节点用来写，从节点用来读。
- 主节点宕机需要人工介入恢复。


#### 哨兵模式
- 用来监视 Redis 主从服务器，当 Redis 的主从服务器发生故障之后，Redis 哨兵提供了自动容灾修复的功能。
- 如果一个主服务器被标记为主观下线，那么正在监视这个主服务器的所有哨兵节点，要以每秒 1 次的频率确认主服务器是否进入了主观下线的状态。如果有足够数量（quorum 配置值）的哨兵证实该主服务器为主观下线，那么这个主服务器被标记为客观下线。此时所有的哨兵会按照规则（协商）自动选出新的主节点服务器，并自动完成主服务器的自动切换功能。

#### Redis集群
将数据分布在不同的主服务器上，以此来降低系统对单主节点的依赖，并且可以大大提高 Redis 服务的读写性能。Redis 集群除了拥有主从模式 + 哨兵模式的所有功能之外，还提供了多个主从节点的集群功能。


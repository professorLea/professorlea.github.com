---
title: Redis practice（一）基础篇
date: 2016-08-22 13:42:33
tags: Redis
categories: Redis
---

#### 引言

> 先罗列一下缓存的基本概念。

**所有类型的缓存：**
- L1 L2: CPU缓存（Cache Memory）是位于CPU与内存之间的临时存储器，它的容量比内存小的多但是交换速度却比内存要快得多
- DB cache(mysql query cache)
- Brower
- CDN
- Mobile image cache

**那些数据适合缓存：**
- 热点数据：top10，秒杀
- 中间结果：比如哨兵的报警表达式触发次数，这次次数需要累计起来，存入数据库似乎没有意义
- 预计算结果：计数、总量、max avg等统计结果。

**常用的缓存框架**
- Ehcache: “JAVA’S MOST WIDELY-USED CACHE”，rmi协议支持jvm间数据同步问题
- Spring cache ：spring 给出的 缓存抽象，不是具体实现
这些都属于本地缓存，Ehcache可以满足JVM间数据同步的问题。具体实现可以用spring cache + echcache

**常用的缓存数据库**
- Map，虽然听起来有点土，但是属于程序猿做本地缓存的一种办法
- Redis
- MemCached

那么离不开这两者的比较：
**Redis vs MemCached**

|item |Redis|MemCached|
|:---:|:---:|:---:|
|高可用|支持|自己实现|
|持久化|支持|不支持|
|数据类型|5种类型|String类型|
|线程|单线程|多线程|
|事务|支持|使用CAS命令支持|

还有以下特点：
- Both fast enough
- Memcached is suitable for small and static data
- Redis is preferable for structured data
- Data can be persisted in Redis
- If only one can be choosed, Redis is a better choice ^_^

**Redis属于缓存数据库**

**应用**
- 业务层缓存 (spring cache)
- 持久层缓存 (Mybatis的二级缓存)
- 数据层缓存  (eg:mysql query cache)

**架构**
- 本地缓存 Map Ehcache
- 分布式缓存 JBossCache
- 集中式缓存 memcached redis


#### Redis安装及配置文件，及启动
安装的话就不用赘述了，从官方下一个tar包，比如http://download.redis.io/releases/redis-2.8.24.tar.gz
解压make，make install即可

**Redis.conf**
基本配置
- pidfile 记录redis-server的pid文件路径
- port Redis-server的监听端口
- daemonize Redis-server配置成守护进程
- loglevel  Redis-server的日志级别（warning， notice，verbose，debug）
- logfile 日志文件路径
- requirepass 密码
- maxmemory  Redis能使用的最大内存量
- maxmemory-policy 内存用完后的置换策略
    -   volatile-lru：设置了过期时间的key中，删除最近最少用的key
    - allkeys-lru：所有的key中，删除最近最少用的key
    - noeviction：不置换，直接返回OOM错误
    - 其他。
- databases 与mysql类似，一个redis实例中可以分多个库
    - 相同的库key是唯一的，不同的库key可以复用
    - 此配置项配置redis中库的个数

还有一些其他比如持久化和高可用的配置，另外一篇进阶篇会介绍。
更详细的文档，请参考官方：[详细配置](http://download.redis.io/redis-stable/redis.conf)

服务器端启动：
```powershell
redis-server redis.conf
```
客户端访问：
```powershell
redis-cli -h host -p port -a auth
```

#### Redis常用数据类型
> Tips: [命令大全](http://doc.redisfans.com)，[试用网站](http://try.redis.io/)

##### String类型
**格式：**"key" : "value"
**常用命令：**
- set key value
- get key
- del key
- incr/decr key
- setex key seconds value

##### List
**格式：**类似于JAVA的List类型，"key":["value1","value2"]
**常用命令：**
- lpush key value1 value2 …
- lrange key start end
- lrem count value
- lindex key index
- llen key

#####  Hash
**格式：**类似于Java中的Map类型，"key":{"field1":"value1", "field2":value2,...}
**常用命令：**
- hset key field value
- hget key field
- hdel key field1 field2…
- hkeys key
- hvals key

##### Set
**格式：**类似于Java中的set类型。"key":{"member1","member2",...}
**常用命令：**
- sadd key member1 member2 …
- srem key member1 member2 …
- smembers
- scard key
- sinter key1 key2
- sunion key1 key2

##### Sorted Set
**格式：** “key”: {("score1","memeber1", ("score2","memeber2")...)
**常用命令：**
- zadd key score member
- zrem key member
- zrangebyscore key min max (withscores)
- zcard key

##### 其他常用命令
- **Keys + 匹配条件:**  keys * ： 返回所有key
- **expire key seconds:**  设置key的过期时间
- **ttl key:**  查询key的过期时间
- **type key:** 查询key的类型
- **exists key:** 检查key是否存在
- **Publish/subcribe:**
    - 发布一个推送频道以及订阅一个推送频道
    - 推送端和接收端均连上同一个redis
    - Subscribe + channel1 + channel2
    - Publish channel1 “hello world”
    - 类似于MQ
- Multi/exec/discard
    - 事务
    - 要么全成功，要么全回滚


#### 常用的客户端
##### Linux
Redis-cli：安装后自带

##### Windows
[Redis destop manger](https://redisdesktop.com/)

#### 支持的编程语言
##### JAVA
**Jedis**
```java
public class SharedJedisTest {
    public  static JedisPoolConfig poolConfig = new JedisPoolConfig();
    public  static List<JedisShardInfo> shards = new ArrayList<JedisShardInfo>();
    static {
        poolConfig.setMaxTotal(100);
        poolConfig.setMaxIdle(10);
        poolConfig.setMaxWaitMillis(2000);
        JedisShardInfo info1 = new JedisShardInfo("10.165.124.10",16379);
        info1.setPassword("123456");
        JedisShardInfo info2 = new JedisShardInfo("10.165.124.10",16381);
        info2.setPassword("123456");
        shards.add(info1);
        shards.add(info2);
    }
    public static ShardedJedisPool shardedJedisPool = new ShardedJedisPool(poolConfig,shards);

    public static void main(String[] args){
        ShardedJedis jedis = null;
        try {
            jedis = shardedJedisPool.getResource();
            for (int i =0; i<50; i++){
                String key = String.format("key%d",i);
                String value = String.format("value%d",i);
                jedis.set(key, value);
                Client client1= jedis.getShard(key).getClient();
                System.out.println(String.format(("%s in server: %s, and port is: %d"),key,client1.getHost(),client1.getPort()));
            }
        }finally {
            if (jedis != null){
                jedis.close();
            }
        }

    }
}
```

##### Python
**pip install redis**

```python
import redis


def addString():
    redis_cli.set('string', 'value1')
    print redis_cli.get('string')


def addList():
    redis_cli.lpush('list', 'value1')
    print redis_cli.lrange('list', 0, -1)


def addHash():
    redis_cli.hmset('hash', {'field1': 'value1', 'field2': 'value2', 'field3': 'value3'})
    print redis_cli.hmget('hash', 'field1', 'field3')
    print redis_cli.hkeys('hash')
    print redis_cli.hvals('hash')


def addSet():
    redis_cli.sadd('set', {'member1': 'value1', 'member2': 'value2', 'member3': 'value3'})
    redis_cli.sadd('set', 'member4', 'member5')
    print redis_cli.smembers('set')
    print redis_cli.scard('set')


def addSortedSet():
    redis_cli.zadd('SortedSet', memeber1=100, memeber2=80, memeber3=60, memeber4=30)
    print redis_cli.zcount('SortedSet', min=20, max=90)
    print redis_cli.zrangebyscore('SortedSet', min=0, max=90)


if __name__ == '__main__':
    redis_cli = redis.Redis(host='10.165.124.10', port=16379, db=0, password='123456')
    redis_cli_db1 = redis.Redis(host='10.165.124.10', port=16379, db=1, password='123456')

    # base opeartion for String List Set Hash SortedSet
    # addString()
    # addList()
    # addHash()
    # addSet()
    # addSortedSet()

    # use pipeline 大量写入
    for i in range(13000, 13050):
        string_key = 'stringkey' + str(i)
        list_key = 'listkey' + str(i)
        set_key = 'setkey' + str(i)
        hash_key = 'hashey' + str(i)
        z_key = 'zkey' + str(i)

        print "========= add keys starts ========"

        redis_cli_db1.set(string_key, 'string value1')
        print ("add string key %s, value %s", string_key, redis_cli_db1.get(string_key))

        redis_cli_db1.rpush(list_key, 'list value1','list value2', 'list value3')
        print ("add list key %s, value %s", list_key, redis_cli_db1.lrange(list_key, start=0, end=-1))

        redis_cli_db1.sadd(set_key, 'member1', 'member2', {'member1': 'value1', 'member3': 'value3'})
        print("add set key %s, value %s", set_key, redis_cli_db1.smembers(set_key))

        redis_cli_db1.hmset(hash_key, {'field1': 'value1', 'field2': 'value2', 'field3': 'value3'})
        print("add hash key %s, keys = %s values = %s", \
              hash_key, redis_cli_db1.hkeys(hash_key), redis_cli_db1.hvals(hash_key))

        redis_cli_db1.zadd(z_key, memeber1=100, memeber2=80, memeber3=60, memeber4=30)
        print("add sorted key %s, keys = %s values = %s", \
              z_key, redis_cli_db1.zremrangebyscore(z_key, min=0, max=90))
```

#### 其他
- **Ruby：**Redis-rb
- **PHP:** phpredis
- **C++:**hiredis.h

#### Reference
- **[Redis官网](http://redis.io/)**
- **[命令大全](http://doc.redisfans.com/index.html)**
- **[Github讨论区](https://github.com/antirez/redis/issues)**
- **书籍：** *Redis in Action*
 






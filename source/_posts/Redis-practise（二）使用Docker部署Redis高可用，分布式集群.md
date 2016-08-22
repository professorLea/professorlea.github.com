---
title: Redis practise（二）使用Docker部署Redis高可用，分布式集群
date: 2016-08-22 13:46:31
tags: Redis
---

####  思路

鉴于之间学习过的Docker一些基础知识，这次准备部署一个简单的分布式，高可用的redis集群，如下的拓扑
![Alt text](tuopu.png)

下面介绍下，对于这张拓扑图而言，需要了解的一些基础概念。

#### Redis持久化

Redis有两种持久化策略。

**Rdb**
- 全量备份
- 形成二进制文件: dump.rdb

在使用命令 **save（停写）**, **BgSave**。或者Save配置条件触发时，开始全量持久化Rdb文件。
相关的Redis.conf配置有：
- dir
- dbfilename
- save（停写）
 例如：
 
```powershell
save 900 1  #after 900 sec (15 min) if at least 1 key changed
save 300 10 #after 300 sec (5 min) if at least 10 keys changed
save 60 10000 #after 60 sec if at least 10000 keys changed
dbfilename dump.rdb # The filename where to dump the DB
dir ./ # The DB will be written inside this directory
```

**aof**
- Append only file
- 增量备份

**Redis.conf配置**
```
appendonly yes
appendfilename "appendonly.aof" # The name of the append only file (default: "appendonly.aof")
```

那么aof和Rdb的一些对比呢：

| Col1      |     rdb|   aof   |
| :-------- | :--------| :------ |
| 性能影响| 平时不影响性能<br>备份时性能影响较大 |  性能一直会受到影响|
| 数据丢失 | 两次备份之间的增量数据会丢失|不会丢失数据|
| 备份文件大小|只有数据，较小 | 除了数据还记录了数据的变化过程，较大|

> 可参考 [官方文档](http://redis.io/topics/persistence)

#### Redis高可用

从两个角度来考虑高可用：主从复制，主从切换

##### 主从复制
可以通过命令*SLAVEOF*，来配置当前节点是某个Redis的Slave节点。参考 [slaveof](http://redisdoc.com/server/slaveof.html)

##### 主从切换
使用官方的Redis-sentinel可以实现Redis的主从切换。
参考[Redis官方文档](http://ifeve.com/redis-sentinel/) 和 [sentinel.md](https://github.com/antirez/redis-doc/blob/master/topics/sentinel.md)

值得注意的是，配饰抖个Sentinel实例，他们之间是**互相通信**的。官方推荐：一个健康的集群部署，至少需要3个Sentinel实例。这样他们之间会通过**quorum**参数来执行主从切换操作ODOWN。
- SDOWN：主观离线，Sentinel发现redis挂了。
- ODOWN：客观离线，Sentinel根据规则投票后确定redis挂了
- 规则为 #of SDOWN>quorum

启动Sentinel

```powershell
redis-sentinel /path/to/sentinel.conf
```

在本文的例子中，Sentienl的配置文件如下所示：
sentinel 实例1：

```powershell
port 26379
daemonize yes
logfile "/var/log/redis/sentinel_26379.log"
sentinel monitor firstmaster 10.165.124.10 16379 2
sentinel auth-pass firstmaster 123456
sentinel config-epoch firstmaster 0
sentinel leader-epoch firstmaster 0
sentinel monitor sencondmaster 10.165.124.10 16381 2
sentinel auth-pass sencondmaster 123456
sentinel config-epoch sencondmaster 0
sentinel leader-epoch sencondmaster  0
```
sentinel 实例2：

```powershell
port 26380
daemonize yes
logfile "/var/log/redis/sentinel_26380.log"
sentinel monitor firstmaster 10.165.124.10 16379 2
sentinel auth-pass firstmaster 123456
sentinel config-epoch firstmaster 0
sentinel leader-epoch firstmaster 0
sentinel monitor sencondmaster 10.165.124.10 16381 2
sentinel auth-pass sencondmaster 123456
sentinel config-epoch sencondmaster 0
sentinel leader-epoch sencondmaster  0
```

sentinel 实例3：

```powershell
port 26381
daemonize yes
logfile "/var/log/redis/sentinel_26381.log"
sentinel monitor firstmaster 10.165.124.10 16379 2
sentinel auth-pass firstmaster 123456
sentinel config-epoch firstmaster 0
sentinel leader-epoch firstmaster 0
sentinel monitor sencondmaster 10.165.124.10 16381 2
sentinel auth-pass sencondmaster 123456
sentinel config-epoch sencondmaster 0
sentinel leader-epoch sencondmaster  0
```

可以看出，3个实例监控了 firstmaster及secondmaster，两个集群。
对于其中一个sentinel实例，看到它的信息如下：

```powershell
$ redis-cli -h  10.165.124.10 -p 26380 -a 123456
10.165.124.10:26380> info
# Server
redis_version:2.8.24
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:7de005b109aa3dd5
redis_mode:sentinel
os:Linux 3.16.0-0.bpo.4-amd64 x86_64
arch_bits:64
multiplexing_api:epoll
gcc_version:4.7.2
process_id:28837
run_id:630dfd8f09105bc84147923319afbf1d5cebe9a5
tcp_port:26380
uptime_in_seconds:347096
uptime_in_days:4
hz:13
lru_clock:12213499
config_file:/etc/redis/sentinel_26380.conf

# Sentinel
sentinel_masters:2
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
master0:name=firstmaster,status=ok,address=10.165.124.10:16379,slaves=1,sentinels=3
master1:name=secondmaster,status=ok,address=10.165.124.10:16381,slaves=1,sentinels=3
```
最后2行可以看出监控的集群，其余2个sentinel实例也是如此。

下面讲，如何在这样的集群上，通过Jedis来进行集群操作

##### JedisSentinelPool

它解决了痛点：
- 不确定主redis 的ip port
- 需要从sentinel获取

**JedisSentinelPool**可以直接想sentinel查询当前master的ip port，在建立连接。

```java
package com.netease.nlp;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPoolConfig;
import redis.clients.jedis.JedisSentinelPool;

import java.util.HashSet;
import java.util.Set;

public class JedisSentinelTest {
    public static Set<String> sentinels = new HashSet<String>();
    public static JedisPoolConfig config = new JedisPoolConfig();
    static{
        config.setMaxTotal(100);
        config.setMaxIdle(100);
        config.setMaxWaitMillis(1000);
        sentinels.add("10.165.124.10:26379");
    }
    public static JedisSentinelPool sentinelPool = new JedisSentinelPool("firstmaster", sentinels, config);

    public static void main(String[] args){
        Jedis jedis = sentinelPool.getResource();
        try{
            jedis.auth("123456");
            jedis.set("sentinel_one", "sentinel_one");
            System.out.println(jedis.get("sentinel_one"));
        }finally {
            if (jedis!= null){
                jedis.close();
            }

        }
    }
}
```

从代码里可以看出，我们的Jedis链接是通过Sentinel来获取的。

#### 分布式
解决痛点：
- 用户较多数据量大，一台服务器内存不足。
- 部署多台增加容量，但是需要考虑如何分割数据

Redis的分布式使用的是**一致性哈希**算法。满足：
- 分散性：避免相同的内容映射到不同节点
- 平衡性：均匀分布，每个节点上的内容数量差不多
- 单调性：增删节点时，不影响旧的映射。

一致性哈希的原理如图所示：
![Alt text](haxi.png)

可参考：[中文介绍](http://blog.csdn.net/cywosp/article/details/23397179)，[英文介绍](http://www.martinbroadhurst.com/Consistent-Hash-Ring.html)

有了算法，那么数据分片在Redis中还有3种分类：

**1，客户端分片**
使用比如ShardedJedis，Predis，Redis-rb等客户端的包，在客户端金鑫该数据分片。
![Alt text](shardJedis.png)

在本例中，使用ShardedJedisPool，代码如下：

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
可以看出，两个Master都在一个shardedJedisPool里。但是这里有个问题就是需要维护Master的地址，所以后续如果可能，也需要开发Sentinel管理的JedisPool的插件。

**2，代理分片**
Twemproxy, codis, onecache。如图：
![Alt text](proxy.png)
特点：
- 服务端计算分片，客户端简单
- 客户端无需维护redis的ip
- Proxy会增加响应时间

**3，查询路由**
Redis 3.0:Redis集群自己做好分片。
特点：
- 无需proxy
- 客户端可以记录下每个key对应的redis以增加性能
- 支持3.0 的客户端还很少

docker 镜像：
https://hub.docker.com/r/tutum/redis/
https://github.com/tutumcloud/redis

#### Redis监控统计

**Info ｛options｝**
- Server/Clients/Memory/Persistence/Stats/Replications/Cpu/Commandstats/Keyspaces

**常用的**
- Memory : used_memory_human 当前使用内存量
- Memory : mem_fragmentation_ratio 内存碎片率
- Stats : instantaneous_ops_per_sec 实时的QPS
- Stats: expired_keys, evicted_keys, keyspace_hits, keyspace_misses 过期的key，被置换的key，命中的key数量，未命中的key数量
- Replication 主从连接是否正常，复制是否正常
- Commandstats 每个命令的执行情况
- Keyspaces 每个db上有多少key


#### 彩蛋
##### 网易内部的一款产品的架构图：
单点式： 一主多从
![Alt text](single.png)

分布式：
- Redis：多个Redis节点，每个节点都一主一从。
- Redis-sentinel: 主从探获、切换。
- Proxy：业务代理转发，数据分片
- NLB：负载均衡，Proxy去单点
![Alt text](multi.png)

##### 部署Redis的Docker file
```dockerfile
FROM tutum/redis:latest

MAINTAINER Xiaolei Li "professorLea@163.com"

# redis configuration
ENV REDIS_PASS "123456"
ENV REDIS_MAXMEMORY_POLICY="allkeys-lru"
ENV REDIS_MAXMEMORY="10mb"
ENV REDIS_DATABASES="2"
ENV REDIS_APPENDONLY="yes"
ENV REDIS_APPENDFSYNC=everysec
```
**启动容器：**
集群1：

```powershell
docker run -d --name master1 -p 16379:6379 professorlea/redis
docker run -d --name slave1 -p 16380:6379 -e -e REDIS_MASTERAUTH="123456" professorlea/redis
```

集群2：

```powershell
docker run -d --name master2 -p 16381:6379 professorlea/redis
docker run -d --name slave2 -p 16382:6379 -e REDIS_MASTERAUTH="123456" professorlea/redis
```

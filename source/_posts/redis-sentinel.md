---
title: Redis High Available (一)
tags:
  - redis
  - 中间件
  - 高可用
categories:
  - Redis
index_img: /images/redis-logo.png
date: 2022-01-18 22:43:47
mermaid: true
---

{% note info %}
如何配置redis哨兵集群?
{% endnote %}

<!-- more -->


# 哨兵(Sentinel)

> 运行在特殊模式的Redis Server

仅包含了如下功能：

- 监控：监控主从服务器状态
- 通知：实例异常通知
- 故障转移：主服务器故障，重新选择新的主服务器
- 配置提供者：用户获取主服务器信息

# 哨兵集群

哨兵集群是Redis高可用的解决方案之一，对于一主多从场景，当主服务器发生故障时，能够自动的进行故障转移，选举新的主服务，减少人工参与，增强系统的健壮性。

## 配置

### 搭建一主多从

> 本例以一主三从为例

![图片1-一主多从](/images/redis/redis-cluster.png)

``` shell
redis-server
redis-server ./replica[1|2|3]
```

replica.conf
``` conf
replicaof 127.0.0.1 6379 
```

### 配置哨兵集群

> 至少需要3个哨兵实例，才能达到高可用目的

![图片2-哨兵集群](/images/redis/redis-monitor.png)

``` shell
redis-sentinel ./sentinel1.conf
```

sentinel.conf
``` conf
sentinel monitor mymaster 127.0.0.1 6379 2
```

### 哨兵命令介绍

sentinel monitor mymaster 127.0.0.1 6379 2
redis-server /path/to/sentinel.conf --sentinel
redis-sentinel /path/to/sentinel.conf
redis-sentinel ./sentinel1.conf

# 客观下线

## 探活

哨兵实例会以每秒一次的频率向所有与它连接的实例（主/从服务器、其他哨兵实例）发送*PING*指令。
有效回复为:

- *+PONG*
- *-LOADING*
- *-MASTERDOWN*

其他的任何回复都被认为是无效回复。若在**down-after-milliseconds**内一直收到无效回复，则该实例会被标记为**主观下线**

## 主观下线 & 客观下线

主观下线：哨兵实例自身判定主服务器已下线
客观下线：哨兵集群中 **>= quorum** 的哨兵主观判断主服务器下线	

当某一哨兵实例判定主服务器主观下线时，它会向监视该主服务器的其他哨兵实例询问（*SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>*），并对收到的回复进行汇总，当 **>= quorum** 的哨兵判断主服务器下线时，则标记该主服务器为客观下线。

# leader选举

![图片3-leader选举](/images/redis/redis-leader.png)

> 当一个主服务器被判断为客观下线时，监视该主服务器的各个哨兵会进行协商，选出leader实例进行故障转移

1. 哨兵leader申请
- 每个实例都可以做为leader
- 每一个配置纪元（epoch），只能设置一次leader
- 实例发现主服务器客观下线时，当前epoch加1，向其他哨兵发送leader申请（即默认同意自己为leader）


2. 当哨兵接受到leader申请时
- 若当前纪元已同意过其他leader，则返回其leader信息
- 若当前纪元未同意过，则设置该哨兵为leader，并返回该leader信息

3. 发出leader申请的哨兵，统计同意自己为leader的实例数量，若数量 **>= (N/2 + 1)**，且**>= quorum** 时则它成功被推选为leader

4. 若给定时间内，没有leader胜出，则在一段时间后重新选举（时间为failover_timeout\*2，默认6min)

{% note info %}
SENTINEL IS-MASTER-DOWN-BY-ADDR <ip> <port> <current-epoch> <runid>  runid不是 \* 时，则代表leader申请
{% endnote %}


# 故障转移-选择新主

![图片4-故障转移](/images/redis/redis-new-main.png)

当选举出哨兵leader后，leader sentinel会对下线的主服务器执行故障转移操作，主要有三个步骤：

1. 它会选择旧Master的其中一个Slave升级为新的Master, 并让失效Master以及其他Slave改为复制新的Master
2. 当客户端试图连接失效的 Master 时，集群也会向客户端返回新 Master 的地址，使得集群可以使用新的 Master 替换失效 Master
3. 主从服务器切换后，所有实例的配置文件会发生相应的更新。


## 新主选择规则

- 过滤掉主观下线的节点
- 选择slave-priority最高的节点，如果有则返回，没有继续选择
- 选择出复制偏移量最大的节点，因为复制偏移量越大则数据复制的越完整，如果有则返回，没有继续选择
- 选择run_id最小的节点，因为run_id越小说明重启次数越少

# 总结

诚然，单个主Redis Server自然无法满足大数据时代的要求，Redis在3.0版本开始支持[Redis Cluster](https://redis.io/docs/manual/scaling/).

# 参考文献

[1] [https://redis.io/docs/manual/sentinel/](https://redis.io/docs/manual/sentinel/)
[2] [https://www.cnblogs.com/myd620/p/7811156.html](https://www.cnblogs.com/myd620/p/7811156.html)























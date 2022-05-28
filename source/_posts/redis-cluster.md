---
title: Redis High Available (二)
date: 2022-05-28 14:05:55
tags:
  - redis
  - 中间件
  - 高可用
categories:
  - Redis
index_img: /images/redis-logo.png
mermaid: true
---

{% note info %}
如何配置redis集群?
{% endnote %}

<!-- more -->

# 简介

**Redis集群（Cluster）** 提供了数据分片能力，减轻单主（或单主多从）大并发下的读写压力，同时提供灵活的横向纵向扩容。
配合哨兵集群，更大程度上实现性能与吞吐量双重提升。

# 配置流程

## redis-nodex.conf

``` shell
port 6001
cluster-enabled yes
cluster-config-file nodes-xx.conf
cluster-node-timeout 5000
```

## 开始集群

``` shell
# 启动Redis Node
redis-server ./redis-nodex.conf
# 注册集群，本文使用四个node 600[1-4]
redis-cli --cluster create 127.0.0.1:6001 127.0.0.1:6002 127.0.0.1:6003 127.0.1:6004
```

## 测试

``` shell
# 查看集群状态
redis-cli --cluster check 127.0.0.1:6001
# 查看集群节点
redis-cli -p 6001 cluster nodes
# 连接集群
redis-cli -c -p 6001
```

![图片1-集群存值](/images/redis/redis-cluster-set-value.png)


# 分片策略

**Redis Cluster** 引入哈希槽（hash slot）的概念，Redis集群有16384个哈希槽，每个key经CRC16校验后对16384取模来决定放置哪个槽，集群的每个节点负责一部分哈希槽。

```
crc16(key) mod 16384
```

使用哈希槽的优点：
- key与哈希槽关联，提高迁移效率
- 便于删除

{% note info %}
[为何哈希槽设值为16384个？](https://github.com/redis/redis/issues/2576)
{% endnote %}

# 宕机处理

## 多主无从

> 主节点不设置从节点

此时主节点宕机，**Redis Cluster**默认不可用，但可以通过修改配置，使得其他节点仍旧可以正常处理请求
```shell
cluster-require-full-coverage no
```

## 多主多从

多主多从的场景，就可以配合上篇文章讲到的哨兵模式，进行快速的主从切换，达到更高的可用性。

{% note info %}
对于集群场景，多主多从还是更加推荐的。特别是对于微服务来说，一方面主从切换自动化可以提高服务稳定性，另一方面集群模式有利于提高查询效率。
{% endnote %}

# 增加/删除节点

``` shell
# 增加节点
redis-cli --cluster add-node 127.0.0.1:6004 127.0.0.1:6001
# 增加从节点
redis-cli --cluster add-node 127.0.0.1:6005 127.0.0.1:6001 --cluster-slave
# 删除节点
redis-cli --cluster del-node 127.0.0.1:7000 `<node-id>`
# 删除节点前，需要利用resharding保证删除节点是空的
# 新增节点后，需要利用resharding分配槽
redis-cli --cluster reshard 127.0.0.1:7000
```

## 哈希槽迁移

哈希槽迁移时，集群依旧可以正常处理请求，主要思想类似于Redis Hash的扩容操作，简单概述为：
- 迁出Node与迁入Node会分别标记为 **MIGRATING**、**IMPORTING**
- **MIGRATING**状态的Node可以正常处理迁移槽内的请求，如果请求key已经被迁移，则会告诉client向新节点（**IMPORTING**）使用ASKING发起请求
- **IMPORTING**状态的Node，节点仅在接收到ASKING 命令之后，才会接受关于这个槽的命令请求，若非ASKING则转回原Node

{% note info %}
**Redis Cluster**数据迁移是同步的，在目标节点执行restore指令到源节点删除key之间，源节点主线程处于阻塞状态，直达key 被成功删除
{% endnote %}

# 小结

本文简述了Redis集群的基本配置方法，详细的信息可以参考[官方文档](https://redis.io/docs/manual/scaling/#redis-cluster-101)

# 参考链接

1. [Redis集群如何保证高可用？](https://blog.51cto.com/u_15351425/3741065)
2. [Redis集群槽的迁移原理分析](https://blog.csdn.net/keep_learn/article/details/106834425)
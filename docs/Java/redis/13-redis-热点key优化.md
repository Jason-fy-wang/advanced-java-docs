---
tags:
  - redis
  - hotkey
---
这是redis官方的benchmark的结果[benchmark](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/)可以看到qps可以达到10+w. 当然了针对真实的redis-cluster需要再次进行测试，以此作为自己真实环境的容量。

![](./images/redis-cluster)



当搭建好redis-cluster集群后，其会有16374个slot，key通过 hash(key) % 16384 来获取slot。

> 问题是: 单机的key操作达到10w+, 那么当流量达到20w+ 或更高时, 此类型的hotkey要如何优化?

1) 通过useId 来让key 分布在不同的 slot
如果此key保存的是商品数量，那么针对此类型，可以把商品数量分开存储到集群中各个节点上，当用户过来时，通过 hash(userID) % number(nodes) 来获取来操作的node， 以此来提高整个集群的处理能力。


2) 通过hash 标签 {} 控制slot 分布
Redis cluster中对key中{} 内部的  字符串计算slot， 称为 hash tag
```shell
SET user:{1001}:profile  "xxxx"
SET user:{1002}:profile  "yyyy"

## 查看key的 slot
redis-cli -c cluster keyslot yourkey

## slot 分布情况
redis-cli -c cluster nodes


## 如何发现hot key？
redis-cli monitor  (生成慎用!!!)
redis-cli --latency
relic-cli --stat

## 内部统计
redis-cli --bigkeys
redis-cli --scan
redis-cli info keyspace

```











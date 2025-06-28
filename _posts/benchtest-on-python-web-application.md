---
title: Python Web 应用并发测试
date: 2024-08-11 16:59
---

网上常听到这样一种论调：Python 太慢，因而不适合做 Web 开发。为了详细了解 Python 在实用场景下的性能，笔者做了一个简单的博客系统，支持简单的博客展示，提供上传、修改的接口，然后尝试在迭代过程中提高并发性能。

项目已开源在 [GitHub](https://github.com/HairlessVillager/myblog)。这个系统最终的实现是 MongoDB + Redis + local cache 作为分级存储，尽可能使用异步逻辑，使用 Uvicorn 的多进程提高效率。为了直观地看到性能的变化，本文按照倒序的方式来编排。

测试的具体实现就是用 locust 模仿若干用户并发访问一个指定的博客，见项目下的 `bench.py` 文件。

测试硬件条件：
- CPU：2 核
- 内存：2GB
- 带宽：100Mbps

## 性能极限

![](/images/benchtest-on-python-web-application/ef25cb1-2000.jpeg)

用最终实现测得极限性能，前期最大 RPS 在 2500 左右，然后系统变得非常不稳定，此时有 2000 个用户。因此以下测试均使用 1000 个用户作为测试条件。

## 最终实现

![](/images/benchtest-on-python-web-application/ef25cb1-1000.jpeg)

[ef25cb1](https://github.com/HairlessVillager/myblog/tree/ef25cb13541aee3a749c835d12b39f3f6c1e041a)

- RPS ~ 2300
- RTp50 ~ 200ms
- RTp95 ~ 1300ms

后面一段异常的数据可能是压测主机不太稳定。

## 移除本地缓存

![](/images/benchtest-on-python-web-application/84f2587-1000.jpeg)

[84f2587](https://github.com/HairlessVillager/myblog/tree/84f2587b874a82efa7dc1cce514e333c21395acd)

- RPS ~ 1300
- RTp50 ~ 300ms
- RTP95 ~ 3300ms

本地缓存（local cache）其实就是 Python 里一个字典类型的全局变量。移除后的最高层级缓存是 Redis 实例，在读取数据时必须经过网络通信，所以这里的性能下降了很多。

中间一段异常数据原因暂未知，忘记拿日志了😭

## 移除 Redis 缓存

![](/images/benchtest-on-python-web-application/b2b6168-1000.jpeg)

[b2b6168](https://github.com/HairlessVillager/myblog/tree/b2b61685406f56167896eb382920c06e0a2b954d)

- RPS ~ 1000
- RTp50 ~ 300ms
- RTp95 ~ 3300ms

移除 Redis 缓存后，所有读请求必须经过 MongoDB 数据库，但是这里的 RPS 并没有降低多少。猜测原因是无论从 Redis 读还是从 MongoDB 读，大部分的时间都花在了网络通信上。

## 把 MongoDB 换成 Postgres

![](/images/benchtest-on-python-web-application/9ae9101-1000.jpeg)

[9ae9101](https://github.com/HairlessVillager/myblog/tree/9ae91017b1e48e0b4f86cb969b8546b819d86f56)

- RPS ~ 510
- RTp50 ~ 2000ms
- RTp95 ~ 5000ms

MongoDB 属于 NoSQL 数据库。相比于传统的关系型数据库，NoSQL 不支持事务，没有 Schema，只保证最终一致性，好处是带来了一定的性能提升。可以看到把数据库换成关系型数据库后，性能显著下降。

当然 Postgres 也有 NoSQL 特性，这里没有启用。

## 多进程变单进程

![](/images/benchtest-on-python-web-application/29fc343-1000.jpeg)

[29fc343](https://github.com/HairlessVillager/myblog/tree/29fc343653cfd63167a2880a7ad07238dab600a3)

- RPS ~ 220
- RTp50 ~ 4000ms
- RTp95 ~ 12000ms

服务器只有两个核心，这里我只开了两个进程。变成单进程后性能确实减半了。

## 移除异步

![](/images/benchtest-on-python-web-application/fae58bb-1000.jpeg)

[fae58bb](https://github.com/HairlessVillager/myblog/tree/fae58bb3ad794559e899ee1f262344fca3a9b13c)

- RPS ~ 180
- RTp50 ~ 6000ms
- RTp95 ~ 6000ms

这个结果比较奇怪，为什么移除异步后 RPS 变化还没有 RTp95 变化大。

## 监控

最后附上服务器的监控。

![](/images/benchtest-on-python-web-application/Screenshot-2024-09-10-115850.png)

![](/images/benchtest-on-python-web-application/Screenshot-2024-09-10-115933.png)

![](/images/benchtest-on-python-web-application/Screenshot-2024-09-10-115949.png)
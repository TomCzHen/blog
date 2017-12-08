title: 阿里云ECS自建Redis性能测试
date: 2016-04-10 13:28:00
categories: 
  - Database
feature: /images/logo/redis-logo.webp
tags: 
  - Redis
  - ECS
toc: true
---

项目需要使用 Redis 数据库，需要在自建实例与云实例间做出选择，所以进行了测试。由于项目不需要数据持久化，因此没有测试持久化 IO 相关性能。

<!-- more -->

<h2 id="env">测试环境说明</h2>

<h3 id="hardware">硬件环境</h3>

* **服务器**
    * 实例系列 : 系列I
    * 实例规格 : 2 核 2GB
    * 公网带宽 : 10Mpbs
    * 操作系统 : FreeBSD 10.1 64位

* **内网客户端**
    * 实例系列 : 系列 I
    * 实例规格 : 2 核 4GB
    * 操作系统 : Windows Server 2012 标准版 64位

* **外网客户端**
    * 硬件配置 : Intel Core i5 4200U / DDR3L 1600 8G
    * 上行带宽 : 4Mpbs
    * 操作系统 : Windows 10 64位

* **云数据库Redis** 

    * 存储容量 : 1G

<h3 id="software">软件环境</h3>

* Redis 3.0.2

配置为最大内存可用1024mb,其余为默认配置.

* Python 2.7

<h3 id="code">测试脚本</h3>

分别使用单线程,2 线程,4 线程增加 10 万个 TTL 值为 60 秒的 key , key 名使用 UUID , value 统一为 0 .

```
# -*- coding: utf-8 -*-

import uuid
import redis
from threading import Thread
from timeit import timeit

redis_host = "127.0.0.1"
redis_port = 6379
redis_pwd = ""

threads_num = 4
keys_num = 25000
threads = []


def set_redis_key(num):
    r = redis.StrictRedis(host=redis_host, port=redis_port, password=redis_pwd)

    def __gen_keys_values(i):
        yield 'python' + ':' + str(uuid.uuid4())
        yield ''

    for k, v in tuple(map(__gen_keys_values, range(num))):
        r.set(name=k, value=v, ex=60, px=None, nx=False, xx=False)


def start():
    for i in range(threads_num):
        t = Thread(target=set_redis_key, args=(keys_num,))
        threads.append(t)

    for i in range(threads_num):
        threads[i].start()

    for i in range(threads_num):
        threads[i].join()


if __name__ == "__main__":
    cost_time = timeit('start()', 'from __main__ import start', number=1)
    print('耗时 {} 秒。'.format(cost_time))
```

<h2 id="result">测试结果</h2>

注意 : 云数据库 Redis 仅支持从内网访问

<h3 id="table">测试数据</h3>

<h4 id="redisonecs">ECS 自建 Redis 实例</h4>

* **ECS 本地**
    * 1 线程 : 22 秒
    * 2 线程 : 18 秒
    * 4 线程 : 18 秒

* **内网客户端**
    * 1 线程 : 44秒
    * 2 线程 : 33秒
    * 4 线程 : 30秒

* **外网客户端**
    * 1 线程 : 471 秒
    * 2 线程 : 312 秒
    * 4 线程 : 154 秒

<h4 id="redisonyun">云数据库 Redis 实例</h4>

* **内网客户端**
    * 1 线程 : 191 秒
    * 2 线程 : 106.5 秒
    * 4 线程 : 55 秒

<h3 id="expiry">结论</h3>

阿里云云数据库 Redis 存储容量 2G 实例每年费用为 2700 元,2 核 4GB 系列 II 实例价格为 2160 元.(注 : 系列 II 的性能比 系列 I 更好.)
云数据库 Redis 的优势是自带集群并且免维护,但性价比低.自建 Redis 实例性价比高,但需要自行维护并且没有集群(需要自建集群).
如果对性能要求高,并且没有持久化的需求(持久化 IO 并未测试)选择自建实例.如果对可靠性要求高,有其他复杂应用运维有难度可以直接选择云数据库实例.

<h3 id="other">后话</h3>

批量增加大量 key 时可以使用管道和连接池来提高效率，python 代码 demo 如下：
```
# -*- coding: utf-8 -*-
import uuid
import redis
import threading


class RedisBench(object):
    def __init__(self, host, port=6379, pwd=""):
        self.pool = redis.ConnectionPool(host=host, port=port, password=pwd)

    def __init_pool(self):
        pool = redis.Redis(connection_pool=self.pool)
        return pool

    def set_key(self, keys_num):
        j = 0
        connect_pool = self.__init_pool()
        pipe = connect_pool.pipeline()

        def __gen_keys_values(i):
            yield 'python' + ':' + str(uuid.uuid4())
            yield ''

        for k, v in tuple(map(__gen_keys_values, range(keys_num))):
            pipe.set(name=k, value=v, ex=60, px=None, nx=False, xx=False)

        pipe.execute()

    def lpush(self, values_num):
        connect_pool = self.__init_pool()
        pipe = connect_pool.pipeline()
        tuple_values = tuple(range(values_num))
        pipe.lpush("key_list", *tuple_values)
        pipe.execute()

    def start(self):
        configs = [{"test": self.set_key, "threads_num": 1, "keys_num": 10000}]

        threads = []

        for config in configs:
            threads.append([])
            for i in range(config['threads_num']):
                t = threading.Thread(target=config['test'], args=(config['keys_num'],))
                threads[configs.index(config)].append(t)

            for i in range(config['threads_num']):
                threads[configs.index(config)][i].start()

            for i in range(config['threads_num']):
                threads[configs.index(config)][i].join()


if __name__ == "__main__":
    redis_host = "127.0.0.1"
    redis_port = 6379
    redis_pwd = ""

    my_redis_bench = RedisBench(host=redis_host, pwd=redis_pwd)
    my_redis_bench.start()

```
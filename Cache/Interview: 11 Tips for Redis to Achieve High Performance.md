Interview: 11 Tips for Redis to Achieve High Performance
========================================================

[![Dylan Smith](https://miro.medium.com/v2/resize:fill:88:88/1*4oAGfnElULhbcKXWArnexA@2x.jpeg)](https://medium.com/@junfeng0828?source=post_page-----5e8bf18eebb7--------------------------------)

[![Stackademic](https://miro.medium.com/v2/resize:fill:48:48/1*U-kjsW7IZUobnoy1gAp1UQ.png)](https://blog.stackademic.com/?source=post_page-----5e8bf18eebb7--------------------------------)

[Dylan Smith](https://medium.com/@junfeng0828?source=post_page-----5e8bf18eebb7--------------------------------)


![](https://miro.medium.com/v2/resize:fit:700/1*0vF-np9zTQsAISAWB9qN5w.png)

> My articles are open to everyone; non-member readers can read the full article by clicking this [link](https://medium.com/@junfeng0828/5e8bf18eebb7?sk=a1164e8058bb570aa32ae9b886f92958).

When your system decides to use Redis, presumably it is because of its excellent performance.

We know that a single-node Redis can handle a traffic volume of up to 100,000 QPS. Such high performance also means that if there is latency during use, it will not meet our expectations.

Therefore, when using Redis, how to continuously exert its high performance and avoid the occurrence of operation latency is also our focus of attention.

In this regard, I have summarized 11 suggestions for you:

1\. Avoid storing bigkeys.
==========================

Storing bigkeys not only uses excessive memory as mentioned earlier but also has a great impact on the performance of Redis. Since Redis processes requests in a single thread, when your application writes a bigkey, more time will be consumed in "memory allocation", and at this time, the operation latency will increase.

Similarly, when deleting a bigkey, it will also take time during "memory release". Moreover, when you read this bigkey, more time will be spent on "network data transmission". At this time, the subsequent requests to be executed will queue up, and the performance of Redis will decline.

![](https://miro.medium.com/v2/resize:fit:700/1*YrXAeiRza_vOwU09faZegQ.png)

So, your business application should try not to store bigkeys to avoid operation latency.

> If you do indeed have a need to store bigkeys, you can split the bigkey into multiple small keys for storage.

2\. Do not use commands with overly high complexity.
====================================================

Redis processes requests in a single-threaded model. In addition to the situation where subsequent requests are queued up when operating bigkeys, this can also happen when executing commands with overly high complexity.

Because executing commands with overly high complexity will consume more CPU resources, and other requests in the main thread can only wait, and at this time, queuing latency will also occur.

So, you need to avoid executing aggregation commands such as SORT, SINTER, SINTERSTORE, ZUNIONSTORE, and ZINTERSTORE.

For this kind of aggregation operation, I suggest you perform it on the client side and not let Redis bear too much computational work.

3\. Pay attention to the time complexity of the DEL command.
============================================================

![](https://miro.medium.com/v2/resize:fit:700/0*v-41xaVpX7JQfD-R)

Photo by [Sam Pak](https://unsplash.com/@sampakpak?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

When deleting a key, if the operation is not correct, it may also affect the performance of Redis.

When deleting a key, we usually use the DEL command. Recall, what do you think is the time complexity of DEL?

O(1)? Actually, not necessarily.

When you delete a String-type key, the time complexity is indeed O(1).

But when the key you want to delete is of List/Hash/Set/ZSet type, its complexity is actually O(N), where N represents the number of elements.

That is to say, when deleting a key, the more elements there are, the slower the execution of DEL!

The reason is that when deleting a large number of elements, the memory of each element needs to be recycled in turn. The more elements there are, the longer it takes!

Moreover, this process is executed in the main thread by default, which will inevitably block the main thread and cause performance issues.

So, how should we handle deleting a key with relatively many elements?

My suggestion to you is to delete in batches:

-   For List type: Execute LPOP/RPOP multiple times until all elements are deleted.
-   For Hash/Set/ZSet type: First execute HSCAN/SSCAN/SCAN to query elements, and then execute HDEL/SREM/ZREM to delete each element in turn.

Surprised? A small deletion operation, if not careful, may also cause performance issues. You need to be extremely careful when operating.

4\. Enable the lazy-free mechanism.
===================================

If you cannot avoid storing bigkeys, then I suggest you enable Redis's lazy-free mechanism. (Supported in version 4.0+).

When this mechanism is enabled, when Redis deletes a bigkey, the time-consuming operation of releasing memory will be executed in a background thread, which can avoid the impact on the main thread to the greatest extent.

![](https://miro.medium.com/v2/resize:fit:700/1*vnuFY0X5W2dxuU2GpHDwEg.png)

5\. When executing O(N) commands, pay attention to the size of N.
=================================================================

Is it possible to rest easy after avoiding using commands with overly high complexity?

The answer is no.

When you are executing O(N) commands, you also need to pay attention to the size of N.

If you query too much data at one time, it will also take a long time in the network transmission process and increase operation latency.

So, for container types (List/Hash/Set/ZSet), when the number of elements is unknown, never blindly execute commands such as `LRANGE key 0 -1`, `HGETALL`, `SMEMBERS`, `ZRANGE key 0 -1`.

When querying data, you should follow these principles:

1.  First query the number of data elements (involving commands: LLEN, HLEN, SCARD, ZCARD).
2.  If the number of elements is small, you can query all the data at one time.
3.  If the number of elements is very large, query the data in batches (involving commands: LRANGE, HSCAN, SSCAN, ZSCAN).

6\. Use batch commands instead of individual commands.
======================================================

When you need to operate multiple keys at once, you should use batch commands to handle it.

The advantage of batch operations compared to multiple individual operations is that it can significantly reduce the number of round-trips of network I/O between the client and the server.

So my suggestion to you is:

-   For String/Hash, use MGET/MSET instead of GET/SET, and HMGET/HMSET instead of HGET/HSET.
-   For other data types, use Pipeline to pack and send multiple commands to the server for execution at one time.

![](https://miro.medium.com/v2/resize:fit:700/1*4ybuXklhXU1S3bgIeqblDA.png)

7\. Avoid concentrated expiration of keys.
==========================================

Redis cleans up expired keys in a timed and lazy way, and this process is executed in the main thread. If there are a large number of keys in your business that expire concentratedly, then when Redis cleans up expired keys, there is also a risk of blocking the main thread.

To avoid this situation, when setting the expiration time, you can add a random time to scatter the expiration times of these keys, thereby reducing the impact of concentrated expiration on the main thread.

8\. Operate Redis with long connections and configure connection pools reasonably.
==================================================================================

Your business should operate Redis with long connections and avoid short connections. When operating Redis with short connections, TCP three-way handshakes and four-way wave-offs are required each time, and this process will also increase operation time consumption.

At the same time, your client should access Redis in the way of connection pool and set reasonable parameters. When not operating Redis for a long time, connection resources should be released in time.

9\. Only use db 0.
==================

Although Redis provides 16 databases, I only recommend you to use db 0. Why? I have summarized the following three reasons:

1.  When operating multiple database data on one connection, you need to execute SELECT first every time, which will bring additional pressure to Redis.
2.  The purpose of using multiple databases is to store data according to different business lines. Then why not split and store them in multiple instances? Deploying multiple instances and splitting storage, multiple business lines will not affect each other and can also improve the access performance of Redis.
3.  Redis Cluster only supports db 0. If you want to migrate to Redis Cluster later, it will increase the migration cost.

10\. Use read-write separation + sharded cluster.
=================================================

If the read request volume of your business is very large, then you can deploy multiple slave databases to achieve read-write separation, let the slave databases of Redis share the read pressure and thereby improve performance.

![](https://miro.medium.com/v2/resize:fit:700/1*j8IrzCMI5N1F7OjWEAt_tg.png)

If the write request volume of your business is very large and a single Redis instance can no longer support such a large write traffic, then at this time you need to use a sharded cluster to share the write pressure.

![](https://miro.medium.com/v2/resize:fit:700/1*tEbu3uTVBiqBAOECKiMU1A.png)

11\. Do not enable AOF or configure AOF for disk flushing every second.
=======================================================================

For businesses that are not sensitive to data loss, I suggest you do not enable AOF to avoid the performance degradation of Redis caused by AOF writing to the disk.

If AOF does need to be enabled, then I suggest you configure it as `appendfsync everysec` and put the disk flushing operation of data persistence into a background thread for execution to minimize the impact of Redis writing to the disk on performance.

Well😄, these are the practical optimizations in terms of Redis "high performance". If you are very concerned about the performance issues of Redis, you can conduct targeted optimizations by combining these aspects.

For more Redis-related content, please read my column.

![Dylan Smith](https://miro.medium.com/v2/resize:fill:40:40/1*4oAGfnElULhbcKXWArnexA@2x.jpeg)

[Dylan Smith](https://medium.com/@junfeng0828?source=post_page-----5e8bf18eebb7--------------------------------)

Mastering Redis
---------------

[View list](https://medium.com/@junfeng0828/list/mastering-redis-04ac4dbc355d?source=post_page-----5e8bf18eebb7--------------------------------)

14 stories

![](https://miro.medium.com/v2/resize:fill:388:388/1*UVwQnxGR581r6yRm53otPA.png)

![7 Use Cases for Redis](https://miro.medium.com/v2/resize:fill:388:388/1*0MiDH5y3iMdne7sV8gp9bw.png)

![](https://miro.medium.com/v2/resize:fill:388:388/1*XOw2HCPJe47wLY9sXQQwFQ.png)

Stackademic 🎓
==============

Thank you for reading until the end. Before you go:

-   Please consider clapping and following the writer! 👏
-   Follow us [X](https://twitter.com/stackademichq) | [LinkedIn](https://www.linkedin.com/company/stackademic) | [YouTube](https://www.youtube.com/c/stackademic) | [Discord](https://discord.gg/in-plain-english-709094664682340443)
-   Visit our other platforms: [In Plain English](https://plainenglish.io/) | [CoFeed](https://cofeed.app/) | [Differ](https://differ.blog/)
-   More content at [Stackademic.com](https://stackademic.com/)

[Redis](https://medium.com/tag/redis?source=post_page-----5e8bf18eebb7---------------redis-----------------)

[Interview](https://medium.com/tag/interview?source=post_page-----5e8bf18eebb7---------------interview-----------------)

[Programming](https://medium.com/tag/programming?source=post_page-----5e8bf18eebb7---------------programming-----------------)

[Performance](https://medium.com/tag/performance?source=post_page-----5e8bf18eebb7---------------performance-----------------)

[Software Development](https://medium.com/tag/software-development?source=post_page-----5e8bf18eebb7---------------software_development-----------------)

[![Dylan Smith](https://miro.medium.com/v2/resize:fill:144:144/1*4oAGfnElULhbcKXWArnexA@2x.jpeg)](https://medium.com/@junfeng0828?source=post_page-----5e8bf18eebb7--------------------------------)

[![Stackademic](https://miro.medium.com/v2/resize:fill:64:64/1*U-kjsW7IZUobnoy1gAp1UQ.png)](https://blog.stackademic.com/?source=post_page-----5e8bf18eebb7--------------------------------)

[Written by Dylan Smith
----------------------](https://medium.com/@junfeng0828?source=post_page-----5e8bf18eebb7--------------------------------)

[1.1K Followers](https://medium.com/@junfeng0828/followers?source=post_page-----5e8bf18eebb7--------------------------------)

-Writer for

[Stackademic](https://blog.stackademic.com/?source=post_page-----5e8bf18eebb7--------------------------------)

Software Engineer for a leading global e-commerce company. Dedicated to explaining every programming knowledge with interesting and simple examples.
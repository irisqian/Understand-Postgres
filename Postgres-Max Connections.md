什么是max_connections?
max_connections决定了 当前能并发连接 到数据库的最大连接数目。
不过不要忘了，还有另外一个参数 super_reserved_connections , 这个参数控制的连接数只允许给superusers. 这样即使连接满了，还有余量给superuser进入。
默认的配置是
max_connections = 100
superuser_reserved_connections = 3,
总共还剩97个连接能够给应用使用。

为什么大家会选择使用很大数值的max_connections?
首先，因为如果需要增加max_connections，必须重启db，大家都希望避免总是重启，就会倾向于放个大的"安全值"。
其次，在运维过程中碰到：
ERROR: remaining connection slots are reserved for non-replication superuser connections
这说明已经遇到了连接数瓶颈，就会需要提高max_connections。
第三，application开发人员告诉DBA，他们必须要有更多的DB连接才能达到很好的performance。

大数值的max_connections会造成什么问题呢？
1. DB负载过高
当设置连接数很大，并且所有连接都进入active状态，那CPU和I/O系统就会负载过高。CPU会在进程中频繁切换上下文，或者等待storage的I/O。这种情况下很有可能任意一个session都没有取得什么进展，而系统却非常繁忙。然后，系统就进入僵死状态不响应，只能重启了。这种情况，增加最大连接数也并没有让application performance变好。
2. 每个连接资源不足
DB服务器的RAM是有限的。如果增加连接数，能分配给每个连接的RAM就会减少。可想而知，run out of memory就是最终的结局。在query执行过程中，每个operation能使用的内存由work_mem控制。所以如果想要增加max_connections就必须相应地减少work_mem。work_mem会直接影响查询的效率，如果有足够多的RAM，排序肯定会更快，或者Postgres会选择hash join或者hash aggregate，来避免排序。所以，把max_connections调高反而会影响查询的效率，否则可能会更糟糕，直接run out of memory。


max_connections上限值的一个经验计算公式
max_connections < max(num_cores, parallel_io_limit) / (session_busy_ratio * avg_parallelism)

- num_cores，CPU核数
- parallel_io_limit，storage能够处理的并发I/O数
- session_busy_ratio，DB连接处于active的时间比例
- avg_parallelism，执行查询的backend进程的平均数

参考文档：
https://www.cybertec-postgresql.com/en/tuning-max_connections-in-postgresql/ Posted on 2020-04-17 by Laurenz Albe
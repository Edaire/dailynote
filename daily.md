## 2021.03.05

### centos php docker

#### build nginx from centos7

Dockerfile

`FROM centos:centos7.6.1810`

`RUN yum -y install epel-release && yum -y install nginx && yum -y install net-tools vim wget pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop`

`EXPOSE 80/tcp 443/tcp`

`CMD ["nginx"]`

build image

 `sudo docker build --network=host -t harbor-registry.inner.youdao.com/ynote-test/centos7-nginx .`

push image

`sudo docker push harbor-registry.inner.youdao.com/ynote-test/centos7-nginx`

`docker container ls`

#### build php from nginx

`yum -y install epel-release yum-utils`

`yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm`

`yum install -y php73-php-fpm php73-php-cli php73-php-bcmath php73-php-gd php73-php-json php73-php-mbstring php73-php-mcrypt php73-php-mysqlnd php73-php-opcache php73-php-pdo php73-php-pecl-crypto php73-php-pecl-mcrypt php73-php-pecl-geoip php73-php-pecl-swoole php73-php-recode php73-php-snmp php73-php-soap php73-php-xmll`

`php73 -v #查看版本`

`systemctl enable php73-php-fpm #开启开机自启`

`systemctl restart php73-php-fpm #重启`

`systemctl start php73-php-fpm #启动`

`systemctl stop php73-php-fpm #关闭`

`systemctl status php73-php-fpm #检查状态`



`#查找php.ini位置：

find /etc/opt/remi/php73 -name php.ini`

`#The current PHP memory limit is below the recommended value of 512MB.`

`vi /etc/opt/remi/php73/php.ini`

`memory_limit = 512M`

`#如果你运行的是 nginx 而不是 apache，修改`

`vi /etc/opt/remi/php73/php-fpm.d/www.conf`

`user = apache`

`group = apache`

`#Replace the values with`

`user = nginx`

`group = nginx`

`#查找 php 和扩展的安装包：`

`rpm -qa | grep 'php'`

`#查看 php73-php-pecl-swoole4-4.4.15-1.el7.remi.x86_64 的安装路径：`

`rpm -ql php73-php-pecl-swoole4-4.4.15-1.el7.remi.x86_64`



#### nginx zj174

hosts

`103.72.47.155 discuzq-test.youdao.com`

![image-20210310111635077](.\image-20210310111635077.png)

![image-20210310111746794](.\image-20210310111746794.png)

![image-20210310112139001](.\image-20210310112139001.png)





### 消息队列

pulsar    在其存储层上使用了Apache BookKeeper

kafka

rabbitMQ  实现AMQP消息传递标准的开源传统的面向消息的中间件。它的功能包括排队，交换，路由和低延迟消息传递。

![image-20210319115340165](.\image-20210319115340165.png)

![image-20210329170921425](.\image-20210329170921425.png)

<div id="main"><h1>JEP 376: ZGC: Concurrent Thread-Stack Processing</h1><table class="head"><tbody><tr><td>Owner</td><td>Erik Österlund</td></tr><tr><td>Type</td><td>Feature</td></tr><tr><td>Scope</td><td>Implementation</td></tr><tr><td>Status</td><td>Closed / Delivered</td></tr><tr><td>Release</td><td>16</td></tr><tr><td>Component</td><td>hotspot / gc</td></tr><tr><td>Discussion</td><td>hotspot dash gc dash dev at openjdk dot java dot net</td></tr><tr><td>Effort</td><td>S</td></tr><tr><td>Duration</td><td>S</td></tr><tr><td>Reviewed by</td><td>Mikael Vidstedt, Per Liden, Stefan Karlsson</td></tr><tr><td>Endorsed by</td><td>Mikael Vidstedt</td></tr><tr><td>Created</td><td>2020/02/21 09:12</td></tr><tr><td>Updated</td><td>2021/03/07 19:41</td></tr><tr><td>Issue</td><td><a href="https://bugs.openjdk.java.net/browse/JDK-8239600">8239600</a></td></tr></tbody></table><div class="markdown"><h2 id="Summary">Summary</h2>
<p>Move ZGC thread-stack processing from safepoints to a concurrent phase.</p>
<h2 id="Goals">Goals</h2>
<ul>
<li>Remove thread-stack processing from ZGC safepoints.</li>
<li>Make stack processing lazy, cooperative, concurrent, and incremental.</li>
<li>Remove all other per-thread root processing from ZGC safepoints.</li>
<li>Provide a mechanism by which other HotSpot subsystems can lazily process stacks.</li>
</ul>
<h2 id="Non-Goals">Non-Goals</h2>
<ul>
<li>It is not a goal to implement concurrent per-thread processing of non-GC safepoint operations, such as class redefinition.</li>
</ul>
<h2 id="Success-Metrics">Success Metrics</h2>
<ul>
<li>The throughput cost of the improved latency should be insignificant.</li>
<li>Less than one millisecond should be spent inside ZGC safepoints on typical machines.</li>
</ul>
<h2 id="Motivation">Motivation</h2>
<p>The ZGC garbage collector (GC) aims to make GC pauses and scalability issues in HotSpot a thing of the past. We have, so far, moved all GC operations that scale with the size of the heap and the size of metaspace out of safepoint operations and into concurrent phases. Those include marking, relocation, reference processing, class unloading, and most root processing.</p>
<p>The only activities still done in GC safepoints are a subset of root processing and a time-bounded marking termination operation. The roots include Java thread stacks and various other thread roots. These roots are problematic, since they scale with the number of threads. With many threads on large machine, root processing becomes a problem.</p>
<p>In order to move beyond what we have today, and to meet the expectation that time spent inside of GC safepoints does not exceed one millisecond, even on large machines, we must move this per-thread processing, including stack scanning, out to a concurrent phase.</p>
<p>After this work, essentially nothing of significance will be done inside ZGC safepoint operations.</p>
<p>The infrastructure built as part of this project may eventually be used by other projects, such as Loom and JFR, to unify lazy stack processing.</p>
<h2 id="Description">Description</h2>
<p>We propose to address the stack-scanning problem with a <em>stack watermark barrier</em>. A GC safepoint will logically invalidate Java thread stacks by flipping a global variable. Each invalidated stack will be processed concurrently, keeping track of what remains to be processed. As each thread wakes up from the safepoint it will notice that its stack is invalid by comparing some epoch counters, so it will install a <em>stack watermark</em> to track the state of its stack scan. The stack watermark makes it possible to distinguish whether a given frame is above the watermark (assuming that stacks grow downward) and hence must not be used by a Java thread since it may contain stale object references.</p>
<p>In all operations that either pop a frame or walk below the last frame of the stack (e.g., stack walkers, returns, and exceptions), hooks will compare some stack-local address to the watermark. (This stack-local address may be a frame pointer, where available, or a stack pointer for compiled frames where the frame pointer is optimized away but frames have a reasonably constant size.) When above the watermark, a slow path will be taken to fix up one frame by updating the object references within it and moving the watermark upward. In order to make returns as fast as they are today, the stack watermark barrier will use a slightly modified safepoint poll. The new poll not only takes a slow path when safepoints (or indeed thread-local handshakes) are pending, but also when returning to a frame that has not yet been fixed up. This can be encoded for compiled methods with a single conditional branch.</p>
<p>An invariant of the stack watermark is that, given a callee which is the last frame of the stack, both the callee and the caller are processed. To ensure this, when the stack watermark state is installed when waking up from safepoints, both the caller and the callee are processed. The callee is armed so that returns from that callee will trigger further processing of the caller, moving the armed frame to the caller, and so on. Hence processing triggered by frame unwinding or walking always occurs two frames above the frame being unwound or walked. This simplifies the passing of arguments that have to be owned by the caller yet are used by the callee; both the caller and the callee frames (and hence the extra stack arguments) can be accessed freely.</p>
<p>Java threads will process the minimum number of frames needed to continue execution. Concurrent GC threads will take care of the remaining frames, ensuring that all thread stacks and other thread roots are eventually processed. Synchronization, utilizing the stack watermark barrier, will ensure that Java threads do not return into a frame while the GC is processing it.</p>
<h2 id="Alternatives">Alternatives</h2>
<p>When it comes to dealing with stack walkers, we considered the alternative solution of sprinkling load barriers across the VM where object references are loaded from the stack. We dismissed this because it fundamentally could not guarantee that root processing of internal pointers into objects are processed correctly. The base pointer of an internal pointer must always be processed after an internal pointer, and stack walkers would risk violating that invariant. Therefore we chose the approach of processing the whole frame, if not already processed, via stack walking.</p>
<h2 id="Testing">Testing</h2>
<p>The main code paths affected by this work are paths that other tests already stress to a great degree, so stress testing with the existing testing infrastructure should be sufficient.</p>
</div></div>

<div id="main"><h1>JEP 376: ZGC: 并行线程栈过程</h1><table class="head"><tbody><tr><td>Owner</td><td>Erik Österlund</td></tr><tr><td>Type</td><td>Feature</td></tr><tr><td>Scope</td><td>Implementation</td></tr><tr><td>Status</td><td>Closed / Delivered</td></tr><tr><td>Release</td><td>16</td></tr><tr><td>Component</td><td>hotspot / gc</td></tr><tr><td>Discussion</td><td>hotspot dash gc dash dev at openjdk dot java dot net</td></tr><tr><td>Effort</td><td>S</td></tr><tr><td>Duration</td><td>S</td></tr><tr><td>Reviewed by</td><td>Mikael Vidstedt, Per Liden, Stefan Karlsson</td></tr><tr><td>Endorsed by</td><td>Mikael Vidstedt</td></tr><tr><td>Created</td><td>2020/02/21 09:12</td></tr><tr><td>Updated</td><td>2021/03/07 19:41</td></tr><tr><td>Issue</td><td><a href="https://bugs.openjdk.java.net/browse/JDK-8239600">8239600</a></td></tr></tbody></table><div class="markdown"><h2 id="Summary">概述</h2>
<p>ZGC线程栈处理从安全点(safepoints)改为并发段（concurrent phase）</p>
<h2 id="Goals">目标</h2>
<ul>
<li>从ZGC安全点中移除线程栈处理</li>
<li>Make stack processing lazy, cooperative, concurrent, and incremental.</li>
<li>Remove all other per-thread root processing from ZGC safepoints.</li>
<li>Provide a mechanism by which other HotSpot subsystems can lazily process stacks.</li>
</ul>
<h2 id="Non-Goals">Non-Goals</h2>
<ul>
<li>It is not a goal to implement concurrent per-thread processing of non-GC safepoint operations, such as class redefinition.</li>
</ul>
<h2 id="Success-Metrics">Success Metrics</h2>
<ul>
<li>The throughput cost of the improved latency should be insignificant.</li>
<li>Less than one millisecond should be spent inside ZGC safepoints on typical machines.</li>
</ul>
<h2 id="Motivation">Motivation</h2>
<p>The ZGC garbage collector (GC) aims to make GC pauses and scalability issues in HotSpot a thing of the past. We have, so far, moved all GC operations that scale with the size of the heap and the size of metaspace out of safepoint operations and into concurrent phases. Those include marking, relocation, reference processing, class unloading, and most root processing.</p>
<p>The only activities still done in GC safepoints are a subset of root processing and a time-bounded marking termination operation. The roots include Java thread stacks and various other thread roots. These roots are problematic, since they scale with the number of threads. With many threads on large machine, root processing becomes a problem.</p>
<p>In order to move beyond what we have today, and to meet the expectation that time spent inside of GC safepoints does not exceed one millisecond, even on large machines, we must move this per-thread processing, including stack scanning, out to a concurrent phase.</p>
<p>After this work, essentially nothing of significance will be done inside ZGC safepoint operations.</p>
<p>The infrastructure built as part of this project may eventually be used by other projects, such as Loom and JFR, to unify lazy stack processing.</p>
<h2 id="Description">Description</h2>
<p>We propose to address the stack-scanning problem with a <em>stack watermark barrier</em>. A GC safepoint will logically invalidate Java thread stacks by flipping a global variable. Each invalidated stack will be processed concurrently, keeping track of what remains to be processed. As each thread wakes up from the safepoint it will notice that its stack is invalid by comparing some epoch counters, so it will install a <em>stack watermark</em> to track the state of its stack scan. The stack watermark makes it possible to distinguish whether a given frame is above the watermark (assuming that stacks grow downward) and hence must not be used by a Java thread since it may contain stale object references.</p>
<p>In all operations that either pop a frame or walk below the last frame of the stack (e.g., stack walkers, returns, and exceptions), hooks will compare some stack-local address to the watermark. (This stack-local address may be a frame pointer, where available, or a stack pointer for compiled frames where the frame pointer is optimized away but frames have a reasonably constant size.) When above the watermark, a slow path will be taken to fix up one frame by updating the object references within it and moving the watermark upward. In order to make returns as fast as they are today, the stack watermark barrier will use a slightly modified safepoint poll. The new poll not only takes a slow path when safepoints (or indeed thread-local handshakes) are pending, but also when returning to a frame that has not yet been fixed up. This can be encoded for compiled methods with a single conditional branch.</p>
<p>An invariant of the stack watermark is that, given a callee which is the last frame of the stack, both the callee and the caller are processed. To ensure this, when the stack watermark state is installed when waking up from safepoints, both the caller and the callee are processed. The callee is armed so that returns from that callee will trigger further processing of the caller, moving the armed frame to the caller, and so on. Hence processing triggered by frame unwinding or walking always occurs two frames above the frame being unwound or walked. This simplifies the passing of arguments that have to be owned by the caller yet are used by the callee; both the caller and the callee frames (and hence the extra stack arguments) can be accessed freely.</p>
<p>Java threads will process the minimum number of frames needed to continue execution. Concurrent GC threads will take care of the remaining frames, ensuring that all thread stacks and other thread roots are eventually processed. Synchronization, utilizing the stack watermark barrier, will ensure that Java threads do not return into a frame while the GC is processing it.</p>
<h2 id="Alternatives">Alternatives</h2>
<p>When it comes to dealing with stack walkers, we considered the alternative solution of sprinkling load barriers across the VM where object references are loaded from the stack. We dismissed this because it fundamentally could not guarantee that root processing of internal pointers into objects are processed correctly. The base pointer of an internal pointer must always be processed after an internal pointer, and stack walkers would risk violating that invariant. Therefore we chose the approach of processing the whole frame, if not already processed, via stack walking.</p>
<h2 id="Testing">Testing</h2>
<p>The main code paths affected by this work are paths that other tests already stress to a great degree, so stress testing with the existing testing infrastructure should be sufficient.</p>
</div></div>

InnoDB采用MVCC来支持高并发，1并且实现了四个标准的隔离级别。其默认级别是RR，并且通过间隙锁（next-key locking）策略防止幻读的出现。间隙锁使得InnoDB不仅仅锁定查询涉及的行，还会对索引中的间隙进行锁定，以防止幻影行的插入

### redis 6.0新特性

https://zhuanlan.zhihu.com/p/139079822

1.多线程IO

Redis 6引入多线程IO，但多线程部分只是用来处理网络数据的读写和协议解析，执行命令仍然是单线程。

2.重新设计了客户端缓存功能

实现了Client-side-caching（客户端缓存）功能。放弃了caching slot，而只使用key names。https://redis.io/topics/client-side-caching

3.RESP3协议

RESP（Redis Serialization Protocol）是 Redis 服务端与客户端之间通信的协议。Redis 5 使用的是 RESP2，而 Redis 6 开始在兼容 RESP2 的基础上，开始支持 RESP3。https://redis.io/topics/client-side-caching

4.支持SSL

连接支持SSL，更加安全。

5.ACL权限控制

\1. 支持对客户端的权限控制，实现对不同的key授予不同的操作权限。

\2. 有一个新的ACL日志命令，允许查看所有违反ACL的客户机、访问不应该访问的命令、访问不应该访问的密钥，或者验证尝试失败。这对于调试ACL问题非常有用。

6.提升了RDB日志加载速度

根据文件的实际组成（较大或较小的值），可以预期20/30%的改进。当有很多客户机连接时，信息也更快了，这是一个老问题，现在终于解决了。

7.发布官方的Redis集群代理模块 Redis Cluster proxy

在 Redis 集群中，客户端会非常分散，现在为此引入了一个集群代理，可以为客户端抽象 Redis 群集，使其像正在与单个实例进行对话一样。同时在简单且客户端仅使用简单命令和功能时执行多路复用。

8.提供了众多的新模块（modules）API

### Redis 4.0新特性

https://blog.huangz.me/diary/2016/redis-4-outline.html

1、模块系统

2、PSYNC 2.0某些情况下下slave-》master，其他slave不需要全量复制；slave重启可以不全量复制；

3、缓存取值的策略优化，新增LFU；

4、非阻塞DEL、FLUSHDB、FLUSHALL

5、交换数据库

6、混合RDB-AOF持久化格式（这一点最重要）

7、内存命令MEMORY

8、兼容NAT和Docker

### Redis 5.0 新特性

https://cloud.tencent.com/developer/article/1360819

1、新的流数据类型（Stream data type）https://redis.io/topics/streams-intro

2、新的 Redis 模块 API：定时器、集群和字典 API(Timers, Cluster and Dictionary APIs)

3、RDB 现在可存储 LFU 和 LRU 信息

4、新的有序集合(sorted set)命令：ZPOPMIN/MAX 和阻塞变体(blocking variants)

5、增强 HyperLogLog 的实现

### Redis高可用

https://zhuanlan.zhihu.com/p/142832200

### Redis watch

由于WATCH命令的作用只是当被监控的键值被修改后阻止之后一个事务的执行，而不能保证其他客户端不修改这一键值，所以在一般的情况下我们需要在EXEC执行失败后重新执行整个函数。

执行EXEC命令后会取消对所有键的监控，如果不想执行事务中的命令也可以使用UNWATCH命令来取消监控。

### Redis pub/sub

### Redis 数据淘汰策略

redis5.0为我们提供了八个不同的内存置换策略。很早之前提供了6种。

（1）volatile-lru：从已设置过期时间的数据集中挑选最近最少使用的数据淘汰。

（2）volatile-ttl：从已设置过期时间的数据集中挑选将要过期的数据淘汰。

（3）volatile-random：从已设置过期时间的数据集中任意选择数据淘汰。

（4）volatile-lfu：从已设置过期时间的数据集挑选使用频率最低的数据淘汰。

（5）allkeys-lru：从数据集中挑选最近最少使用的数据淘汰

（6）allkeys-lfu：从数据集中挑选使用频率最低的数据淘汰。

（7）allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰

（8） no-enviction（驱逐）：禁止驱逐数据，这也是默认策略。意思是当内存不足以容纳新入数据时，新写入操作就会报错，请求可以继续进行，线上任务也不能持续进行，采用no-enviction策略可以保证数据不被丢失。

这八种大体上可以分为4中，lru、lfu、random、ttl。
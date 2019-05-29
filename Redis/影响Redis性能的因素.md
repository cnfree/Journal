# [影响Redis性能的因素][1]

* Network bandwidth and latency usually have a direct impact on the performance. It is a good practice to use the ping program to quickly check the latency between the client and server hosts is normal before launching the benchmark. Regarding the bandwidth, it is generally useful to estimate the throughput in Gbit/s and compare it to the theoretical bandwidth of the network. 
  For instance a benchmark setting 4 KB strings in Redis at 100000 q/s, would actually consume 3.2 Gbit/s of bandwidth and probably fit within a 10 Gbit/s link, but not a 1 Gbit/s one. In many real world scenarios, Redis throughput is limited by the network well before being limited by the CPU. To consolidate several high-throughput Redis instances on a single server, it worth considering putting a 10 Gbit/s NIC or multiple 1 Gbit/s NICs with TCP/IP bonding.
  
  网络带宽和延迟通常直接影响性能，估算Redis性能的一个最佳实践就是在测试前ping一下客户端到服务端，检查网络延迟。关于带宽，可以估算一下Redis吞吐量，然后对比一下网络理论带宽。举个例子，4K的字符串，每秒10万次请求，需要消耗3.2Gb/s的带宽，可能需要10Gb/s的网络而不是1Gb/s。在现实场景，Redis吞吐量经常受限于网络带宽而不是CPU。要在一台服务器上整合多个高吞吐量Redis实例，需要考虑将10Gbit/s网卡或多个1Gbit/s网卡与TCP/IP绑定。

* CPU is another very important factor. Being single-threaded, Redis favors fast CPUs with large caches and not many cores. At this game, Intel CPUs are currently the winners. It is not uncommon to get only half the performance on an AMD Opteron CPU compared to similar Nehalem EP/Westmere EP/Sandy Bridge Intel CPUs with Redis. When client and server run on the same box, the CPU is the limiting factor with redis-benchmark.  

  CPU是另外一个非常重要的因素。由于单线程的设计，Redis喜欢大缓存，高速，核数少的CPU。Intel的CPU性能测试更好，AMD CPU仅有Intel CPU一半的得分。在单机测试中，CPU是限制性能的因素。
  
* Speed of RAM and memory bandwidth seem less critical for global performance especially for small objects. For large objects (>10 KB), it may become noticeable though. Usually, it is not really cost-effective to buy expensive fast memory modules to optimize Redis.

  内存的速度和带宽对小对象的性能影响不是很关键。但是对于大对象（>10KB），则有明显影响。通常，购买昂贵的快速内存模块来优化Redis并不是真正的经济有效。
  
* Redis runs slower on a VM compared to running without virtualization using the same hardware. If you have the chance to run Redis on a physical machine this is preferred. However this does not mean that Redis is slow in virtualized environments, the delivered performances are still very good and most of the serious performance issues you may incur in virtualized environments are due to over-provisioning, non-local disks with high latency, or old hypervisor software that have slow fork syscall implementation.

  Redis在一台虚拟机上慢于相同硬件无虚拟化的机器。如果有条件，Redis最好运行在物理机上。尽管如此，这不意味着Redis在虚拟环境里运行很慢，Redis性能仍然非常好。你在虚拟化环境中可能遇到的大多数严重性能问题都是由于过度配置、具有高延迟的非本地磁盘或具有缓慢fork syscall实现的旧管理程序软件造成的。
  
* When an ethernet network is used to access Redis, aggregating commands using pipelining is especially efficient when the size of the data is kept under the ethernet packet size (about 1500 bytes). Actually, processing 10 bytes, 100 bytes, or 1000 bytes queries almost result in the same throughput.

  当在局域网里访问Redis，当指令数据大小小于数据包大小（1500bytes）时，使用pipelining聚合命令非常有效。实际上，处理10bytes，100bytes，1000bytes的请求，吞吐量是一样的。

* With high-end configurations, the number of client connections is also an important factor. Being based on epoll/kqueue, the Redis event loop is quite scalable. Redis has already been benchmarked at more than 60000 connections, and was still able to sustain 50000 q/s in these conditions. As a rule of thumb, an instance with 30000 connections can only process half the throughput achievable with 100 connections.     

  使用高端配置时，客户端连接数也是一个非常重要的因素。基于epoll/kqueue模型，Redis事件循环伸缩性非常好。当Redis超过60000个连接时，仍然能够稳定在50000q/s的吞吐量。根据经验，一个实例，30000个连接时的可用吞吐量只有100个连接时的一半。

# [关于redis性能问题分析和优化][2]
  
  * 倘若当前实例延迟很明显，那么使用管道去降低延迟是非常有效的。
  * 避免操作大集合的慢命令：如果命令处理频率过低导致延迟时间增加，这可能是因为使用了高时间复杂度的命令操作导致，这意味着每个命令从集合中获取数据的时间增大。
  * 尽量使用O(1)的命令。 

[1]: https://redis.io/topics/benchmarks
[2]: https://www.cnblogs.com/chenpingzhao/p/6859041.html
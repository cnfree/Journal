* cluster-require-full-coverage=yes
  * 不允许集群部分失败
  
* cluster-require-full-coverage=no
  * 允许集群部分失败。
  
* cluster-node-timeout
  * Redis集群节点超时时限，默认15秒
  
* Redis-Cluster是无中心的架构，判断节点失败是通过仲裁的方式来进行（gossip和raft）
  * 大部分节点认为一个节点挂掉了，就会做fail判定。
  * 如果某个节点在执行比较重的操作（flushall, slaveof等等）（可能短时间redis客户端连接会阻塞(redis单线程)）或者由于网络原因，造成其他节点认为它挂掉了，会做fail判定。
  * Redis-Cluster提供了cluster-node-timeout这个参数（默认15秒），作为fail依据。
  
* 一对主从所在机器：不跨机房、要跨机架、可以在一个机柜。

* 不要用乱用cluster setslot node

* redis cluster 分布式一致性协议基于 Gossip 算法
  * 当 Redis-Cluster 出现主节点故障后，集群会经历故障检测、选举、故障倒换三大步骤，在此期间 Redis-Cluster 是不能提供服务的 

* 使用Redis Cluster的方针是首要保证能够快速创建一个可用的集群, 其次要严格限制可对集群做的变更操作, 还有尽可能用小集群

* 在使用gossip协议中, 如果多个节点声称不同的集群信息, 那对于某个节点来说究竟要相信谁呢? Redis Cluster规定了每个主节点的epoch都不可以相同。 而一个节点只会去相信拥有更大node epoch的节点声称的信息, 因为更大的epoch代表更新的集群信息。

  原则上:
  1. 如果epoch不变, 集群就不应该有变更(包括选举和迁移槽位)
  2. 每个节点的node epoch都是独一无二的
  3. 拥有越高epoch的节点, 集群信息越新

* Redis Cluster 现在这种单进程的做法导致大集群产生很大的ping包流量, 有一个几百个节点的集群光放在那里没有任何请求都有300MB的流量。  

* gossip协议包含多种消息，包括ping，pong，meet，fail，等等
  * meet: 某个节点发送meet给新加入的节点，让新节点加入集群中，然后新节点就会开始与其他节点进行通信
    * redis-trib.rb add-node，其实内部就是发送了一个gossip meet消息，给新加入的节点，通知那个节点去加入我们的集群。
  * ping: 每个节点都会频繁给其他节点发送ping，其中包含自己的状态还有自己维护的集群元数据，互相通过ping交换元数据，每个节点每秒都会频繁发送ping给其他的集群，ping，频繁的互相之间交换数据，互相进行元数据的更新。
  * pong: 返回ping和meet，包含自己的状态和其他信息，也可以用于信息广播和更新。
  * fail: 某个节点判断另一个节点fail之后，就发送fail给其他节点，通知其他节点，指定的节点宕机了。

* Asynchronous AOF fsync is taking too long
  * 本地IO较大使用造成的
  * 通过命令 tsar --io -n 2 | head -200 查看IO状态
  
* auto-aof-rewrite-percentage 100
  * 当前写入日志文件的大小超过上一次rewrite之后的文件大小的百分之100时就是2倍时触发Rewrite
  * rewrite之后aof文件会保存keys的最后的状态，清除掉之前冗余的，来缩小这个文件
  * aof文件的大小超过基准百分之多少后触发bgrewriteaof

* auto_aofrewrite_min_size
  * 当前aof文件大于多少字节后才触发。避免在aof较小的时候无谓行为。默认大小为64mb
    
* appendfsync 同步频率
  * always    
    * 每个 Redis 命令都要同步写入硬盘。这样会严重降低 Redis 的性能
  * everysec  
    * 每秒执行一次同步，显式地将多个写命令同步到硬盘
  * no        
    * 让操作系统来决定应该何时进行同步

# 参考文章
 * [redis-cluster核心原理分析：gossip通信、jedis smart定位、主备切换][1]
 * [Redis Cluster:Too many Cluster redirections异常][2]
 
 
[1]: https://blog.csdn.net/A_BlackMoon/article/details/85345645
[2]: https://carlosfu.iteye.com/blog/2251034 
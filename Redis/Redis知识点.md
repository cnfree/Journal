# Redis知识点
关键字： `数据结构` `应用` `哨兵` `Hash槽`

## Redis数据结构
 * 五种基础数据结构：String, List, Set, ZSet, Dict(Hash)
 * SDS(simple dynamic string)：简单动态字符串
    > Redis 没有直接使用c语言的字符串，SDS的字符串长度通过sds->len来控制，不受限于C语言字符串\0，可以存储二进制数据，并且将获取字符串长度的时间复杂度降到了O(1)
 * SkipList 跳跃表
    > 跳跃表是一种**有序**的数据结构，可以快速查找对应的元素，查找、删除和插入的平均效率为 O(logN) 。redis 用跳跃表来实现hash和zset。
 * 其他需要关注的底层数据结构：ADList, IntSet, ZipList, QuickList, Geometry

## Redis基础应用
 * 记录帖子的点赞数、评论数和点击数 (hash)。
 * 记录用户的帖子 ID 列表 (排序)，便于快速显示用户的帖子列表 (zset)。
 * 记录帖子的标题、摘要、作者和封面信息，用于列表页展示 (hash)。
 * 记录帖子的点赞用户 ID 列表，评论 ID 列表，用于显示和去重计数 (zset)。
 * 缓存近期热帖内容 (帖子内容空间占用比较大)，减少数据库压力 (hash)。
 * 记录帖子的相关文章 ID，根据内容推荐相关帖子 (list)。
 * 如果帖子 ID 是整数自增的，可以使用 Redis 来分配帖子 ID(计数器)。
 * 收藏集和帖子之间的关系 (zset)。
 * 记录热榜帖子 ID 列表，总热榜和分类热榜 (zset)。
 * 缓存用户行为历史，进行恶意行为过滤 (zset,hash)。
 
## Redis高级应用
 * [UV统计][3]：Redis HyperLogLog：提供不精确的去重计数方案，指令：pfadd 和 pfcount 
 * [缓存穿透，布隆过滤器][2]：bf.add，bf.madd 添加元素，bf.exists，bf.mexists 查询元素是否存在
 * [周活跃用户][1]：bitmap，setbit，bitop 
  
 
## 主从模式配置 
 * Redis Slaveof 命令可以将当前服务器转变为指定服务器的从属服务器
    ```
    redis 127.0.0.1:6379> SLAVEOF host port
    ```
 * 对一个从属服务器执行命令 SLAVEOF NO ONE 将使得这个从属服务器关闭复制功能，并从从属服务器转变回主服务器，原来同步所得的数据集不会被丢弃。   
    ```
    redis 127.0.0.1:6379> SLAVEOF NO ONE  
    ```
## 哨兵模式
 * 每个哨兵节点本质上就是一个redis实例，但仅仅监控redis数据节点
 * 哨兵模式可以解决主节点的故障转移，但不能解决从节点的故障转移
 * 本质上就是选取主节点执行 SLAVEOF NO ONE，其余从节点 执行 SLAVEOF master port
 
## 集群模式 
 * 一致性哈希加减节点会造成部分数据无法命中，需要手动处理，少量节点负载均衡，数据映射不好
 * Hash槽是虚拟槽位，节点变化时，数据映射的槽位不会发生变化
 * 计算key的插槽值：key的有效部分使用CRC16算法计算出哈希值，再将哈希值对16384取余，得到插槽值
 * 什如果key中包含了{符号，且在{符号后存在}符号，并且{和}之间至少有一个字符，则有效部分是指{和}之间的部分，否则整个key都是有效部分
   ```
   key = {hello}_tatao 的有效部分是hello，详情可以关注HashTag
   key = hello_taotao 的有效部分是hello_taotao
   ```
   
## 分布式锁

## 消息队列

## 发布者订阅者

## 延时队列

## 迭代扫描Scan

## 大key扫描

## 线程IO模型

## 持久化

## 管道(Pipeline)

## 事务
 > Redis对事务的支持目前还比较简单。Redis只能保证一个client发起的事务中的命令可以连续的执行，而中间不会插入其他client的命令。
 
 * MULTI
   * 用于标记事务块的开始。Redis会将后续的命令逐个放入队列中，然后才能使用EXEC命令原子化地执行这个命令序列。
   * 这个命令的返回值是一个简单的字符串，总是OK。
 
 * EXEC
   * 在一个事务中执行所有先前放入队列的命令，然后恢复正常的连接状态。
   *  当使用WATCH命令时，只有当受监控的键没有被修改时，EXEC命令才会执行事务中的命令，这种方式利用了检查再设置（CAS）的机制。
   *  这个命令的返回值是一个数组，其中的每个元素分别是原子化事务中的每个命令的返回值。 当使用WATCH命令时，如果事务执行中止，那么EXEC命令就会返回一个Null。
          
 * DISCARD
   * 清除所有先前在一个事务中放入队列的命令，然后恢复正常的连接状态。如果使用了WATCH命令，那么DISCARD命令就会将当前连接监控的所有键取消监控。
   * 这个命令的返回值是一个简单的字符串，总是OK。
 
 * WATCH   
   * Redis使用WATCH命令实现事务的 "检查再设置"（CAS）行为。
   * 作为WATCH命令的参数的键会受到Redis的监控，Redis能够检测到它们的变化。在执行EXEC命令之前，如果Redis检测到至少有一个键被修改了，那么整个事务便会中止运行，然后EXEC命令会返回一个Null值，提醒用户事务运行失败。
   * 这个命令的返回值是一个简单的字符串，总是OK。
   * 在每个代表数据库的 redis.h/redisDb 结构类型中， 都保存了一个 watched_keys 字典， 字典的键是这个数据库被监视的键， 而字典的值则是一个链表， 链表中保存了所有监视这个键的客户端。
   * 触发过程：
     * 在任何对数据库键空间（key space）进行修改的命令成功执行之后 （比如 FLUSHDB 、 SET 、 DEL 、 LPUSH 、 SADD 、 ZREM ，诸如此类）， multi.c/touchWatchedKey 函数都会被调用 —— 它检查数据库的 watched_keys 字典， 看是否有客户端在监视已经被命令修改的键， 如果有的话， 程序将所有监视这个/这些被修改键的客户端的 REDIS_DIRTY_CAS 选项打开。
     * 当客户端发送 EXEC 命令、触发事务执行时， 服务器会对客户端的状态进行检查：
     * 如果客户端的 REDIS_DIRTY_CAS 选项已经被打开，那么说明被客户端监视的键至少有一个已经被修改了，事务的安全性已经被破坏。服务器会放弃执行这个事务，直接向客户端返回空回复，表示事务执行失败。
     * 如果 REDIS_DIRTY_CAS 选项没有被打开，那么说明所有监视键都安全，服务器正式执行事务
 
 * UNWATCH
   * 清除所有先前为一个事务监控的键。
   * 如果你调用了EXEC或DISCARD命令，那么就不需要手动调用UNWATCH命令
   * 这个命令的返回值是一个简单的字符串，总是OK
   
 * WATCH 可以被多次调用。所有的WATCH 调用都会在 EXEC 调用之前起作用。WATCH可以接收任意多的key 。
   
 * 当 EXEC 被调用后, 所有的keys都将UNWATCH，不管这个事务会不会终止。同样，当一个客户端链接关闭后, 一切都将UNWATCH。  
   
 * 在Redis中事务总是具有原子性(Atomicity),一致性(Consistency)和隔离性(Isolation)。
 
 * 当Redis运行在某种特定的持久化模式下,事务也具有耐久性(Durability)。
 
 * Redis事务不支持回滚操作，就算有命令失败，队列中的其他命令也会被执行。
    
 # Lua 脚本
   * Redis支持lua脚本原子执行
   
## 参考资料
 * [Redis深度历险--核心原理与应用实践][4]
 * [Redis源码解读][5]

 [1]: https://www.jianshu.com/p/62cf39db5c2f
 [2]: https://www.cnblogs.com/lingyejun/p/10087135.html
 [3]: https://www.jianshu.com/p/ce09046f0206
 [4]: https://blog.csdn.net/weixin_37766296/article/details/84783434
 [5]: http://czrzchao.com/redisSourceSds#
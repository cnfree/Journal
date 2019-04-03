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
## 参考资料
 * [Redis深度历险--核心原理与应用实践][4]
 * [Redis源码解读][5]

 [1]: https://www.jianshu.com/p/62cf39db5c2f
 [2]: https://www.cnblogs.com/lingyejun/p/10087135.html
 [3]: https://www.jianshu.com/p/ce09046f0206
 [4]: https://blog.csdn.net/weixin_37766296/article/details/84783434
 [5]: http://czrzchao.com/redisSourceSds#
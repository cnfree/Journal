# Spring缓存

## Java缓存体系
 ![cache]

 Java Caching定义了5个核心接口，分别是CachingProvider, CacheManager, Cache, Entry 和 Expiry。
 * **CachingProvider**定义了创建、配置、获取、管理和控制多个CacheManager。一个应用可以在运行期访问多个CachingProvider。
 * **CacheManager**定义了创建、配置、获取、管理和控制多个唯一命名的Cache，这些Cache存在于CacheManager的上下文中。一个CacheManager仅被一个CachingProvider所拥有。
 * **Cache**是一个类似Map的数据结构并临时存储以Key为索引的值。一个Cache仅被一个CacheManager所拥有。
 * **Entry**是一个存储在Cache中的key-value对。
 * **Expiry** 每一个存储在Cache中的条目有一个定义的有效期。一旦超过这个时间，条目为过期的状态。一旦过期，条目将不可访问、更新和删除。缓存有效期可以通过ExpiryPolicy设置。

## Spring缓存抽象

  * Spring从3.1开始定义了org.springframework.cache.Cache和org.springframework.cache.CacheManager接口来统一不同的缓存技术，并支持使用JCache（JSR-107）注解简化我们开发
  * Cache接口为缓存的组件规范定义，包含缓存的各种操作集合
  * Cache接口下Spring提供了各种xxxCache的实现；如RedisCache，EhCacheCache , ConcurrentMapCache等；

  每次调用需要缓存功能的方法时，Spring会检查检查指定参数的指定的目标方法是否已经被调用过；如果有就直接从缓存中获取方法调用后的结果，如果没有就调用方法并缓存结果后返回给用户。下次调用直接从缓存中获取。

  使用Spring缓存抽象时我们需要关注以下两点；
  1. 确定方法需要被缓存以及他们的缓存策略
  2. 从缓存中读取之前缓存存储的数据

   ![cache1]

   ![cache2]

## LRU
 最近最少未使用
   * 每次访问就把这个元素放到队列头部，队列满了淘汰队列尾的元素，也就是淘汰最长时间没有被访问的。
   * 缺点: 某一时刻大量数据的到来容易把热点数据挤出缓存，而这些数据却是只访问了一次的，今后不会再访问了的或者访问频率极低的

 LRU 通过历史数据来预测未来是局限的，它会认为最后到来的数据是最可能被再次访问的，从而给与它最高的优先级。这样就**意味着淘汰真正热点数据**

 Mysql的缓存池，内部实现是一个LRU，但是其内部有个中间点,指向倒数3/8，一半是old区，另一半是young区，新数据插入是直接插入young区，这样就保护了真正的老数据不会被冲刷掉

## LFU
最近最少未使用
  * 每次访问就把这个元素放到队列头部，队列满了淘汰队列尾的元素，也就是淘汰最长时间没有被访问的。
  * 缺点: 某一时刻大量数据的到来容易把热点数据挤出缓存，而这些数据却是只访问了一次的，今后不会再访问了的或者访问频率极低的

传统LFU的实现通过外接一个HashMap统计频率，但是HashMap存在Hash冲突，这会导致频率统计的不准确。

## Caffeine
* **Caffeine是Spring 5默认支持的Cache**
* **Caffeine的API的操作功能和Guava是基本保持一致的**，并且Caffeine为了兼容之前是Guava的用户，做了一个Guava的Adapter
* Caffeine提出一种新的算法**W-TinyLFU**，它可以解决频率统计不准确以及访问频率衰减问题
* 在Caffeine中有两种缓存，一中无界,一种有界，只有有界时候才存在缓存淘汰策略，无界不存在淘汰也没有界限。
  * 有界缓存提供三个配置
    * expireAfterWrite：代表着写了之后多久过期。
    * expireAfterAccess: 代表着最后一次访问了之后多久过期。
    * expireAfter:在expireAfter中需要自己实现Expiry接口，这个接口支持create,update,以及access了之后多久过期。注意这个API和前面两个API是互斥的。这里和前面两个API不同的是，需要你告诉缓存框架，他应该在具体的某个时间过期，也就是通过前面的重写create,update,以及access的方法，获取具体的过期时间。

参考文档：
 * [为什么Caffeine比Guava好？]
 * [深入解密比Guava Cache更优秀的缓存-Caffein]
 * [guava和caffeine性能测试]


[cache]:img/cache.png
[cache1]:img/cache1.png
[cache2]:img/cache2.png
[为什么Caffeine比Guava好？]: https://blog.csdn.net/qq_33330687/article/details/88857030
[深入解密比Guava Cache更优秀的缓存-Caffein]: https://my.oschina.net/u/4072299/blog/3025253
[guava和caffeine性能测试]: https://blog.csdn.net/hy245120020/article/details/78080686
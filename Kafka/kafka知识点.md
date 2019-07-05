# Kafka知识点
 * kafka中的broker 是干什么的
   * broker 是消息的代理，Producers往Brokers里面的指定Topic中写消息，Consumers从Brokers里面拉取指定Topic的消息，然后进行业务处理，broker在中间起到一个代理保存消息的中转站。
 
 * kafka中的 zookeeper 起到什么作用，可以不用zookeeper么
   * zookeeper 是一个分布式的协调组件，早期版本的kafka用zk做meta信息存储，consumer的消费状态，group的管理以及 offset的值。考虑到zk本身的一些因素以及整个架构较大概率存在单点问题，新版本中逐渐弱化了zookeeper的作用。新的consumer使用了kafka内部的group coordination协议，也减少了对zookeeper的依赖， 
     但是broker依然依赖于ZK，zookeeper 在kafka中还用来选举controller 和 检测broker是否存活等等。

 * kafka follower如何与leader同步数据
   * Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。完全同步复制要求All Alive Follower都复制完，这条消息才会被认为commit，这种复制方式极大的影响了吞吐率。而异步复制方式下，Follower异步的从Leader复制数据，数据只要被Leader写入log就被认为已经commit，这种情况下，如果leader挂掉，会丢失数据，kafka使用ISR的方式很好的均衡了确保数据不丢失以及吞吐率。Follower可以批量的从Leader复制数据，而且Leader充分利用磁盘顺序读以及send file(zero copy)机制，这样极大的提高复制性能，内部批量写磁盘，大幅减少了Follower与Leader的消息量差。
    
 * 什么情况下一个 broker 会从 isr中踢出去
   * leader会维护一个与其基本保持同步的Replica列表，该列表称为ISR(in-sync Replica)，每个Partition都会有一个ISR，而且是由leader动态维护 ，如果一个follower比一个leader落后太多，或者超过一定时间未发起数据复制请求，则leader将其重ISR中移除
 
 * kafka 为什么那么快
   * Cache Filesystem Cache PageCache缓存
   * 顺序写 由于现代的操作系统提供了预读和写技术，磁盘的顺序写大多数情况下比随机写内存还要快。
   * Zero-copy 零拷⻉技术减少拷贝次数
   * Batching of Messages 批量量处理。合并小的请求，然后以流的方式进行交互，直顶网络上限。
   * Pull 拉模式 使用拉模式进行消息的获取消费，与消费端处理能力相符。
 
 * kafka producer如何优化打入速度
   * 增加线程
   * 提高 batch.size
   * 增加更多 producer 实例
   * 增加 partition 数
   * 设置 acks=-1 时，如果延迟增大：可以增大 num.replica.fetchers（follower 同步数据的线程数）来调解；
   * 跨数据中心的传输：增加 socket 缓冲区设置以及 OS tcp 缓冲区设置。
   
 * kafka producer 打数据，ack  为 0， 1， -1 的时候代表啥， 设置 -1 的时候，什么情况下，leader 会认为一条消息 commit了
   * 1（默认）  数据发送到Kafka后，经过leader成功接收消息的的确认，就算是发送成功了。在这种情况下，如果leader宕机了，则会丢失数据。
   * 0 生产者将数据发送出去就不管了，不去等待任何返回。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
   * -1 producer需要等待ISR中的所有follower都确认接收到数据后才算一次发送完成，可靠性最高。当ISR中所有Replica都向Leader发送ACK时，leader才commit，这时候producer才能认为一个请求中的消息都commit了。
   
 * kafka unclean 配置代表啥，会对spark streaming 消费有什么影响
   * unclean.leader.election.enable 为true的话，意味着非ISR集合的broker 也可以参与选举，这样有可能就会丢数据，spark streaming在消费过程中拿到的 end offset 会突然变小，导致 spark streaming job挂掉。如果unclean.leader.election.enable参数设置为true，就有可能发生数据丢失和数据不一致的情况，Kafka的可靠性就会降低；而如果unclean.leader.election.enable参数设置为false，Kafka的可用性就会降低。
     
 * 如果leader crash时，ISR为空怎么办
   * kafka在Broker端提供了一个配置参数：unclean.leader.election,这个参数有两个值：
     * true（默认）：允许不同步副本成为leader，由于不同步副本的消息较为滞后，此时成为leader，可能会出现消息不一致的情况。
     * false：不允许不同步副本成为leader，此时如果发生ISR列表为空，会一直等待旧leader恢复，降低了可用性。
  
 * kafka的message格式是什么样的
   * 一个Kafka的Message由一个固定长度的header和一个变长的消息体body组成
   * header部分由一个字节的magic(文件格式)和四个字节的CRC32(用于判断body消息体是否正常)构成。
   * 当magic的值为1的时候，会在magic和crc32之间多一个字节的数据：attributes(保存一些相关属性，
   * 比如是否压缩、压缩格式等等);如果magic的值为0，那么不存在attributes属性 
   * body是由N个字节构成的一个消息体，包含了具体的key/value消息
  
 * kafka中consumer group 是什么概念
   * 同样是逻辑上的概念，是Kafka实现单播和广播两种消息模型的手段。同一个topic的数据，会广播给不同的group；同一个group中的worker，只有一个worker能拿到这个数据。换句话说，对于同一个topic，每个group都可以拿到同样的所有数据，但是数据进入group后只能被其中的一个worker消费。group内的worker可以使用多线程或多进程来实现，也可以将进程分散在多台机器上，worker的数量通常不超过partition的数量，且二者最好保持整数倍关系，因为Kafka在设计时假定了一个partition只能被一个worker消费（同一group内）。
 
 * Kafka的设计是什么样的呢
   * Kafka将消息以topic为单位进行归纳
   * 将向Kafka topic发布消息的程序成为producers.
   * 将预订topics并消费消息的程序成为consumer.
   * Kafka以集群的方式运行，可以由一个或多个服务组成，每个服务叫做一个broker.
   * producers通过网络将消息发送到Kafka集群，集群向消费者提供消息
 
 * 数据传输的事物定义有哪三种
   *  数据传输的事务定义通常有以下三种级别：
      * 最多一次: 消息不会被重复发送，最多被传输一次，但也有可能一次不传输
      * 最少一次: 消息不会被漏发送，最少被传输一次，但也有可能被重复传输.
      * 精确的一次（Exactly once）: 不会漏传输也不会重复传输,每个消息都传输被一次而且仅仅被传输一次，这是大家所期望的
 
 * Kafka判断一个节点是否还活着有那两个条件
   * 节点必须可以维护和ZooKeeper的连接，Zookeeper通过心跳机制检查每个节点的连接
   * 如果节点是个follower,他必须能及时的同步leader的写操作，延时不能太久                    
   
 * producer是否直接将数据发送到broker的leader(主节点)？
   * producer直接将数据发送到broker的leader(主节点)，不需要在多个节点进行分发，为了帮助producer做到这点，所有的Kafka节点都可以及时的告知:哪些节点是活动的，目标topic目标分区的leader在哪。这样producer就可以直接将消息发送到目的地了
   
 * Kafka consumer是否可以消费指定分区消息
   * Kafka consumer消费消息时，向broker发出"fetch"请求去消费特定分区的消息，consumer指定消息在日志中的偏移量（offset），就可以消费从这个位置开始的消息，customer拥有了offset的控制权，可以向后回滚去重新消费之前的消息，这是很有意义的
   
 * Kafka消息是采用Pull模式，还是Push模式
   * Kafka最初考虑的问题是，customer应该从brokes拉取消息还是brokers将消息推送到consumer，也就是pull还push。在这方面，Kafka遵循了一种大部分消息系统共同的传统的设计：producer将消息推送到broker，consumer从broker拉取消息
   * 一些消息系统比如Scribe和Apache Flume采用了push模式，将消息推送到下游的consumer。这样做有好处也有坏处：由broker决定消息推送的速率，对于不同消费速率的consumer就不太好处理了。消息系统都致力于让consumer以最大的速率最快速的消费消息，但不幸的是，push模式下，当broker推送的速率远大于consumer消费的速率时，consumer恐怕就要崩溃了。最终Kafka还是选取了传统的pull模式
   * Pull模式的另外一个好处是consumer可以自主决定是否批量的从broker拉取数据。Push模式必须在不知道下游consumer消费能力和消费策略的情况下决定是立即推送每条消息还是缓存之后批量推送。如果为了避免consumer崩溃而采用较低的推送速率，将可能导致一次只推送较少的消息而造成浪费。Pull模式下，consumer就可以根据自己的消费能力去决定这些策略
   * Pull有个缺点是，如果broker没有可供消费的消息，将导致consumer不断在循环中轮询，直到新消息到t达。为了避免这点，Kafka有个参数可以让consumer阻塞知道新消息到达(当然也可以阻塞知道消息的数量达到某个特定的量这样就可以批量发
 
 * Kafka存储在硬盘上的消息格式是什么
   * 消息由一个固定长度的头部和可变长度的字节数组组成。头部包含了一个版本号和CRC32校验码。
     * 消息长度: 4 bytes (value: 1+4+n)
     * 版本号: 1 byte
     * CRC校验码: 4 bytes
     * 具体的消息: n bytes
 
 * Kafka高效文件存储设计特点
   * Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
   * 通过索引信息可以快速定位message和确定response的最大大小。
   * 通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作。
   * 通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小。
 
 * Kafka 与传统消息系统之间有三个关键区别
   * Kafka 持久化日志，这些日志可以被重复读取和无限期保留
   * Kafka 是一个分布式系统：它以集群的方式运行，可以灵活伸缩，在内部通过复制数据提升容错能力和高可用性
   * Kafka 支持实时的流式处理      
 
 * Kafka创建Topic时如何将分区放置到不同的Broker中
   * 副本因子不能大于 Broker 的个数；
   * 第一个分区（编号为0）的第一个副本放置位置是随机从 brokerList 选择的；
   * 其他分区的第一个副本放置位置相对于第0个分区依次往后移。也就是如果我们有5个 Broker，5个分区，假设第一个分区放在第四个 Broker 上，那么第二个分区将会放在第五个 Broker 上；第三个分区将会放在第一个 Broker 上；第四个分区将会放在第二个 Broker 上，依次类推；
   * 剩余的副本相对于第一个副本放置位置其实是由 nextReplicaShift 决定的，而这个数也是随机产生的
   
 * Kafka新建的分区会在哪个目录下创建
   * 在启动 Kafka 集群之前，我们需要配置好 log.dirs 参数，其值是 Kafka 数据的存放目录，这个参数可以配置多个目录，目录之间使用逗号分隔，通常这些目录是分布在不同的磁盘上用于提高读写性能。
   * 当然我们也可以配置 log.dir 参数，含义一样。只需要设置其中一个即可。
   * 如果 log.dirs 参数只配置了一个目录，那么分配到各个 Broker 上的分区肯定只能在这个目录下创建文件夹用于存放数据。
   * 但是如果 log.dirs 参数配置了多个目录，那么 Kafka 会在哪个文件夹中创建分区目录呢？答案是：Kafka 会在含有分区目录最少的文件夹中创建新的分区目录，分区目录名为 Topic名+分区ID。注意，是分区文件夹总数最少的目录，而不是磁盘使用量最少的目录！也就是说，如果你给 log.dirs 参数新增了一个新的磁盘，新的分区目录肯定是先在这个新的磁盘上创建直到这个新的磁盘目录拥有的分区目录不是最少为止。
 
 * partition的数据如何保存到硬盘
   * topic中的多个partition以文件夹的形式保存到broker，每个分区序号从0递增，
   * 且消息有序
   * Partition文件下有多个segment（xxx.index，xxx.log）
   * segment 文件里的 大小和配置文件大小一致可以根据要求修改 默认为1g
   * 如果大小大于1g时，会滚动一个新的segment并且以上一个segment最后一条消息的偏移量命名
 
 * Kafka的消费者如何消费数据
   * 消费者每次消费数据的时候，消费者都会记录消费的物理偏移量（offset）的位置
   * 等到下次消费时，他会接着上次位置继续消费
 
 * 消费者负载均衡策略
   * 一个消费者组中的一个分片对应一个消费者成员，他能保证每个消费者成员都能访问，如果组中成员太多会有空闲的成员
   
 * 数据有序
   * 一个消费者组里它的内部是有序的
   * 消费者组与消费者组之间是无序的    
  
 * kafaka生产数据时数据的分组策略
   * 生产者决定数据产生到集群的哪个partition中
   * 每一条消息都是以（key，value）格式
   * Key是由生产者发送数据传入
   * 所以生产者（key）决定了数据产生到集群的哪个partition  
 
 * Kafka服务器能接收到的最大信息是多少?
   * Kafka服务器可以接收到的消息的最大大小是1000000字节。
   
 * 我们可以在没有Zookeeper的情况下使用Kafka吗?
   * 不可能越过Zookeeper，直接联系Kafka broker。一旦Zookeeper停止工作，它就不能服务客户端请求。    
   * Zookeeper主要用于在集群中不同节点之间进行通信
   * 在Kafka中，它被用于提交偏移量，因此如果节点在任何情况下都失败了，它都可以从之前提交的偏移量中获取
   * 除此之外，它还执行其他活动，如: leader检测、分布式同步、配置管理、识别新节点何时离开或连接、集群、节点实时状态等等。
 
 * Kafka的用户如何消费信息
   * 在Kafka中传递消息是通过使用sendfile API完成的。它支持将字节从套接口转移到磁盘，通过内核空间保存副本，并在内核用户之间调用内核。
   
 * 如何提高远程用户的吞吐量
   * 如果用户位于与broker不同的数据中心，则可能需要调优套接口缓冲区大小，以对长网络延迟进行摊销。
 
 * 在数据制作过程中，你如何能从Kafka得到准确的信息?
   * 在数据中，为了精确地获得Kafka的消息，你必须遵循两件事: 在数据消耗期间避免重复，在数据生产过程中避免重复。
   * 这里有两种方法，可以在数据生成时准确地获得一个语义:
     * 每个分区使用一个单独的写入器，每当你发现一个网络错误，检查该分区中的最后一条消息，以查看您的最后一次写入是否成功     
     * 在消息中包含一个主键(UUID或其他)，并在用户中进行反复制
 
 * 有可能在生产后发生消息偏移吗
   * 在大多数队列系统中，作为生产者的类无法做到这一点，它的作用是触发并忘记消息。broker将完成剩下的工作，比如使用id进行适当的元数据处理、偏移量等。
   * 作为消息的用户，你可以从Kafka broker中获得补偿。如果你注视SimpleConsumer类，你会注意到它会获取包括偏移量作为列表的MultiFetchResponse对象。此外，当你对Kafka消息进行迭代时，你会拥有包括偏移量和消息发送的MessageAndOffset对象。
         
 * Kafka高级消费者
   * 消费者消费哪个分区不由消费者决定，而是由高阶API决定，固定分区个数时如果消费者个数大于分区个数，那么将会有消费者空转，造成资源的浪费。
   
 * 自动提交offset 
   * 在高阶消费者中，Offset采用自动提交的方式。
   * 自动提交时，假设1s提交一次offset的更新
   * 高阶消费者存在一个弊端，即消费者消费到哪里由高阶消费者API进行提交，提交到ZooKeeper，消费者线程不参与offset更新的过程，这就会造成数据丢失（消费者读取完成，高级消费者API的offset已经提交，但是还没有处理完成Spark Streaming挂掉，此时offset已经更新，无法再消费之前丢失的数据），还有可能造成数据重复读取（消费者读取完成，高级消费者API的offset还没有提交，读取数据已经处理完成后Spark Streaming挂掉，此时offset还没有更新，重启后会再次消费之前处理完成的数据）。
   
 * Kafka高可靠性存储
   * 在Kafka文件存储中，同一个topic下有多个不同的partition，每个partiton为一个目录，partition的名称规则为：topic名称+有序序号，以segment为单位又将partition细分。 
   * segment文件由两部分组成，分别为“.index”文件和“.log”文件，分别表示为segment索引文件和数据文件（引入索引文件的目的就是便于利用二分查找快速定位message位置）。这两个文件的命令规则为：partition全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值，数值大小为64位，20位数字字符长度，没有数字用0填充，如下：
        
        ```
        000000000000000000000.index
        00000000000000000000.log
        00000000000000170410.index
        00000000000000170410.log
        00000000000000239430.index
        00000000000000239430.log
        ```   
 
 * kafka在高并发的情况下,如何避免消息丢失和消息重复  
   * 消息丢失解决方案:
     * 首先对kafka进行限速， 其次启用重试机制，重试间隔时间设置长一些，最后Kafka设置acks=all，即需要相应的所有处于ISR的分区都确认收到该消息后，才算发送成功
   * 消息重复解决方案:
     * 消息可以使用唯一id标识        
     * 生产者（ack=all 代表至少成功发送一次) 
     * 消费者 （offset手动提交，业务逻辑成功处理后，提交offset） 
     * 落表（主键或者唯一索引的方式，避免重复数据）  
     * 业务逻辑处理（选择唯一主键存储到Redis或者mongdb中，先查询是否存在，若存在则不处理；若不存在，先插入Redis或Mongdb,再进行业务逻辑处理）
 
 * kafka怎么保证数据消费一次且仅消费一次
   * 幂等producer：保证发送单个分区的消息只会发送一次，不会出现重复消息     
   * 事务(transaction)：保证原子性地写入到多个分区，即写入到多个分区的消息要么全部成功，要么全部回滚 
   * 流处理EOS：流处理本质上可看成是“读取-处理-写入”的管道。此EOS保证整个过程的操作是原子性。注意，这只适用于Kafka Streams
 
 * kafka保证数据一致性和可靠性
   * 数据一致性保证
     * 一致性定义：若某条消息对client可见，那么即使Leader挂了，在新Leader上数据依然可以被读到
     * HW-HighWaterMark: client可以从Leader读到的最大msg offset，即对外可见的最大offset， HW=max(replica.offset)
     * 对于Leader新收到的msg，client不能立刻消费，Leader会等待该消息被所有ISR中的replica同步后，更新HW，此时该消息才能被client消费，这样就保证了如果Leader fail，该消息仍然可以从新选举的Leader中获取。
     * 对于来自内部Broker的读取请求，没有HW的限制。同时，Follower也会维护一份自己的HW，Folloer.HW = min(Leader.HW, Follower.offset)
   * 数据可靠性保证
     * Producer向Leader发送数据时，可以通过acks参数设置数据可靠性的级别
     * 0: 不论写入是否成功，server不需要给Producer发送Response，如果发生异常，server会终止连接，触发Producer更新meta数据；
     * 1: Leader写入成功后即发送Response，此种情况如果Leader fail，会丢失数据
     * -1: 等待所有ISR接收到消息后再给Producer发送Response，这是最强保证
 
 * kafka到spark streaming怎么保证数据完整性，怎么保证数据不重复消费？
   * 保证数据不丢失（at-least）
     * spark RDD内部机制可以保证数据at-least语义。
     * Receiver方式
       * 开启WAL（预写日志），将从kafka中接受到的数据写入到日志文件中，所有数据从失败中可恢复。
     * Direct方式
       * 依靠checkpoint机制来保证。
   
   * 保证数据不重复（exactly-once）
     * 要保证数据不重复，即Exactly once语义。
     * 幂等操作：重复执行不会产生问题，不需要做额外的工作即可保证数据不重复。
     * 业务代码添加事务操作 
     * 就是说针对每个partition的数据，产生一个uniqueId，只有这个partition的所有数据被完全消费，则算成功，否则算失效，要回滚。下次重复执行这个uniqueId时，如果已经被执行成功，则skip掉。
     
 * kafka的消费者高阶和低阶API有什么区别
   * kafka 提供了两套 consumer API：The high-level Consumer API和 The SimpleConsumer API     
   * 其中 high-level consumer API 提供了一个从 kafka 消费数据的高层抽象，而 SimpleConsumer API 则需要开发人员更多地关注细节。
   * The high-level consumer API
     * high-level consumer API 提供了 consumer group 的语义，一个消息只能被 group 内的一个 consumer 所消费，且 consumer 消费消息时不关注 offset，最后一个 offset 由 zookeeper 保存。
     * 使用 high-level consumer API 可以是多线程的应用，应当注意：
       * 如果消费线程大于 patition 数量，则有些线程将收不到消息
       * 如果 patition 数量大于线程数，则有些线程多收到多个 patition 的消息
       * 如果一个线程消费多个 patition，则无法保证你收到的消息的顺序，而一个 patition 内的消息是有序的
   * The SimpleConsumer API
     * 如果你想要对 patition 有更多的控制权，那就应该使用 SimpleConsumer API，比如：
     * 多次读取一个消息
     * 只消费一个 patition 中的部分消息
     * 使用事务来保证一个消息仅被消费一次但是使用此 API 时，partition、offset、broker、leader 等对你不再透明，需要自己去管理。你需要做大量的额外工作：
     * 必须在应用程序中跟踪 offset，从而确定下一条应该消费哪条消息
     * 应用程序需要通过程序获知每个 Partition 的 leader 是谁
     * 要处理 leader 的变更
 
 * 如何保证从Kafka获取数据不丢失?
   * 生产者数据的不丢失
     * kafka的ack机制：在kafka发送数据的时候，每次发送消息都会有一个确认反馈机制，确保消息正常的能够被收到。  
   * 消费者数据的不丢失
     * 通过offset commit 来保证数据的不丢失，kafka自己记录了每次消费的offset数值，下次继续消费的时候，接着上次的offset进行消费即可。  
 
 * 如果想消费已经被消费过的数据
   * consumer是底层采用的是一个阻塞队列，只要一有producer生产数据，那consumer就会将数据消费。当然这里会产生一个很严重的问题，如果你重启一消费者程序，那你连一条数据都抓不到，但是log文件中明明可以看到所有数据都好好的存在。换句话说，一旦你消费过这些数据，那你就无法再次用同一个groupid消费同一组数据了
   * 采用不同的group。
   * 通过一些配置，就可以将线上产生的数据同步到镜像中去，然后再由特定的集群区处理大批量的数据。    
 
 * kafka partition和consumer数目关系   
   * 如果consumer比partition多，是浪费，因为kafka的设计是在一个partition上是不允许并发的，所以consumer数不要大于partition数 。
   * 如果consumer比partition少，一个consumer会对应于多个partitions，这里主要合理分配consumer数和partition数，否则会导致partition里面的数据被取的不均匀 。最好partiton数目是consumer数目的整数倍，所以partition数目很重要，比如取24，就很容易设定consumer数目 。
   * 如果consumer从多个partition读到数据，不保证数据间的顺序性，kafka只保证在一个partition上数据是有序的，但多个partition，根据你读的顺序会有不同
   * 增减consumer，broker，partition会导致rebalance，所以rebalance后consumer对应的partition会发生变化
 
 * kafka topic 副本问题
   * Kafka尽量将所有的Partition均匀分配到整个集群上。一个典型的部署方式是一个Topic的Partition数量大于Broker的数量。
 
 * 如何分配副本:
   * Producer在发布消息到某个Partition时，先通过ZooKeeper找到该Partition的Leader，然后无论该Topic的Replication Factor为多少（也即该Partition有多少个Replica），Producer只将该消息发送到该Partition的Leader。Leader会将该消息写入其本地Log。每个Follower都从Leader pull数据。这种方式上，Follower存储的数据顺序与Leader保持一致。
   * Kafka分配Replica的算法如下：
     * 将所有Broker（假设共n个Broker）和待分配的Partition排序
     * 将第i个Partition分配到第（imod n）个Broker上
     * 将第i个Partition的第j个Replica分配到第（(i + j) mode n）个Broker上

 * zookeeper如何管理kafka
   * Producer端使用zookeeper用来"发现"broker列表,以及和Topic下每个partition leader建立socket连接并发送消息.
   * Broker端使用zookeeper用来注册broker信息,以及监测partition leader存活性.
   * Consumer端使用zookeeper用来注册consumer信息,其中包括consumer消费的partition列表等,同时也用来发现broker列表,并和partition leader建立socket连接,并获取消息.
   
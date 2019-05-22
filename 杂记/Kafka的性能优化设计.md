# Kafka的性能优化设计

## OS PageCache，避免GC
 * Kafka、Elasticsearch等系统虽然基于JVM运行，但都是重度依赖OS Cache来管理大量的数据。
 * 磁盘文件在写入之前会进入os cache，也就是操作系统管理的内存空间。
 * 消费数据是，也是优先从os cache(内存缓冲)里读取数据。
 * 不依赖JVM来缓冲大量数据，避免JVM垃圾回收。
 * 对于重度依赖OS Cache的中间件，不要为JVM分配过大内存，够用就行，为OS Cache保留足够的内存空间。
 * 如果Kafka重启，所有的In-Process Cache都会失效，而OS管理的PageCache依然可以继续使用。
 * PageCache在内核态，大小不限(新版本内核可以限制)，只要有空闲内存，就会尽量将其用作cache，在free命令中可以看到cache的实际用量。

## 客户端网络通信Batch和缓冲池，无GC操作
 * 消息会先写入一个内存缓冲，然后多条消息组成一个Batch，一次网络通信把Batch发送出去。避免一条消息一次网络请求，从而提升吞吐量。
 * 不断的清理发送成功的Batch会造成JVM GC，因此Kafka引入了缓冲池。
 * 每个Batch底层对应一块内存空间，不交给JVM垃圾回收，而是放入一个缓冲池里。Batch发送出去后，内存空间还回缓冲池。
 * 如果缓冲池满了，阻塞写入操作，等待内存块释放。

 ![Kafka客户端][Kafka客户端]

## CopyOnWriteMap
 * 不同于读写锁，**读写锁是读写互斥**，而CopyOnWrite是空间换时间，复制一份数据进行写操作，而不会影响读操作，**解决了高并发下的读写互斥问题**，适用于读多写少的操作。
 * 类似的有Mysql InnoDB MVCC 的**快照读**，根据时间点开辟一个内存空间用来读取数据，大大提升Mysql的查询性能。
 * Java的CopyOnWriteArrayList也是基于CopyOnWrite思想，利用高并发往往是读多写少的特性，对读操作不加锁，对写操作，先复制一份新的集合，在新的集合上面修改，然后将新集合赋值给旧的引用，并通过volatile 保证其可见性，当然写操作的锁是必不可少的了。

## Sequence I/O
 * 磁盘驱动器的吞吐量跟寻道延迟是相背离的
 * Kafka官方测试数据(Raid-5，7200rpm): 
   ```
   Sequence I/O: 600MB/s  
   Random I/O: 100KB/s  
   ```
 * Kafka在磁盘上只做Sequence I/O

## 内存zero-copy(零拷贝)
 * 常规网络传输操作步骤
   ```
    1. OS 从硬盘把数据读到内核区的PageCache。
    2. 用户进程把数据从内核区Copy到用户区。
    3. 然后用户进程再把数据写入到Socket，数据流入内核区的Socket Buffer上。
    4. OS 再把数据从Buffer中Copy到网卡的Buffer上，这样完成一次发送。
   ```
    ![传统网络传输方式][传统网络传输方式]
    
 * 整个过程共经历两次Context Switch，四次System Call。同一份数据在内核Buffer与用户Buffer之间重复拷贝，效率低下。  
 * 其中2、3两步没有必要，完全可以直接在内核区完成数据拷贝。
 * 采用操作系统高级函数**sendfile**进行传输，可以避免2、3两步的拷贝
    ![采用sendfile方式传输][采用sendfile方式传输]
 * IBM 零拷贝技术文章: [Efficient data transfer through zero copy][1] 
 * FileChannel有一个**transferTo**方法，底层实现了操作系统的sendfile函数
   ```
     public void transferTo(long position, long count, WritableByteChannel target);
     
     #include <sys/socket.h>
     ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
   ``` 
 * 性能对比: Traditional approach vs. zero copy  

    | File size |Normal file transfer (ms)|transferTo (ms) |
    |:---------:|:-----------------------:|:---------------:|
    | 7MB	    |156	                  |45              |
    |  21MB	    |337	                  |128             | 
    |  63MB	    |843	                  |387             | 
    |  98MB	    |1320	                  |617             |
    |  200MB	|2124	                  |1150            |
    |  350MB	|3631	                  |1762            |
    |  700MB	|13498	                  |4422            | 
    |  1GB	    |18399	                  |8537            |



[Kafka客户端]:img/Kafka客户端.png
[传统网络传输方式]:img/traditional_transfer.jpg
[采用sendfile方式传输]:img/sendfile.jpg
[1]: https://developer.ibm.com/articles/j-zerocopy/
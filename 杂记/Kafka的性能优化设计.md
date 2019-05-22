# Kafka的性能优化设计

## OS PageCache
 * Kafka、Elasticsearch等系统虽然基于JVM运行，但都是重度依赖OS Cache来管理大量的数据。
 * 磁盘文件在写入之前会进入os cache，也就是操作系统管理的内存空间。
 * 消费数据是，也是优先从os cache(内存缓冲)里读取数据。
 * 不依赖JVM来缓冲大量数据，避免JVM垃圾回收。
 * 对于重度依赖OS Cache的中间件，不要为JVM分配过大内存，够用就行，为OS Cache保留足够的内存空间。
 * 如果Kafka重启，所有的In-Process Cache都会失效，而OS管理的PageCache依然可以继续使用。
 * PageCache在内核态，大小不限(新版本内核可以限制)，只要有空闲内存，就会尽量将其用作cache，在free命令中可以看到cache的实际用量。

## 客户端网络通信Batch和缓冲池，无GC操作
 * 消息会先写入一个内存缓冲，然后多条消息组成一个Batch，一次网络通信把Batch发送出去。
 * 不断的清理发送成功的Batch会造成JVM GC，因此Kafka引入了缓冲池。
 * 每个Batch底层对应一块内存空间，不交给JVM垃圾回收，而是放入一个缓冲池里。
 * 如果缓冲池满了，阻塞写入操作，等待内存块释放。

 ![Kafka客户端][Kafka客户端]

## CopyOnWrite无锁Map

## Sequence I/O


[Kafka客户端]:img/Kafka客户端.png
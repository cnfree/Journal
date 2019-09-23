# RingBuffer

## 优点
 * Ring Buffer 比链表要快，因为它是数组，而且有一个容易预测的访问模式。这很不错，对 CPU 高速缓存友好 （CPU-cache-friendly）－数据可以在硬件层面预加载到高速缓存，因此 CPU 不需要经常回到主内存 RAM 里去寻找 Ring Buffer 的下一条数据。
   * 因为只要一个元素被加载到缓存行，其他相邻的几个元素也会被加载进同一个缓存行
 * 可以为数组预先分配内存，使得数组对象一直存在（除非程序终止）。这就意味着不需要花大量的时间用于垃圾回收
   * 此外，不像链表那样，需要为每一个添加到其上面的对象创造节点对象—对应的，当删除节点时，需要执行相应的内存清理操作。
 * RingBuffer和普通的FIFO队列相比还有一个重要的区别就是，RingBuffer避免了头尾节点的竞争，多个事件发布者/事件处理者之间不必竞争同一个节点，只需要安全申请序列值各自存取事件就好了

## 伪共享和缓存行填充

#### 缓存行
 ![cacheline]
 * 数据在缓存中不是以独立的项来存储的，如不是一个单独的变量，也不是一个单独的指针。缓存是由缓存行组成的，通常是64字节，并且它有效地引用主内存中的一块地址。
 * 一个Java的long类型是8字节，因此在一个缓存行中可以存8个long类型的变量。
 * 如果你访问一个long数组，当数组中的一个值被加载到缓存中，它会额外加载另外7个。因此你能非常快地遍历这个数组。
 * 如果数据结构中的项在内存中不是彼此相邻的（比如链表），将得不到免费缓存加载所带来的优势。并且在这些数据结构中的每一个项都可能会出现缓存未命中。

#### 伪共享
  * 缓存系统中是以缓存行（cache line）为单位存储的。缓存行是2的整数幂个连续字节，一般为32-256个字节。最常见的缓存行大小是64个字节。当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。
  * 比如head和tail在一个缓存行，并被两个不同的线程分别来处理，每次线程对缓存行进行写操作时，每个内核都要把另一个内核上的缓存块无效掉并重新读取里面的数据。实际上两个线程之间的写冲突了，尽管它们写入的是不同的变量。

#### 缓存行填充
  * 针对伪共享，通过缓存行填充，保证无关的东西不会同时存在于一个缓存行
      ```java
      public long p1, p2, p3, p4, p5, p6, p7; // cache line padding
      private volatile long cursor = INITIAL_CURSOR_VALUE;
      public long p8, p9, p10, p11, p12, p13, p14; // cache line padding
      ```
  * 没有伪共享，就没有和其它任何变量的意外冲突，没有不必要的缓存未命中。

# 相关资料
 * [伪共享和缓存行填充]
 * [disruptor-3.3.2源码解析(2)-队列]
 * [剖析Disruptor:为什么会这么快？（一）Ringbuffer的特别之处]
 * [Disruptor和ArrayBlockingQueue、LinkedBlockingQueue的比较]
 * [循环缓冲区（RingBuffer）]


[cacheline]:img/cacheline.webp
[伪共享和缓存行填充]: https://www.jianshu.com/p/85a8df061dc8
[disruptor-3.3.2源码解析(2)-队列]:https://www.iteye.com/blog/brokendreams-2255707
[剖析Disruptor:为什么会这么快？（一）Ringbuffer的特别之处]: http://www.ifeve.com/dissecting-disruptor-whats-so-special/
[Disruptor和ArrayBlockingQueue、LinkedBlockingQueue的比较]: https://www.iteye.com/blog/lvlianghui-1836155
[循环缓冲区（RingBuffer）]: https://www.jianshu.com/p/8499d9603544?from=timeline&isappinstalled=0
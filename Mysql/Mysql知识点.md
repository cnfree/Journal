# Mysql知识点
关键字: `索引` `优化` `锁` `数据结构` `主从复制` `数据一致性` `分库` `分表` `代理中间件`  

## Hash索引和B-Tree索引
 * 单行查询Hash索引更快，如果业务需求全部都是单行访问，可以使用hash索引
    ```
    select * from user where name = 'cnfree'
    ```
 * 对于需要排序的SQL需求，比如分组group by，排序order by，比较等，Hash索引时间复杂度会退化为O(N)
 * ~~InnoDB~~并不支持Hash索引

## 磁盘预读
 * 磁盘读写并不是按需读取，而是按页预读，一次会读一页的数据，每次加载更多的数据，如果未来要读取的数据就在这一页中，可以避免未来的磁盘IO，提高效率
 * 通常，一页数据是4K。
 
## 局部性原理
 * 内存读写块，磁盘读写慢，而且慢很多
 * 软件设计要尽量遵循“数据读取集中”与“使用到一个数据，大概率会使用其附近的数据”，这样磁盘预读能充分提高磁盘IO

## B+树
 * B+树结构
 
   ![B+树][B+TreeImage]
   
 * 相对于B树，叶子之间，增加了链表，获取所有节点，不再需要中序遍历；
 * 叶子节点存储实际记录行，记录行相对比较紧密的存储，适合大数据量磁盘存储；非叶子节点存储记录的PK，用于查询加速，适合内存存储；
 * 非叶子节点，不存储实际记录，而只存储记录的KEY的话，那么在相同内存的情况下，B+树能够存储更多索引；  
 * 很适合磁盘存储，能够充分利用局部性原理，磁盘预读；
 
 
## 索引常见坑
 * InnoDB的行锁是实现在索引上的，而不是锁在物理行记录上。如果访问没有命中索引，也无法使用行锁，将要退化为表锁。 
    ```
    update t_user set age=10 where uid=1;
    命中索引，行锁。
    
    update t_user set age=10 where uid != 1;
    未命中索引，表锁。
    
    update t_user set age=10 where name='shenjian';
    无索引，表锁。
    ```
## 并发控制
 并发的任务对同一个临界资源进行操作，如果不采取措施，可能导致不一致，故必须进行并发控制（Concurrency Control）。
 * 锁（Locking）
 * 数据多版本（Multi Versioning）    
 
## 锁
 * 共享锁（Share Locks，记为S锁），读取数据时加S锁
 * 排他锁（eXclusive Locks，记为X锁），修改数据时加X锁
 * 共享锁之间不互斥，读读可以并行
 * 排他锁与任何锁互斥，写读，写写不可以并行，类似Java的读写锁
 * 对应到数据库，可以理解为，写事务没有提交，读相关数据的select也会被阻塞。
 
## 数据多版本
 数据多版本是一种能够进一步提高并发的方法，它的核心原理是:
 1. 写任务发生时，将数据克隆一份，以版本号区分；
 2. 写任务操作新克隆的数据，直至提交；
 3. 并发读任务可以继续读取旧版本的数据，不至于阻塞；  
 
 * 普通锁，本质是串行执行
 * 读写锁，可以实现读读并发
 * 数据多版本，可以实现读写并发
 
 
#### InnoDB是基于多版本并发控制的存储引擎 
 InnoDB是高并发互联网场景最为推荐的存储引擎，根本原因，就是其多版本并发控制（Multi Version Concurrency Control, MVCC）。行锁，并发，事务回滚等多种特性都和MVCC相关。
 
## 快照读
 回滚段里的数据，其实是历史数据的快照（snapshot），这些数据是不会被修改，select可以肆无忌惮的并发读取他们。 
 * 除非显示加锁，普通的select语句都是快照读
   ```
   select * from t where id>2;
   ```
 * 非快照读
   ```
   select * from t where id>2 lock in share mode;
   
   select * from t where id>2 for update;
   ```  
## 小结
 
 1. 常见并发控制保证数据一致性的方法有锁，数据多版本；
 2. 普通锁串行，读写锁读读并行，数据多版本读写并行；
 3. redo日志保证已提交事务的ACID特性，设计思路是，通过顺序写替代随机写，提高并发；
 4. undo日志用来回滚未提交的事务，它存储在回滚段里；
 5. InnoDB是基于MVCC的存储引擎，它利用了存储在回滚段里的undo日志，即数据的旧版本，提高并发；
 6. InnoDB之所以并发高，快照读不加锁； 
 7. InnoDB所有普通select都是快照读；
 
## InnoDB的索引  
 InnoDB的索引有两类索引，**聚集索引**(Clustered Index)与**普通索引**(Secondary Index)。  
 
 InnoDB的每一个表都会有聚集索引:
 1. 如果表定义了PK，则PK就是聚集索引；
 2. 如果表没有定义PK，则第一个非空unique列是聚集索引；
 3. 否则，InnoDB会创建一个隐藏的row-id作为聚集索引；
 4. 聚集索引，叶子节点存储行记录(row)；
 5. 普通索引，叶子节点存储了PK的值；
   
## 聚集索引和普通索引
 InnoDB的普通索引，实际上会扫描两遍:第一遍，由普通索引找到PK；第二遍，由PK找到行记录；

 ![B+树][Index]
 1. 第一幅图，id PK的聚集索引，叶子存储了所有的行记录；
 2. 第二幅图，name上的普通索引，叶子存储了PK的值；
 
    ```
    select * from t where name=’shenjian’;
  
    (1)会先在name普通索引上查询到PK=1；
  
    (2)再在聚集索引上查询到(1,shenjian, m, A)的行记录；
    ```
 
## 隔离级别
 Mysql默认的事务隔离级别为**可重复读**(Repeated Read, RR)。
 
## 记录锁(Record Locks) 
 记录锁，它封锁索引记录，以阻止其他事务插入，更新，删除
 ```
  --记录锁，在id=1的索引记录上加锁，以阻止其他事务插入，更新，删除id=1的这一行
  select * from t where id=1 for update; 
  
  --快照读，无锁
  select * from t where id=1; 
 ```
 
## 间隙锁(Gap Locks) 
 间隙锁，它封锁索引记录中的间隔，或者第一条索引记录之前的范围，又或者最后一条索引记录之后的范围。

 ```
    -- 会封锁区间，以阻止其他事务id=10的记录插入
    select * from t 
        where id between 8 and 15 
        for update;
 ```
 
  * 间隙锁的主要目的，就是为了防止其他事务在间隔中插入数据，以导致“不可重复读”。
       
  * 如果把事务的隔离级别降级为读提交(Read Committed, RC)，间隙锁则会自动失效。   
 
### 幻读(Phantom Read)

 * 为什么要阻止id=10的记录插入？
 
    如果能够插入成功，头一个事务执行相同的SQL语句，会发现结果集多出了一条记录，即幻影数据。
    
## 临键锁(Next-Key Locks)
 * 临键锁，是记录锁与间隙锁的组合，它的封锁范围，既包含索引记录，又包含索引区间。
 * 临键锁会封锁索引记录本身，以及索引记录之前的区间。
 * 临键锁的主要目的，也是为了避免幻读。如果把事务的隔离级别降级为RC，临键锁则也会失效。

     ```
      表中有四条记录:
        1, shenjian, m, A
        3, zhangsan, m, A
        5, lisi,     m, A
        9, wangwu,   f, B
       
      PK上潜在的临键锁为:
        (-infinity, 1]
        (1, 3]
        (3, 5]
        (5, 9]
        (9, +infinity]
     ```
 
## 插入意向锁 (Insert Intention Locks)
 * 插入意向锁，是间隙锁(Gap Locks)的一种（所以，也是实施在索引上的），它是专门针对insert操作的。
 * 多个事务，在同一个索引，同一个范围区间插入记录时，如果插入的位置不冲突，不会阻塞彼此。
    
## 自增锁（Auto-inc Locks） 
  
 * 自增锁是一种特殊的表级别锁（table-level lock），专门针对事务插入AUTO_INCREMENT类型的列。
 * InnoDB提供了innodb_autoinc_lock_mode配置，可以调节与改变该锁的模式与行为。
  
## innodb_autoinc_lock_mode
  
 * innodb_autoinc_lock_mode = 0 (“traditional” lock mode) 表级锁
  
    ```
    优点:极其安全
    
    缺点:对于这种模式，写入性能最差。任何一种insert-like语句，都会产生一个table-level
    ```
       
 * innodb_autoinc_lock_mode = 1 (“consecutive” lock mode)  默认锁模式
    
    ```
    优点:非常安全，性能与innodb_autoinc_lock_mode =0 相比要好很多。
    
    对于Simple inserts，则使用的是一种轻量级锁，只要获取了相应的auto  increment就释放锁，并不会等到语句结束。
    
    发生bulk inserts的时候还是会产生表级别的自增锁，防止在bulk insert的时候，被其他的insert语句抢走auto  increment值。
    ```  
  
 * innodb_autoinc_lock_mode = 2 (“interleaved” lock mode)
    
    ```
    优点:性能非常好，提高并发，SBR不安全
    
    缺点:一条bulk insert，得到的自增id可能不连续。SBR模式下:会导致复制出错，不一致
    ```
  
## AUTO_INCREMENT计数器初始化 
 
 * 计数器仅存在于内存中，而不存储在磁盘上。
 * 服务器重新启动后初始化自动递增计数器，InnoDB将在首次插入行到包含AUTO_INCREMENT列的表时执行以下语句的等效语句。
    ```
    SELECT MAX(ai_col) FROM table_name FOR UPDATE; 
    ``` 
    
## 死锁

 * 并发插入相同记录，可能死锁(某一个回滚)
 * 并发插入，可能出现间隙锁死锁(难排查)
 * show engine innodb status; 可以查看InnoDB的锁情况，也可以调试死锁
 
## 可重复读级别下的锁
 
 * 普通Select快照读，无锁
 * 带锁的Select/Update/Delete，where条件只有一行为记录锁，范围查询间隙锁/临键锁 
 * Insert 插入意向锁

## [Binlog][1]（推荐Mixed，默认Statement）
  Mysql binlog日志有三种格式，分别为Statement,MiXED,以及ROW
  
  * Statement:每一条会修改数据的sql都会记录在binlog中。
  
  * Row:不记录sql语句上下文相关信息，仅保存哪条记录被修改。
  
  * Mixed level: 是以上两种level的混合使用
  
    一般的语句修改使用statment格式保存binlog，如一些函数，statement无法完成主从复制的操作，则采用row格式保存binlog,MySQL会根据执行的每一条具体的sql语句来区分对待记录的日志形式，也就是在Statement和Row之间选择 一种.新版本的MySQL中队row level模式也被做了优化，并不是所有的修改都会以row level来记录，像遇到表结构变更的时候就会以statement模式来记录。至于update或者delete等修改数据的语句，还是会记录所有行的 变更。 

## 执行计划
  ```
  explain select * from table where table.id = 1 
  
  table | type | possible_keys | key | key_len | ref | rows | Extra
  ```  
  
  * table: 显示这一行的数据是关于哪张表的
  
  * type: 这是重要的列，显示连接使用了何种类型。从最好到最差的连接类型为const、eq_reg、ref、range、indexhe和ALL
  
    说明: 不同连接类型的解释（按照效率高低的顺序排序）
  
    * system: 表只有一行:system表。这是const连接类型的特殊情况。
  
    * const: 表中的一个记录的最大值能够匹配这个查询（索引可以是主键或惟一索引）。因为只有一行，这个值实际就是常数，因为MYSQL先读这个值然后把它当做常数来对待。
  
    * eq_ref: 在连接中，MYSQL在查询时，从前面的表中，对每一个记录的联合都从表中读取一个记录，它在查询使用了索引为主键或惟一键的全部时使用。
  
    * ref: 这个连接类型只有在查询使用了不是惟一或主键的键或者是这些类型的部分（比如，利用最左边前缀）时发生。对于之前的表的每一个行联合，全部记录都将从表中读出。这个类型严重依赖于根据索引匹配的记录多少—越少越好。
  
    * range: 这个连接类型使用索引返回一个范围中的行，比如使用>或<查找东西时发生的情况。
  
    * index: 这个连接类型对前面的表中的每一个记录联合进行完全扫描（比ALL更好，因为索引一般小于表数据）。
  
    * ALL: 这个连接类型对于前面的每一个记录联合进行完全扫描，**这一般比较糟糕，应该尽量避免**。
  
  * possible_keys: 显示可能应用在这张表中的索引。如果为空，没有可能的索引。可以为相关的域从WHERE语句中选择一个合适的语句
  
  * key: 实际使用的索引。如果为NULL，则没有使用索引。很少的情况下，MYSQL会选择优化不足的索引。这种情况下，可以在SELECT语句中使用**USE INDEX**（indexname）来强制使用一个索引或者用**IGNORE INDEX**（indexname）来强制MYSQL忽略索引
  
  * key_len: 使用的索引的长度。在不损失精确性的情况下，长度越短越好
  
  * ref: 显示索引的哪一列被使用了，如果可能的话，是一个常数
  
  * rows: MYSQL认为必须检查的用来返回请求数据的行数
  
  * Extra: 关于MYSQL如何解析查询的额外信息。将在表4.3中讨论，但这里可以看到的坏的例子是Using temporary和Using filesort，意思MYSQL根本不能使用索引，结果是检索会很慢
  
    说明: extra列返回的描述的意义
  
    * Distinct: 一旦mysql找到了与行相联合匹配的行，就不再搜索了。
  
    * Not exists: mysql优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行，就不再搜索了。
  
    * Range checked for each Record（index map:#）: 没有找到理想的索引，因此对从前面表中来的每一个行组合，mysql检查使用哪个索引，并用它来从表中返回行。这是使用索引的最慢的连接之一。
  
    * Using filesort: **查询就需要优化了**。mysql需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来排序全部行。
  
    * Using index: 列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候。
  
    * Using temporary: **查询需要优化了**。这里，mysql需要创建一个临时表来存储结果，这通常发生在对不同的列集进行ORDER BY上，而不是GROUP BY上。
  
    * Where used: 使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。如果不想返回表中的全部行，并且连接类型ALL或index，这就会发生，或者是查询有问题。
  

 
## 主从不一致的解决方案

 * 允许短时间不一致的业务场景忽略这种情况
 * 选择性读主库
    1. 写主库
    2. 将哪个库，哪个表，哪个主键三个信息拼装一个key设置到cache里，这条记录的超时时间，设置为“主从同步时延”  
    3. 读库之前先根据key查询cache
    4. cache里有这个key，说明1s内刚发生过写请求，数据库主从同步可能还没有完成，此时就应该去主库查询
    5. cache里没有这个key，说明最近没有发生过写请求，此时就可以去从库查询。以此，保证读到的一定不是不一致的脏数据。
    6. 方案5中key可能过期，考虑订阅binlog消息，同步完成后删除key。
    
## 横向分库和拆分库扩容方案
 
 * 每个主库都带一个从库
 * 每次扩容考虑2的n次方
 * 扩容时，只需要将从库变成主库，4台主库，变成8台主库，库1的数据一定扩容后一定落在库1和库5上，而库5是库1的从库，包含库1的全部数据，因为不需要做数据迁移，只需要做好库1和库5的同步即可。
 
## 扩容步骤   
 
 假设原来分了2个库，d0和d1，都放在服务器s0上，s0同时有备机s1。
 
 数据同步:
 
 1. 确保s0 -> s1同步顺利，没有明显延迟
 2. s0暂时关闭读写权限
 3. 确认s1已经完全同步s0更新
 
 拆分库的d0和d1:
 
 1. s1开放读写权限
 2. d1的dns由s0切换到s1。
 3. s0开放读写权限
 
 横向扩容:
 
 1. 由4取模变成由8取模
 2. s1开放读写权限
 3. s0开放读写权限
 
 
 
     
 [B+TreeImage]:img/B+Tree.png
 [Index]:img/index.png
 [1]: https://www.cnblogs.com/itcomputer/articles/5005602.html
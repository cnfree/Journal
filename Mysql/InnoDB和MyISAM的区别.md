# InnoDB和MyISAM的区别
关键字: `InnoDB` `MyISAM`

## count(*)
* MyISAM存储总行数，InnoDB需要按行扫描。
* 但如果包含where条件，则都按索引进行扫描。
* **必须建立好索引**，否则性能消耗极大。

## 全文索引
* MyISAM支持全文索引
* InnoDB 5.6版本之后开始支持全文索引，5.7版本之后通过使用ngram插件开始支持中文。之前仅支持英文，因为是通过空格作为分词的分隔符，对于中文来说是不合适的。
* **都不建议全文索引**，小请求量会占用大量数据库资源，推荐使用Elasticsearch。

## 事务
* MyISAM不支持事务，InnoDB支持事务。
* 事务非常消耗性能，并且MyISAM在文件尾部**顺序增加记录**速度极快。如果**单线程写**，InnoDB的写性能**远远低于**MyISAM。
* InnoDB是通过**事务日志**redo log和undo log来实现事务的commit和rollback，写日志会消耗大量性能。

## 事务隔离级别
* InnoDB有七种锁来实现不同的隔离级别。
* InnoDB支持行锁。
* InnoDB的行锁是实现在**索引**上的，如果未命中索引，将退化为表锁或者GAP锁。
* MyISAM支持表锁，可以在某种程度上**模拟事务**的隔离级别，但是性能极差，在**高并发**的情况下吞吐量远低于InnoDB。

## 外键
* MyISAM不支持外键
* InnoDB支持外键
* 并发量大的情况下**不应该**使用外键，应由应用程序来保证完整性。

## 性能
* 开启binlog的情况下，Mysql的写性能下降60%
* **并发量很小**的情况下，MyISAM的写速度比InnoDB快40%
* MyISAM在文件尾部顺序增加记录速度极快，在**并发量很小**的情况下，性能极快。

## 缓存
* MyISAM将磁盘频繁读取的索引数据加载至内存中操作
* MyISAM设计了一个在存放在内存中的索引缓冲池Key Cache。Key Cache只缓存索引数据，通过LRU算法将读取频繁的索引加载到Key Cache中来。
* MyISAM通过系统变量 key_buffer_size 来控制Key Cache的大小，这个变量关乎到缓存的性能。
* InnoDB引擎中，同样设置了缓存池buffer pool，与MyISAM有所区别的是，buffer pool不仅仅缓存了索引数据，同时还缓存了表数据。
* MySQL官方文档建议，除了用于系统运行的内存外，剩余的内存建议尽可能大的设置buffer pool。这样一来有着大容量的buffer pool，在实际应用上的表现更像与一个in-memory database，相比于对磁盘的读写速度，读写性能简直就是巨大的提升。

## 读库
* 如果数据库数据量很大，采用MyISAM
* 如果数据库数据量一般，但内存容量很大，可以采用InnoDB+大缓存。
* **读库的sql是顺序执行的，主库的sql是并发执行的**，并且读库可以不需要开启binlog和事务日志，因此读库的写性能要远远优于写库。
* **写库默认InnoDB，读库默认MyISAM。**


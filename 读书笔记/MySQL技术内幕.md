# MySQL技术内幕

## 第一章
  * **数据库** 是物理操作系统文件，比如 frm, MYD, MYI, ibd 结尾的文件。
  * **数据库实例** 是一个进程，由后台线程及一个共享内存组成。
  * 数据库实例用来操作数据库文件。
  * 集群环境下，可能一个数据库被多个数据库实例使用。
  * MySQL数据库按 /etc/my.cnf --> /etc/mysql/my.cnf --> /usr/local/mysql/etc/my.cnf --> ~/.my.cnf 顺序读取配置文件。
  * 几个配置文件包含相同的参数，以读取到的最后一个配置文件的参数为准。
  * **数据库是文件的集合，数据库实例是程序**。
  * **存储引擎是基于表的**，而不是数据库。
  * InnoDB特点是**行锁设计**，支持**外键**，**默认读取操作不会产生锁**。MySQL默认存储引擎。
  * InnoDB表单独存放在一个独立的ibd文件中。
  * InnoDB通过使用**多版本并发控制（MVCC）**来获得高并发性。默认为**可重复读**。通过**临键锁来避免幻读**。 
  * 如果没有显式指定主键，会每一行生成一个6字节的ROWID，以此作为主键。
  * MyISAM **不支持事务**，**表锁**设计，支持**全文索引**。**面向OLAP应用**。
  * MyISAM 的缓冲池只**缓存索引文件，而不缓存数据文件**。
  * MyISAM 由MYD和MYI文件组成，MYD存储数据文件，MYI存储索引文件。
  * Memory 引擎默认采用Hash索引，而不是B+树索引。
  * Memory 只支持表锁，并发性较差，**不支持TEXT和BLOB类型**。VARCHAR类型按照CHAR存储。
  * 中间结果集产生的**临时表采用Memory引擎**，但如果包含TEXT和BLOB类型，会转换为MyISAM而损失性能。
  * Archive 引擎**只支持INSERT和SELECT**操作。会对数据进行压缩，压缩比1:10，**非常适合存储归档数据**。
  * ETL操作，MyISAM性能好，OLTP环境，InnoDB性能更好。
  * 不同存储引擎特性比较
  ![不同引擎特性比较][dbengine]
  
  
## 第二章
  * InnoDB缓存池包含：索引页，数据页，undo页，插入缓冲，自适应哈希索引，锁信息，数据字典信息。  
  * InnoDB内存数据对象
  ![InnoDB内存数据对象][dbobject]
  * **SHOW ENGINE INNODB STATUS** 查看InnoDB引擎状态
  * **Buffer pool hit rate** 表示缓冲池的命中率，通常该值不应该小于95%
  * 重做日志缓冲，默认8M，每秒会将重做日志缓冲刷新到日志文件
  * 每个事务提交时会将重做日志缓冲刷新到重做日志文件
  * 事务数据库采用Write Ahead Log策略，即事务提交时，先写重做日志，再修改页
  * 当数据库宕机时，只需对Checkpoint后的重做日志进行恢复
  * 当缓冲池不够时，会强制执行Checkpoint，将脏页刷会磁盘
  * LSN(Log Sequence Number) 标记版本，重做日志有LSN，Checkpoint 也有LSN
  * 当缓冲池中脏页超过75%时，会强制执行Checkpoint，将脏页刷会磁盘
  * innodb_io_capacity 用来表示IO 吞吐量，默认200，当使用SSD时，需要调大这个值
  * 一般聚集索引是顺序的，不需要随机读取，但UUID这类主键，是随机的。
  * 如果是随机插入或者更新，会将插入操作放入Insert Buffer，多个Insert 之后合并操作
  * 自适应Hash索引，通过缓冲池的B+树构建，而不是对整个数据构建，当连续相同的查询条件时，会自动为某些热点也创建哈希索引
  * 当开始事务时，UPDATE操作会创建UNDO日志(undo log)
 
## 第三章
  * 参数文件
  * 日志文件：错误日志、binlog、慢查询日志、查询日志、重做日志等
  * socket文件：当用Unix域套接字方式进行连接时需要的文件
  * pid文件：MySQL实例进程ID文件
  * MySQL表结构文件：存放MySQL表结构定义文件
  * 存储引擎文件 
  * SHOW VARIABLES 查看数据库中所有参数
  * Global 全局参数，Session 当前会话参数
  * autocommit只能在当前会话中修改
  * set global|session 设置动态变量
  * MySQL常见的日志有
    * 错误日志(error log)
    * 二进制日志(binlog)
    * 慢查询日志(slow query log)
    * 查询日志(log)
  * SHOW VARIABLES LIKE 'log_error' 定位错误日志 
  * 慢查询日志默认10秒，long_query_time来设置
  * 默认情况下不启动慢查询日志，需要手工设置 log_slow_queries 为 ON
  * mysqldumpslow -s al -n 10 david.log 导出执行时间最长的10条SQL语句
  * 查询日志默认为 机器名.log
  * 二进制日志(binary log)不包含SELECT 和 SHOW类操作
  * 若修改操作未发生任何变化，该操作也会被记录到binlog
  * 可以通过对binlog进行 point-in-name 进行恢复
  * binlog还可以用来复制从库
  * 通过配置log-bin参数来启动二进制文件
  * 所有未提交的二进制日志会记录到缓存，当事务提交时，直接将缓存中的二进制日志记录到二进制日志文件
  * frm文件存放表和视图结构的定义
  * 重做日志文件(redo log file)
  * **二进制仅在事务提交前提交，不论事务多大，只写磁盘一次**
  * 在事务进行的过程中，不断的有重做日志条目被写入重做日志文件
  * innodb_flush_log_at_trx_commit **设为1，每当有事务提交时，必须确保事务都已写入重做日志文件**
  * 二进制文件支持**STATEMENT（逻辑SQL）、ROW(表的行更改，建议采用该模式)、MIX**三种格式
  
  
  [dbengine]: img/dbengine.png 
  [dbobject]: img/dbobject.png
  
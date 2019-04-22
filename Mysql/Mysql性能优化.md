# MySQL知识点

## 文件结构
 * MyISAM: 
    * .MYD  数据文件 
    * .MYI  索引文件 
    * .frm  保存了每个表的元数据，包括表结构的定义等
 * InnoDB:
    * .idb  表空间，包含了数据和索引
    * .frm  保存了每个表的元数据，包括表结构的定义等
    * innodb_file_per_table 每张表单独的表空间，[点击查看优缺点][1]。默认情况下在MySQL 5.6.6及更高版本中启用。
    ```
        show variables like 'innodb_file_per_table'; 
    ```

## MySQL优化框架
 * SQL语句优化
 * 索引优化
 * 数据库结构优化
 * InnoDB表优化
 * MyISAM表优化
 * Memory表优化
 * 理解查询执行计划
 * 缓冲和缓存
 * 锁优化
 * MySQL服务器优化
 * 性能评估
 
## MySQL优化层次
 * MySQL级别: 表优化、查询优化、MySQL服务器配置优化
 * OS级别
 * 硬件级别

## 数据库层面优化
 * 是否正确设定了表结构的相关属性，尤其是每个字段的字段类型是否为最佳。同时，是否为特定类型的工作组织使用了合适的表及表字段也将影响系统性能。
 * 是否为高效查询创建了合适的索引
 * 是否为每张表选用了合适的引擎，并有效的利用了存储引擎本身的优势和特性
 * 是否基于存储引擎为表选用了合适的行格式(row format)。
 * 是否使用了合适的锁策略
 * 是否为InnoDB的缓冲池，MyISAM的键缓存以及MySQL的查询缓存设定了合适大小的内存空间，以便能够存储频繁访问的数据且又不会引起页面换出。
 
## 操作系统和硬件级别的优化点
 * 选择性能更强劲的CPU
 * 合适大小的物理内存，降低甚至避免磁盘IO，现代程序设计都会基于局部性原理使用到缓存技术
 * 网络设备和网络体系，避免丢包，按需进行网络设置，高效处理大量连接和小查询
 * 基于操作系统选择合适的文件格式
 * 高效的线程管理库
 
## InnoDB实践
 * 基于MySQL查询语句中最常用字段或字段组合创建主键，如果没有合适的主键也最好使用自增类型的某字段为主键
 * 根据需要考虑使用多表查询
 * 关闭autocommit
 * 使用事务
 * 停止使用Lock Tables语句，，如果需要在一系列的行上获取独占访问权限，使用 SELECT ... FOR UPDATE锁定需要更新的行
 * 启用inndb_file_per_table选项，将各表的数据和索引分别进行存储
 * 评估数据和访问模式能否从InnoDB的表压缩功能中受益(创建表时使用ROW_FORMAT=COMPRESSED)  
    
## 覆盖索引
 * 如果一个索引包含(或覆盖)所有需要查询的字段的值，称为‘覆盖索引’。即只需扫描索引而无须回表
 * 覆盖索引必须要存储索引列的值，而哈希索引、空间索引和全文索引不存储索引列的值，所以mysql只能用B-tree索引做覆盖索引
 * 当发起一个索引覆盖查询时，在explain的extra列可以看到using index的信息
 
## 索引
 加速查询，降低写入速度，需要考虑删除、更新操作性能的影响
 
 * 聚集索引
 * 非聚集索引 
 * 主索引
 * 辅助索引
 * 稠密索引
 * 稀疏索引
 * 多级索引
 * B+树(平衡树，全键值、键值范围、键值左前缀查找)
 * Hash索引(等值条件比较，=, IN(), <>，不能根据索引排序，不支持部分键值匹配)
 * 空间索引
 * 全文索引  
 
## MyISAM 配置项
 * key_buffer_size
 * concurrent_insert
 * delay_key_write

## InnoDB 配置项
 * innodb_buffer_pool_size
 * innodb_flush_log_at_trx_commit
 * innodb_log_file_size
 ```
 SHOW ENGINE INNODB STATUS;
 ```  

## Query Cache System Variables
 MySQL根据查询语句的hashcode查找缓存结果，区分大小写
 
 * query_alloc_block_size
 * query_cache_limit
 * query_cache_size
 * query_cache_type(OFF, ON 默认, DEMAND)  
 
## 参考文章
 * [MySQL大表优化方案总结][2]
 * [MySQL 性能调优的10个方法][3]
 * [深度认识 Sharding-JDBC][4]
 * [为什么不要问我DB极限QPS/TPS][5]
 * [mysql qps tps一般多大][6]
 * [mysql 性能容量评估][7]
 * [通过计算多种状态的百分比看MySQL的性能情况][8]
 * [缓存一致性和跨服务器查询的数据异构解决方案canal][9]
 * [重视读写分离的delay影响][10]
 * [全量数据同步与数据校验实践][11]
    
 [1]: https://blog.csdn.net/wanbin6470398/article/details/81633855
 [2]: https://www.jianshu.com/p/ef1ba6d2d02e
 [3]: https://www.jianshu.com/p/4c80457996e1
 [4]: https://my.oschina.net/editorial-story/blog/888650
 [5]: https://www.cnblogs.com/zhiqian-ali/p/6336521.html
 [6]: https://blog.csdn.net/hu2010shuai/article/details/55259516
 [7]: https://www.cnblogs.com/Aiapple/p/5697912.html
 [8]: https://www.cnblogs.com/zengkefu/p/5608096.html
 [9]: https://www.cnblogs.com/huangxincheng/p/7456397.html
 [10]: https://blog.csdn.net/tiwerbao/article/details/47735949
 [11]: https://blog.csdn.net/u010183402/article/details/70314705
# Maxwell 数据同步

* Maxwell 有两种模式, current 和 bootstrap 模式
    * current 从当前position 开始监控
    * bootstrap 同步历史信息
    
* DML语句会触发Maxwell的postion改变，DDL不会，如果多次更新DDL，而没有DML，每次重启Maxwell，会重复收到DDL语句

* 同一个事务的xid都一样，binglog position也一样，但是会有不同的xoffset，提交时会有xcommit

* ts粒度太大不够精确，批量更新时，ts都一样，如果需要排序，需要二次开发，增加一个微秒级别的时间戳

* Mysql DateTime和Timestamp类型都会转换为yyyy-MM-dd HH:mm:ss形式，DateFormat会判断是否为isTimeStamp

* DateTime，TimeStamp，Time，相同的时间，Maxwell输出时，TimeStamp 小8小时，Time 大8小时，需要二次开发进行改造

* Time类型有Bug，和Mysql显示不一样，会把格林威治时间当中国时间显示，时间会增加8小时

* Maxwell是基于事件触发的，单线程处理binlog同步

* 每一个ClientId，第一次初始化时，需要同步数据库的元数据信息，以及Filter对应的Position，可能会比较慢

* Maxwell库的数据库元数据信息不能随便删，否则会无法构建对象，各种启动异常

* BootStrap模式可以直接在库里增加数据出发同步      

* DML和DDL是两套不同的处理逻辑，DDL的改变会持久化到Maxwell的数据库

* Maxwell 会格式化Filter表达式，可以不输入空格进行间隔

* 如果Binlog被删除，Position位置早于Mysql当前binlog文件位置，Maxwell会启动后异常退出
    * 异常信息：Could not find first log file name in binary log index file

* 如果采用Kafka方式连接，当topic不存在或无法创建时，Maxwell会启动后异常退出

* 改造Maxwell输出考虑从BinlogConnectorReplicator类下手，此类可以直接修改RowMap和DDLMap

* MaxwellContext也是一个较关键的类，用于获取上下文信息

* SchemaStore用于获取抓取库的元数据

* RowMap和DDLMap可以获取DML和DDL的全部嘻嘻

* DateFormatter 用于格式化日期类
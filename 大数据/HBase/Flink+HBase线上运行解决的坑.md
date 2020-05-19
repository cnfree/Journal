# Flink+HBase线上运行解决的坑

## Kerberos 认证部分
* 集群时钟同步问题
  * 访问客户端和集群需要保持时钟一致
* HBase kerberos 问题
  * 需要从CDH下载HBase客户端配置，放置到本地应用资源文件夹
* Flink hdfs checkpoint kerberos问题
  * Caused by: org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.security.authorize.AuthorizationException): User: hbase/cdhlx01@LXKS.COM is not allowed to impersonate hdfs/cdhlx01@LXKS.COM
  * 需要修改hdfs配置，增加
  ```xml
    <property>
      <name>hadoop.proxyuser.hbase.hosts</name>
      <value>*</value>
    </property>
    <property>
      <name>hadoop.proxyuser.hbase.groups</name>
      <value>*</value>
    </property>
  ```

## 域名映射部分
* CDH集群Kafka无法访问
  * 和普通的Kafka集群不同，CDH的Kafka集群需要配置本地域名映射才能访问

## Flink 部分
* Flink内存模型
  * 参见 https://ci.apache.org/projects/flink/flink-docs-release-1.10/zh/ops/memory/mem_detail.html
  * 任务堆内存（Task Heap Memory）用于 Flink 应用的算子及用户代码的 JVM 堆内存。
  * 托管内存（Managed memory）由 Flink 管理的用于排序、哈希表、缓存中间结果及 RocksDB State Backend 的本地内存。
  * taskmanager.memory.managed.fraction：托管内存占Flink总内存taskmanager.memory.flink.size的比例，默认值0.4
  * 可根据实际情况调整 taskmanager.memory.managed.fraction，减少托管内存占用。
  * 参数 taskmanager.numberOfTaskSlots，用于提升并行度，一个job的多个算子可以共用slot

* Flink metaspace out of memory
   * 默认96M, 当Job过多时，会OOM
   * 调整配置参数 taskmanager.memory.jvm-metaspace.size: 512m

## HBase 部分
* HBase数据模型
  * https://blog.csdn.net/WYpersist/article/details/79820715
  * https://blog.csdn.net/xiaoshunzi111/article/details/69844526
* HBase内存模型
  * https://www.pianshen.com/article/2343339600/
* HBase Region计算公式
  * 最佳Region数: heapsize * (Memstore占比)/(Flush Size, 128M)
  * 实际Region数: 一张表至少一个Region, 当大于Region Max File Size(默认10G)，会分裂Region
* HBase 内存计算公式
  * Memstore之和 < HeapSize * Memstore占比上限，否则会导致Region刷盘
* HBase不健康的几种原因
  * Flush Queue Size
  * Compaction Queue Size
  * GC Duration （Memstore不够时，会导致长时间的OOM, 此时Phoenix客户端可能会连接超时）
* HBase集群无法启动
  * 一个Regionserver上有一个BlockCache和N个Memstore，它们的大小之和不能大于等于heapsize * 0.8，否则HBase不能正常启动
  * 参见 https://my.oschina.net/wenziqiu/blog/840843
## Phoenix 部分
* Phoenix Schema映射
  * 参见 https://blog.csdn.net/sinat_36121406/article/details/82863420
* Phoenix 创建索引失败
  * 参见 https://www.jianshu.com/p/a3c24638b498
* Phoenix 索引失效重建，参见
  * https://www.cnblogs.com/haoxinyue/p/6747948.html
  * https://www.cnblogs.com/hbase-community/p/8879848.html
* Phoenix 索引执行计划
  * 执行 explain sql, RANGE SCAN 代表走索引，FULL SCAN则为全表扫描
  * select * from record_mysql_ibank_user.t_user where bigdata_ts = 1 **不会走索引**
  * select bigdata_ts from record_mysql_ibank_user.t_user where bigdata_ts = 1 会走索引
  * select bigdata_id from record_mysql_ibank_user.t_user where bigdata_ts = 1 会走索引
  * select count(*) from record_mysql_ibank_user.t_user where bigdata_ts = 1 会走索引

* Phoenix Core访问无kerberos认证的HBase出现kerberos的问题
  * 查看是否引入了错误的hbase client配置(包含 core-site.xml, hbase-site.xml, hdfs-site.xml)
  * 如有问题，从CDH HBase下载最新的客户端配置

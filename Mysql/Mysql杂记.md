# Mysql杂记

## Select Where For Update
* 如果Where条件没有建立索引，表锁。
* 如果建立索引，但未命中，Gap锁。
* 如果建立索引，并命中，索引为**唯一索引，行锁，非唯一索引，临键锁**。
* 以下情况如果GAP区间有交集会发生死锁：
```
    A事务                       B事务
    set @@autocommit=0;
    start transaction;         set @@autocommit=0;
    select where for update;   start transaction;
                               select where for update; 
    insert;
    commit;
                               insert; 
                               commit;
                               
    Session B 尝试获取插入意向锁，需要等待 Session A 的gap锁
    Session A 事务的插入意向锁处于等待中
    Session A 事务插入意向锁在等待 SessionB 的gap锁
    形成环路，检测到死锁
```

* 以下情况如果GAP区间有交集不会死锁，B事务会堵塞等待：
```
    A事务                       B事务
    set @@autocommit=0;
    start transaction;         set @@autocommit=0;
    select where for update;   start transaction;
    insert;                    select where for update; //堵塞等待 Session A Commit

    commit;                    insert; 
                               commit;
```
* INSERT ... ON DUPLICATE KEY UPDATE 一样会有这个问题。

## In 和 Exists
```
    1. select * from A where id in (select id from B);
    B表驱动A表，先查询B表，再查询A表，适用于B表较小的情况
    
    2. select * from A where exists (select 1 from B where B.id = A.id);
    A表驱动B表，先查询A表，再查询B表，适用于A表较小的情况
    
    表A（小表），表B（大表）
    
    select * from A where cc in (select cc from B) ;//  效率低，用到了A表上cc列的索引；
    select * from A where exists(select cc from B where cc=A.cc) ;// 效率高，用到了B表上cc列的索引。
    
    select * from B where cc in (select cc from A) ; //效率高，用到了B表上cc列的索引；
    select * from B where exists(select cc from A where cc=B.cc) ;//效率低，用到了A表上cc列的索引。
    
``` 
* 如果查询的两个表大小相当，那么用in和exists差别不大。

## not in 和not exists
 * 如果查询语句使用了not in 那么内外表都进行全表扫描，没有用到索引。
 * 而not extsts 的子查询依然能用到表上的索引。
 * 所以**无论那个表大，用not exists都比not in要快**。


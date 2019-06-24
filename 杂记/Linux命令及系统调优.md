# Linux 命令及系统调优

## Linux Performance Observablilty Tools

![Linux Performance Observablilty Tools][Linux]


## 查看各个CPU核的使用情况

   ```
   sudo top -d 1
        
   进入之后，按1，会出现以下的CPU使用情况，当中us列反映了各个CPU核的使用情况，百分比大说明该核在进行紧张的任务。
    
   ```

## 查看某个进程内部线程占用情况分析
   
   ```
   top -H -p 18605 
   ```

## 查看哪个进程在哪个CPU核上执行
    
   ```
   sudo top -d 1
    
   进入之后，依次按f、j和空格，会出现例如以下（当中P列指示的是该进程近期使用的CPU核，如进程mencoder的P列为7，则表示mencoder近期在核7上执行，对于多线程甚至单线程的进程，在不同一时候刻会使用不同的CPU Core）
   ```

## vmstat查看总体的CPU使用情况        
   ```
   sudo vmstat 2 3
        
   參数2表示每一个2秒显示一下结果，3表示显示结果的数目。
        
   cs列表示每秒上下文切换次数，us表示用户CPU时间。
    
   ```

## pidstat实时查看一个进程的CPU使用情况及上下文切换情况
   ```
    pidstat -p 15488 2（你要追踪的进程的pid）
    这样就能每隔2秒显示15488进程的CPU使用情况：
    
    idstat -w —— 显示每一个进程的上下文切换情况
    pidstat -w -p 15488 2 —— 每隔2秒显示15488进程的上下文切换情况：
    
    cswch/s —— 每秒该进程产生的voluntary context switches总数。
    voluntary context switches出如今訪问一个已经被占用的资源，从而不得不挂起（即我们通常说的Synchronization Context Switches）
    
    nvcswch/s —— 每秒该进程产生的involuntary context switches总数。
    involuntary  context switches发生在自己的时间片用完或被更高的优先级抢占（包括Preemption Context Switches）

   ```

## iostat用于输出CPU和磁盘I/O相关的统计信息，

   通过iostat方便查看CPU、网卡、tty设备、磁盘、CD-ROM 等等设备的活动情况,负载信息。

   ```
   iostat -mtx 2
   ```    
    
## iotop

  用来监视磁盘I/O使用状况的工具。
  
  iotop具有与top相似的UI，其中包括PID、用户、I/O、进程等相关信息。
  
  Linux下的IO统计工具如iostat，nmon等大多数是只能统计到per设备的读写情况，如果你想知道每个进程是如何使用IO的就比较麻烦，使用iotop命令可以很方便的查看。    
  
## netstat在内核中访问网络连接状态及其相关信息

  netstat能提供TCP连接，TCP和UDP监听，进程内存管理的相关报告。 
  
  ```
    netstat -nat|grep 7001|wc -l
    查看7001端口的连接数
    
    netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
    查看服务器的各种连接状态，其中ESTABLISHED 就是并发连接状态的显示数
  ``` 
  
## ps -ef |grep java 

  查看Java进程是否存在  
  
## df -lh 查看磁盘占用情况  
 
  ```
    Used：已经使用的空间 
    Avail：可以使用的空间 
    Mounted on：挂载的目录
  ```
  
## 查找100M以上文件

   ```
     find / -type f -size +100M -print0 | xargs -0 du -h | sort -nr
   ``` 
   
## 查找当前目录下前20的大目录  
 
   ```
     sudo du -hm --max-depth=2 | sort -nr | head -20
   ```
   
## free -m 查看linux内存使用情况

   -m 参数就是用 M显示内容使用情况
   
   ```
     buffer ：作为buffer cache的内存，是块设备的读写缓冲区 
     
     cache：作为page cache的内存， 文件系统的cache 
     
     如果 cache 的值很大，说明cache住的文件数很多。如果频繁访问到的文件都能被cache住，那么磁盘的读IO 必会非常小
   ```   
     
   





[Linux]: http://www.brendangregg.com/Perf/linux_observability_tools.png
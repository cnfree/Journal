# Flink基本概念

## 架构图
![Flink1]
![Flink2]

  Flink 运行时架构主要包含几个部分：Client、JobManager(master节点)和TaskManger(slave节点)。
  * Client：Flink 作业在哪台机器上面提交，那么当前机器称之为Client。用户开发的Program 代码，它会构建出DataFlow graph，然后通过Client提交给JobManager。
  * JobManager：是主（master）节点，相当于YARN里面的ResourceManager，生成环境中一般可以做HA高可用。JobManager会将任务进行拆分，调度到TaskManager上面执行。
  * TaskManager：是从节点（slave），TaskManager才是真正实现task的部分
  * ResourceManager：一般是Yarn，当TM有空闲的slot就会告诉JM，没有足够的slot也会启动新的TM。kill掉长时间空闲的TM。
  *nDispatcher（Application Master）提供REST接口来接收client的application提交，它负责启动JM和提交application，同时运行Web UI。
  
## Application Deployment
 
   * Framework模式：Flink作业为JAR，并被提交到Dispatcher or JM or YARN。
   * Library模式：Flink作业为application-specific container image，如Docker image，适合微服务。

## Savepoints

checkpoints加上一些额外的元数据，功能也是在checkpoint的基础上丰富。不同于checkpoints，savepoint不会被Flink自动创造（由用户或者外部scheduler触发创造）和销毁。

savepoint可以重启不同但兼容的作业，从而：

* 修复bugs进而修复错误的结果，也可用于A/B test或者what-if场景。
* 调整并发度
* 迁移作业到其他集群、新版Flink
* 也可以用于暂停作业，通过savepoint查看作业情况。

## 参考文章
 * [Flink架构及其工作原理]
 * [Apache Flink - 架构和拓扑]
 * [精通Apache Flink读书笔记--1、2]
 * [精通Apache Flink读书笔记--3、4]


[Flink1]: img/Flink1.png
[Flink2]: img/Flink2.jpeg
[Flink架构及其工作原理]: https://www.cnblogs.com/code2one/p/10123112.html
[Apache Flink - 架构和拓扑]: https://www.cnblogs.com/ooffff/p/9476032.html
[精通Apache Flink读书笔记--1、2]: https://blog.csdn.net/lmalds/article/details/60575205
[精通Apache Flink读书笔记--3、4]: https://blog.csdn.net/lmalds/article/details/60867262
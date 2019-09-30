# Skywalking架构

## 架构图
![Skywalking]

## 可视化维度
  目前Skywalking已经支持从6个可视化维度剖析分布式系统的运行情况。

* 总览视图是应用和组件的全局视图，其中包括组件和应用数量，应用的告警波动，慢服务列表以及应用吞吐量；
* 拓扑图从应用依赖关系出发，展现整个应用的拓扑关系；
* 应用视图则是从单个应用的角度，展现应用的上下游关系，TopN的服务和服务器，JVM的相关信息以及对应的主机信息。
* 服务视图关注单个服务入口的运行情况以及此服务的上下游依赖关系，依赖度，帮助用户针对单个服务的优化和监控；
* 调用链展现了调用的单次请求经过的所有埋点以及每个埋点的执行时长；
* 告警视图根据配置阈值针对应用、服务器、服务进行实时告警。

## JavaAgent 机制
  SkyWalking Agent 基于 JavaAgent 机制
  
  javaagent 的主要功能如下：
  * 可以在加载 class 文件之前做拦截，对字节码做修改
  * 可以在运行期对已加载类的字节码做变更，但是这种情况下会有很多的限制，后面会详细说
  * 获取所有已经加载过的类
  * 获取所有已经初始化过的类（执行过 clinit 方法，是上面的一个子集）
  * 获取某个对象的大小
  * 将某个 jar 加入到 bootstrap classpath 里作为高优先级被 bootstrapClassloader 加载
  * 将某个 jar 加入到 classpath 里供 AppClassloard 去加载
  * 设置某些 native 方法的前缀，主要在查找 native 方法的时候做规则匹配
  
  在 Java SE 5 及其后续版本当中，开发者可以在一个普通 Java 程序（带有 main 函数的 Java 类）运行时，通过 – javaagent参数指定一个特定的 jar 文件（包含 Instrumentation 代理）来启动 Instrumentation 的代理程序。
  
  简要说来就是如下几个步骤：
  
  1. 编写 **premain** 函数
     * 编写一个 Java 类，包含如下两个方法当中的任何一个
          ```java
           public static void premain(String agentArgs, Instrumentation inst);  [1]
           public static void premain(String agentArgs); [2]
           //其中，[1] 的优先级比 [2] 高，将会被优先执行（[1] 和 [2] 同时存在时，[2] 被忽略）。
          ```
     * 在这个 premain 函数中，开发者可以进行对类的各种操作。
  
     * agentArgs 是 premain 函数得到的程序参数，随同 **–javaagent** 一起传入。与 main 函数不同的是，这个参数是一个字符串而不是一个字符串数组，如果程序参数有多个，程序将自行解析这个字符串。
  
     * Inst 是一个 java.lang.instrument.Instrumentation 的实例，由 JVM 自动传入。java.lang.instrument.Instrumentation 是 instrument 包中定义的一个接口，也是这个包的核心部分，集中了其中几乎所有的功能方法，例如类定义的转换和操作等等。
  
  2. jar 文件打包
    
      将这个 Java 类打包成一个 jar 文件，并在其中的 manifest 属性当中加入**Premain-Class**来指定步骤 1 当中编写的那个带有 premain 的 Java 类。（可能还需要指定其他属性以开启更多功能）
  
  3. 运行
    
      用如下方式运行带有 Instrumentation 的 Java 程序：
      ```java  
        java -javaagent:jar 文件的位置 [= 传入 premain 的参数 ]
      ```
  
## Skywalking Agent实现
  * 通过 JavaAgent 机制，在 #premain(String, Instrumentation) 方法里，调用 Instrumentation#addTransformer(ClassFileTransformer) 方法，向 Instrumentation 注册 java.lang.instrument.ClassFileTransformer 对象，可以修改 Java 类的二进制，从而**动态**修改 Java 类的代码实现。
  * 使用不同框架定义方法切面，从而在在切面记录调用链路。
  * SkyWalking 引入了 byte-buddy 修改Java类的二进制
  
      >byte-buddy 是一个代码生成和操作库，用于在 Java 应用程序运行时创建和修改 Java 类，而徐无需编译器的帮助。 
      > 
      >除了参与 Java 类库一起提供代码生成工具外，byte-buddy 允许创建任意类，并不限于实现用于创建运行时代理的接口。
      >
      > 此外，byte-buddy 提供了一个方便的 API ，用于 Java Agent 或在构建过程中更改类。
  
  SkyWalking 通过 JavaAgent 机制，对需要拦截的类的方法，使用 byte-buddy 动态修改 Java 类的二进制，从而进行方法切面拦截，记录调用链路。
  
  ![Agent]
  
  拦截器组件：
  * 拦截切面： InterceptPoint
  * 拦截器： Interceptor
  * 拦截类的定义：Define 一个类有哪些拦截切面及对应的拦截器
  * Inter：provide a bridge between byte-buddy and sky-walking plugin 
  
## 后端设计
  * 模块化设计；
  * 为客户端提供多种连接方式；
  * 集群发现机制；
  * 流模式；
  * 可切换的存储实现；
  
  1. 模块化
     * SkyWalking收集器（collector）是基于模块化设计，用户可以根据自己的需要，更改或集成收集器的功能。
  
  2. 模块
     * 模块定义了一组特性，其中可包括一些技术上的实现（如：grpc/jetty服务器管理）、跟踪分析（如：trace segment或者zipkin span解析器）或聚合特征。总而言之，这些都是由模块来定义和实现的。
     * 每个模块都可以通过Java接口定义自身的服务，而实现类均要实现这些服务。并且这些实现类要根据实现的功能定义所依赖的类有哪些。这意味着，即使是模块的两个不同的实现，也可以依赖于不同的模块。
     * 另外，收集器中的模块化核心会检查启动序列，如果没有发现循环依赖或者依赖项，该核心功能会终止收集器
     * 收集器会启动所有模块，这些模块在application.yml文件中定义。此文件结构如下
        * 根节点是模块名称，如：cluster，naming
        * 次级节点是此模块的功能实现名称，如：zookeeper是cluster模块
        * 第三级节点是功能实现的属性，如：hostPort和sessionTimeout是zookeeper需要的属性
  
  3. 多连接方式
     * 收集器提供两种类型的连接，也就是两种协议的支持：HTTP和gRPC。
        * 在HTTP中命名服务，在后端集群中，返回所有可用的收集器；
        * Uplink服务支持gRPC（主要用于SkyWalking的本地代理）和HTTP，它跟踪和度量收集器。每个客户端只向单个收集器发送监测数据（跟踪和度量）。若连接的收集器断线，，则尝试连接其他的收集器。
  
     * 客户端lib和收集器集群之间的处理流示例
   　　![collector]
  
  4. 收集器集群发现
  
     当收集器以集群模式运行时，收集器必须以某种方式发现彼此。在默认情况下，SkyWalking使用zookeeper进行协调，并以此作为发现的注册中心。
  
  5. 流模式
     * 流模式倾向于轻量级的storm/spark实现，并允许使用api来构建流过程图（DAG），以及每个节点的输入/输出的数据约定。
     * 新模块可以找到并扩展已有的过程图。
     * 在处理过程中有三种情况：
        1. 同步过程。传统的方法调用。
        2. 异步过程，基于队列缓冲区的a.k.a批处理过程。
        3. 远程过程，聚合矩阵收集器，通过这种方式，选择器在节点中定义，以决定如何在集群中找到收集器。（HashCode，Rolling，ForeverFirst是三种支持的方式）
     * 通过这些特性，收集器就像一个流动的网一样运行。通过聚合指标和不依赖于存储实现功能来支持同时编写同样的id。
  
  6. 可切换的存储实现
     * 因为流模式负责并发，所以存储实现的职责是提供高速写和组查询。
     * 目前支持ElasticSearch，也支持H2预览版，同时支持ShardingSphere项目用于MySql关系数据库集群的管理。
  
  7. Web UI
     * 除了收集器设计的原则之外，UI也是SkyWalking中的另一个核心部分。它基于React、Antd和Zuul代理来提供收集器集群发现、查询分派和可视化。
     * Web UI使用localhost:10800来为收集器集群做命名查询。
     
## 协议
  
  * Trace Data Protocol是skywalking中描述探针与后端数据通信的数据协议。协议描写了探针上行、下行数据格式，可用于客户自定义探针使用，其中：
    * 服务发现功能只提供HTTP上报格式
    * 其他服务，例如注册，Trace等，提供HTTP/JSON和gRPC两种上报接口。
    * 协议文档：https://github.com/apache/skywalking/blob/master/docs/en/protocols/Trace-Data-Protocol-v2.md
  
  * Skywalking Cross Process Propagation Headers Protocol
    * Skywalking是一个偏向APM的分布式追踪系统，所以，为了提供服务端处理性能。头信息会比其他的追踪系统要更复杂一些。 你会发现，这个头信息，更像一个商业APM系统。
    * 协议文档：https://github.com/apache/skywalking/blob/master/docs/en/protocols/Skywalking-Cross-Process-Propagation-Headers-Protocol-v2.md      
  
## 命名空间 
  SkyWalking是一个用于从分布式系统收集指标的监控工具。在实际环境中，一个非常大的分布式系统包括数百个应用程序，数千个应用程序实例。在这种情况下，更大可能的不止一个组，甚至还有一家公司正在维护和监控分布式系统。他们每个人都负责不同的部分，不能共享某些指标。

  在这种情况下,命名空间就应运而生了,它用来隔离追踪和监控数据。

  **当双方使用不同的名称空间时，跨进程传播链会中断。**

  在探针配置中配置 agent.namespace
  ```properties
   agent.namespace=default-namespace
  ```

## 数据采集  
 ![数据采集]

## 参考文档
* [Skywalking容量规划]
  b
       
[Skywalking]:img/Skywalking.png
[Agent]:img/Agent.png
[collector]:img/collector.jpg
[数据采集]:img/数据采集.png
[Skywalking容量规划]: https://blog.csdn.net/flysqrlboy/article/details/89009200
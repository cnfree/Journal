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
  
  
  
  
       
[Skywalking]:img/Skywalking.png
[Agent]:img/Agent.png
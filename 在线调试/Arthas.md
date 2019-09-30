# Arthas

 ## Arthas 组成模块

![arthas]

  * **arthas-boot.jar**和**as.sh**
    * 两个模块功能类似，分别使用java和shell脚本，下载对应的jar包，并生成服务端和客户端的启动命令，然后启动客户端和服务端。
    * 服务端最终生成的启动命令如下：
      ```bash
      ${JAVA_HOME}"/bin/java \
         ${opts}  \
         -jar "${arthas_lib_dir}/arthas-core.jar" \
             -pid ${TARGET_PID} \                            要注入的进程id
             -target-ip ${TARGET_IP} \                       服务器ip地址
             -telnet-port ${TELNET_PORT} \                   服务器telnet服务端口号
             -http-port ${HTTP_PORT} \                       websocket服务端口号
             -core "${arthas_lib_dir}/arthas-core.jar" \     arthas-core目录
             -agent "${arthas_lib_dir}/arthas-agent.jar"     arthas-agent目录
      ```
  * **arthas-core.jar**
     * 是服务端程序的**启动入口**，会调用virtualMachine#attach到目标进程，并加载arthas-agent.jar作为agent jar包。
  * **arthas-agent.jar**
     * 既可以使用**premain**方式（在目标进程启动之前，通过-agent参数静态指定），也可以通过**agentmain**方式（在进程启动之后**attach**上去）。
     * arthas-agent会使用自定义的classloader(ArthasClassLoader)加载arthas-core.jar里面的com.taobao.arthas.core.config.Configure类以及com.taobao.arthas.core.server.ArthasBootstrap。
     * 同时程序运行的时候会使用arthas-spy.jar。
  * **arthas-spy.jar**
     * 里面只包含Spy类，目的是为了将Spy类使用BootstrapClassLoader来加载，从而使目标进程的java应用可以访问Spy类。通过ASM修改字节码，可以将Spy类的方法ON_BEFORE_METHOD，ON_RETURN_METHOD等编织到目标类里面。
     * Spy类你可以简单理解为类似spring aop的Advice，有前置方法，后置方法等。
  * **arthas-client.jar**
     * 客户端程序，用来连接arthas-core.jar启动的服务端代码，使用telnet方式。
     * 一般由arthas-boot.jar和as.sh来负责启动。


 ## 需要注意的地方
 * **_使用exit或者quit命令只会退出交互界面，不会关闭attach的arthas进程。_**
 * 如果需要重连，或者采用web console，需要连接tunnel server时，需要先shutdown，否则不会成功。

 ## WEB Console
 * Tunnel server/client use websocket protocol.
 * 工作原理: **browser <-> arthas tunnel server <-> arthas tunnel client <-> arthas agent**
 * [Web Console]
 * [Tunnel Server]
 * 暂不支持复制粘贴，仍然只是一个半成品

 ## 使用方法
   * 下载arthas
     ```bash
     curl -L https://alibaba.github.io/arthas/install.sh | sh
     ```
   * 执行 ./as.sh

 ## ognl 表达式使用
   * 执行多行表达式，赋值给临时变量，返回一个List：
     ```shell
      $ ognl '#value1=@System@getProperty("java.home"), #value2=@System@getProperty("java.runtime.name"), {#value1, #value2}'
      @ArrayList[
          @String[/opt/java/8.0.181-zulu/jre],
          @String[OpenJDK Runtime Environment],
      ]
     ```

 ## thread -b 找出当前阻塞其他线程的线程

 有时候我们发现应用卡住了，通常是由于某个线程拿住了某个锁，并且其他线程都在等待这把锁造成的。为了排查这类问题，arthas提供了thread -b，一键找出那个罪魁祸首。

 目前只支持找出synchronized关键字阻塞住的线程，如果是java.util.concurrent.Lock，目前还不支持。

 ![block]


 

 ## 参考资料
   * [arthas源码分析]
   * [JVM进程诊断利器——arthas介绍]
   a





[arthas]: img/arthas.webp
[arthas源码分析]: https://www.jianshu.com/p/4e34d0ab47d1
[JVM进程诊断利器——arthas介绍]:https://www.jianshu.com/p/76d9a81ede7e
[Web Console]:https://alibaba.github.io/arthas/web-console.html
[Tunnel Server]:https://github.com/alibaba/arthas/blob/master/tunnel-server/README.md
[block]: img/block.jpg
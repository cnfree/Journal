# Arthas

# Arthas 组成模块

![arthas]

  * **arthas-boot.jar**和**as.sh**
    * 两个模块功能类似，分别使用java和shell脚本，下载对应的jar包，并生成服务端和客户端的启动命令，然后启动客户端和服务端。
    * 服务端最终生成的启动命令如下：
      ```shell
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
     * 是服务端程序的**启动入口类**，会调用virtualMachine#attach到目标进程，并加载arthas-agent.jar作为agent jar包。
  * **arthas-agent.jar**
     * 既可以使用**premain**方式（在目标进程启动之前，通过-agent参数静态指定），也可以通过**agentmain**方式（在进程启动之后**attach**上去）。
     * arthas-agent会使用自定义的classloader(ArthasClassLoader)加载arthas-core.jar里面的com.taobao.arthas.core.config.Configure类以及com.taobao.arthas.core.server.ArthasBootstrap。
     * 同时程序运行的时候会使用arthas-spy.jar。
  * **arthas-spy.jar**
     * 里面只包含Spy类，目的是为了将Spy类使用BootstrapClassLoader来加载，从而使目标进程的java应用可以访问Spy类。通过ASM修改字节码，可以将Spy类的方法ON_BEFORE_METHOD， ON_RETURN_METHOD等编织到目标类里面。
     * Spy类你可以简单理解为类似spring aop的Advice，有前置方法，后置方法等。
  * **arthas-client.jar**
     * 客户端程序，用来连接arthas-core.jar启动的服务端代码，使用telnet方式。
     * 一般由arthas-boot.jar和as.sh来负责启动。


 ## 参考资料
   * [arthas源码分析]








[arthas]: img/arthas.webp
[arthas源码分析]: https://www.jianshu.com/p/4e34d0ab47d1
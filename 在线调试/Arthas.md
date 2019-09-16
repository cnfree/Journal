# Arthas

# Arthas 组成模块

![arthas]

  * arthas-boot.jar和as.sh
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









[arthas]: img/arthas.webp
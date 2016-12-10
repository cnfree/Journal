# Docker使用手册
关键字： `Docker` `安装` `使用` `Redhat`
## 安装 Docker
1. 操作系统：Radhat，内核: `2.6.32-431.11.2.el6.x86_64`。
* 该版本操作系统，Docker最高仅支持到 **1.7.1** 版。  
* 附件：[docker1.7.1.zip][1] 下载包
* 解压 docker1.7.1.zip，里面是一个单二进制文件的docker。
* 后台启动 docker 守护进程命令： `nohup ./docker -d &`
* 如果启动失败，可能需要安装依赖 `libcgroup`。  
 -  安装 libcgroup `yum install libcgroup`  
 -  启动cgconfig服务 `sudo /etc/init.d/cgconfig start`   
 -  将cgconfig设为开机启动 `chkconfig cgconfig on`

## 使用 Docker

### Docker 常用命令
* 下载镜像：`docker pull [OPTIONS] NAME[:TAG|@DIGEST]`
* 列出本地所有镜像：`docker images [OPTIONS] [REPOSITORY]`
* 列出所有运行中容器：`docker ps [OPTIONS]`
* 删除容器：`docker rm [OPTIONS] CONTAINER [CONTAINER...]`
* 删除镜像：`docker rmi [OPTIONS] IMAGE [IMAGE...]`
* 启动、停止和重启容器：`docker start|stop|restart [OPTIONS] CONTAINER [CONTAINER...]`
* 杀死容器进程：`docker kill [OPTIONS] CONTAINER [CONTAINER...]`
* 启动容器：`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`
* 更详细的参数，请参见文章 [Docker命令使用详解][2]。

### Docker 仓库
* 仓库地址：[https://hub.docker.com][3]
* SSH镜像地址：[https://hub.docker.com/r/rastasheep/ubuntu-sshd][4]

### SSH容器
* 启动 ssh 容器：   
  `docker run -d -P --name ssh_docker rastasheep/ubuntu-sshd:16.04`
* 查看容器 ssh 外部端口：`docker port ssh_docker 22`
* 通过 ssh 工具或命令连接容器：`ssh root@localhost -p [port]`
* ssh 镜像缺省用户名和密码均为：`root`

### 安装和使用Java运行环境
* 可以通过 Dockerfile 来安装Java, 详情参见 [博文][5]，镜像制作完毕后600多M。
* 通过 Ubuntu 命令安装 OpenJDK：`apt-get install openjdk-8-jre`。
* 制作 Java 镜像：`docker commit CONTAINER cnfree/sshjava`，大小为480M。
* 导出 Java 镜像：`docker save -o sshjava.tar cnfree/sshjava`。
* 导入 Java 镜像：`docker load < sshjava.tar`。
* 运行 Java 镜像容器：`docker run -d -p 2222:22 --name ssh_docker cnfree/sshjava`。
* 执行 Shll：`docker exec -i -t ssh_docker /bin/bash`。


## 参考资料
* [Docker-容器的操作][6]
* [深入分析Docker镜像原理][7]
* [一篇不一样的docker原理解析][8]
* [全网贯通无需端口映射][9]
* [非常详细的 Docker 学习笔记][10]
* [Docker从入门到实践][11]


  [1]: https://github.com/cnfree/Journal/raw/master/Docker/docker1.7.1.zip
  [2]: http://www.server110.com/docker/201411/11122.html
  [3]: https://hub.docker.com
  [4]: https://hub.docker.com/r/rastasheep/ubuntu-sshd
  [5]: http://blog.csdn.net/zhang__jiayu/article/details/43200685
  [6]: http://www.cnblogs.com/zydev/p/5803461.html
  [7]: http://www.csdn.net/article/2015-08-21/
  [8]: https://zhuanlan.zhihu.com/p/22382728
  [9]: http://dockone.io/article/466
  [10]: http://www.open-open.com/lib/view/open1423703640748.html
  [11]: https://github.com/cnfree/Journal/raw/master/Docker/Docker从入门到实践.pdf

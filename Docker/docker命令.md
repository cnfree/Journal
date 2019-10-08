# Docker命令

## docker run ：创建一个新的容器并运行一个命令
#### 语法
```sh
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```
OPTIONS说明：
* -a stdin: 指定标准输入输出内容类型，可选 STDIN/STDOUT/STDERR 三项；
* -d: 后台运行容器，并返回容器ID；
* -i: 以交互模式运行容器，通常与 -t 同时使用；
* -p: 端口映射，格式为：主机(宿主)端口:容器端口
* -t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
* --name="nginx-lb": 为容器指定一个名称；
* --dns 8.8.8.8: 指定容器使用的DNS服务器，默认和宿主一致；
* --dns-search example.com: 指定容器DNS搜索域名，默认和宿主一致；
* -h "mars": 指定容器的hostname；
* -e username="ritchie": 设置环境变量；
* --env-file=[]: 从指定文件读入环境变量；
* --cpuset="0-2" or --cpuset="0,1,2": 绑定容器到指定CPU运行；
* -m :设置容器使用内存最大值；
* --net="bridge": 指定容器的网络连接类型，支持 bridge/host/none/container: 四种类型；
* --link=[]: 添加链接到另一个容器；
* --expose=[]: 开放一个端口或一组端口；

# docker exec ：在运行的容器中执行命令
#### 语法
```sh
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```
OPTIONS说明：
* -d :分离模式: 在后台运行
* -i :即使没有附加也保持STDIN 打开
* -t :分配一个伪终端

通过 exec 命令对指定的容器执行 bash:
```sh
docker exec -it 9df70f9a0714 /bin/bash
```

# Detached VS Foreground
当我们启动一个容器时，首先需要确定这个容器是运行在前台还是运行在后台。

## Detached (-d)
如果在docker run后面追加-d=true或者-d，那么容器将会运行在后台模式。此时所有I/O数据只能通过网络资源或者共享卷组来进行交互。因为容器不再监听你执行docker run的这个终端命令行窗口。但你可以通过执行docker attach来重新附着到该容器的会话中。需要注意的是，容器运行在后台模式下，是不能使用--rm选项的。

如果你的容器以后台方式运行，只有在父进程即docker进程退出的时候才会去把容器退出，除非你使用了--rm选项。如果你在运行容器时将-d和--rm两个选项一起使用，那么容器会在退出或者后台进程停止的的时候自动移除掉（只要一个情况便会自动移除镜像）。

docker容器后台运行是不能通过service x start来启动的，比如想启动一个后台运行的nginx服务：
```sh
$ docker run -d -p 80:80 my_image service nginx start
```
这样虽然启动了容器内的nginx服务，但是是不可用的

## Foregroud
在前台模式下（不指定-d参数即可），Docker会在容器中启动进程，同时将当前的命令行窗口附着到容器的标准输入、标准输出和标准错误中。也就是说容器中所有的输出都可以在当前窗口中看到。甚至它都可以虚拟出一个TTY窗口，来执行信号中断。

# 退出容器
### 在docker内exit
结束退出, 退出的容器还可以再次进入

### Ctrl-q + Ctrl-p
CTRL-q+CTRL-p方式比较简单，只需要注意docker run时要同时指定-it选项。该方式只会退出docker attach，对容器没有影响

如果-it选项没有同时指定，CTRL-q+CTRL-p无法生效

### 在另一个终端杀死docker attach进程

# docker -v 挂载
启动一个centos容器，宿主机的/test目录挂载到容器的/soft目录，可通过以下方式指定：
```sh
docker run -it -v /test:/soft centos /bin/bash
```
-v参数中，冒号":"前面的目录是宿主机目录，后面的目录是容器内目录。
* 容器目录不可以为相对路径
* 宿主机目录如果不存在，则会自动生成
* 宿主机的目录如果为相对路径:
  * 通过docker inspect命令，查看容器“Mounts”那一部分
  * 相对路径指的是/var/lib/docker/volumes/，与宿主机的当前目录无关。
* 如果只是-v指定一个目录
  * 通过docker inspect命令，查看容器“Mounts”那一部分
  * 在宿主机/var/lib/docker/volumes/目录下随机生成的一个目录名

## 容器销毁了，在宿主机上新建的挂载目录是否会消失
两种情况：
1. 指定了宿主机目录，即 -v /test:/soft
  * 不会消失
2. 没有指定宿主机目录，即-v /soft
  * 不会消失

docker run --rm选项会清理容器的匿名data volumes

**执行docker run命令带--rm命令选项，等价于在容器退出后，执行docker rm -v，可以用于容器日志清理**

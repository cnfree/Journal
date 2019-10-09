# Docker日志收集最佳实践

在linux上，容器日志一般存放在**/var/lib/docker/containers/<contariner ID>**下面

# 查看容器的日志
```sh
$ docker logs [OPTIONS] CONTAINER
```
**Options:**
* --details        显示更多的信息
* -f, --follow         跟踪实时日志
* --since string   显示自某个timestamp之后的日志，或相对时间，如42m（即42分钟）
* --tail string    从日志末尾显示多少行日志， 默认是all
* -t, --timestamps     显示时间戳
* --until string   显示自某个timestamp之前的日志，或相对时间，如42m（即42分钟）

## logging driver
将容器日志发送到STDOUT和STDERR是Docker的默认日志行为。实际上，Docker 提供了多种日志机制帮助用户从运行的容器中提取日志信息。这些机制被称作**logging driver**。

Docker的默认logging driver是**json-file**。

json-file会将容器的日志保存在json文件中，Docker负责格式化其内容并输出到STDOUT和STDERR。

我们可以在Host的容器目录中找到这个文件，文件路径为 **/var/lib/docker/containers/<contariner ID>/<contariner ID>-json.log**

## 参考资料
* [Docker日志收集最佳实践](https://www.cnblogs.com/jingjulianyi/p/6637801.html)
* [Configure logging drivers](https://docs.docker.com/config/containers/logging/configure/)


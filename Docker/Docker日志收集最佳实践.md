# Docker日志收集最佳实践

在linux上，容器日志一般存放在**/var/lib/docker/containers/container_id/**下面

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

## 参考资料
* [Docker日志收集最佳实践](https://www.cnblogs.com/jingjulianyi/p/6637801.html)


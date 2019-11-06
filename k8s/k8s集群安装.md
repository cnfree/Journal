# k8s 集群安装

## 安装翻墙软件
https://blog.csdn.net/xyz_dream/article/details/88372356

## 环境预设
https://www.cnblogs.com/ding2016/p/10784620.html

## 安装Docker
https://blog.csdn.net/xyz_dream/article/details/88372356

## 安装k8s
https://blog.csdn.net/xyz_dream/article/details/88372356

## 预下载镜像
https://www.cnblogs.com/ding2016/p/10784620.html

## 修改docker cgroupdriver参数
https://www.cnblogs.com/ding2016/p/10784620.html

## 初始化集群
* https://www.cnblogs.com/ding2016/p/10784620.html
* **记录好控制台输出 kubeadm join 命令，包含join token**

## 安装网络驱动, flannel or Calico
* https://www.cnblogs.com/ding2016/p/10784620.html
* https://blog.csdn.net/hjxzb/article/details/82823191#master_36

## 加入集群
https://www.cnblogs.com/ding2016/p/10784620.html

## token过期重亲获取
* https://blog.csdn.net/weixin_44208042/article/details/90676155
* https://blog.csdn.net/mailjoin/article/details/79686934

## 查找 kubeadm 配置文件
* find / -name "kubelet.service.d"
* find / -name "10-kubeadm.conf"

## 重新初始化集群
* kubeadm reset
* https://www.cnblogs.com/able7/p/10216299.html

## 查看kubelet服务启动状态
* systemctl status kubelet.service -l

## 查看容器运行状态
* kubectl get pods --all-namespaces
* https://www.cnblogs.com/liangDream/p/7358847.html

## 异常解决
* https://www.jianshu.com/p/c92e46e193aa

## 安装dashboard
* https://blog.csdn.net/hjxzb/article/details/82823191#master_36
* https://www.cnblogs.com/RainingNight/p/deploying-k8s-dashboard-ui.html
* https://www.jianshu.com/p/21d916643560

## k8s dashboard 一键生成ssl证书脚本
* https://blog.csdn.net/weixin_43421352/article/details/86651062

## Dashboard使用自定义证书
* https://blog.csdn.net/chenleiking/article/details/81488028

## k8s dashboard 认证方式
* https://blog.csdn.net/it_zhouzhenfeng/article/details/89392711

## 删除K8s Namespace时卡在Terminating状态
* https://blog.csdn.net/weweeeeeeee/article/details/100036417
# 虚拟机Clone解决方案

1. 关闭需要Clone的虚拟机，选择菜单，Action --> Export，导出虚拟机镜像文件
2. 创建虚拟机，选择 Deploy a virtual machine from an OVF or OVA file, 编辑虚拟机名称，选择第一步导出的ovf文件和镜像文件，创建虚拟机
3. 关闭新创建的虚拟机，修改虚拟机网络适配器MAC地址，选择Manual手动模式，输入 00:50:56:xx:xx:xx 范围内的Mac地址
4. 重启虚拟机，输入【cd /etc/sysconfig/network-scripts/】命令后，再执行【ip addr】命令，查看网卡名称
5. 输入命令【vi ifcfg-网卡名称】打开网卡设置文件，编辑HWADDR，修改为第3步设置的Mac地址，修改IPADDR为新的IP地址
6. 重启网络 service network restart
7. 修改机器名称 hostnamectl set-hostname &lt;newhostname&gt;
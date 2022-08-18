# vrrpd 实现双主服务器

> keepalived 设置比较麻烦，代码也比较复杂，想自行定制太麻烦了。后来在 githup 找到 vrrpd ，这个开源软件实现比较简单的，配置也比较简单，如果想实现自己的功能，还可以自行修改代码进行编译。  

### 安装

```shell
# https://github.com/fredbcode/Vrrpd
cd Vrrpd-master
make clean
make
cp vrrpd /usr/sbin/
cp atropos /usr/sbin/
```



### 配置
我们用云服务器来实现两台机器间的心跳，为了节省成本，我们想两台云服务器只使用一个`IP`。使用`vrrp`协议，我们需使用内网，那两台云服务器我们需添加内网网卡。

```xml
<interface type='bridge'>
  <mac address='00:2b:56:d3:2a:a3'/>
  <source bridge='br2019'/>
  <virtualport type='openvswitch'/>
  <target dev='vm_5910_01_vm'/>
  <model type='virtio'/>
</interface>
```

两台云服务器的外网网卡我们随便设置一个内网`IP`，两台机器的外网`IP`是通过心跳来产生的。

**云服务器1:**
\# 配置公网网卡，使用内网地址

```shell
vi /etc/sysconfig/network-scripts/ifcfg-eth0
NAME=eth0
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.1.93
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
# 上面随便分配一个内网IP给公网网卡

# 当 vrrpd 切换为 master 时，执行的 shell
# vi Master.sh
ip link set eth0 up
ip addr del 192.168.1.93/24 dev eth0
ip route del default
ip route add default via 42.51.67.1
# 以上命令是将公网网卡启动、删除随便设置的内网IP、删除原路由、设置公网路由，总之一句话使公网IP可以正常对外访问。

# 当 vrrpd 切换为 backup 时，执行的 shell
# vi Backup.sh
ip addr del 42.51.67.93/24 dev eth0
ip addr add 192.168.1.93/24 dev eth0
ip link set eth0 address 00:0C:82:AD:B3:29
# 以上命令是将 公网IP删除、重新设置为内网IP、并将MAC地址设置为原MAC地址

# 当编辑好切换的脚本后，就可以通过命令行启动 vrrpd 的进程了
vrrpd -i eth0 42.51.67.93/24 -v 55 -n -U /root/Master.sh -D /root/Backup.sh

# 添加防火墙规则
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface eth0 --destination 224.0.0.18 --protocol vrrp -j ACCEPT;
firewall-cmd --reload;
```

**云服务器2：**

```shell
# 配置公网网卡，使用内网地址
# vi /etc/sysconfig/network-scripts/ifcfg-eth0
NAME=eth0
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.1.94
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
# 上面随便分配一个内网IP给公网网卡

# 当 vrrpd 切换为 master 时，执行的 shell
# vi Master.sh
ip link set eth0 up
ip addr del 192.168.1.94/24 dev eth0
ip route del default
ip route add default via 42.51.67.1
# 以上命令是将公网网卡启动、删除随便设置的内网IP、删除原路由、设置公网路由，总之一句话使公网IP可以正常对外访问。

# 当 vrrpd 切换为 backup 时，执行的 shell
# vi Backup.sh
ip addr del 42.51.67.93/24 dev eth0
ip addr add 192.168.1.94/24 dev eth0
ip link set eth0 address 00:0C:82:AD:B3:29
# 以上命令是将 公网IP删除、重新设置为内网IP、并将MAC地址设置为原MAC地址

# 当编辑好切换的脚本后，就可以通过命令行启动 vrrpd 的进程了
vrrpd -i eth0 42.51.67.93/24 -v 55 -n -U /root/Master.sh -D /root/Backup.sh
# 添加防火墙规则
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface eth0 --destination 224.0.0.18 --protocol vrrp -j ACCEPT;
firewall-cmd --reload;
```

注意: 云服务器1的随便设置IP为192.168.1.93，而云服务器2的为192.168.1.94。当某台服务器为master 时，vrrpd 会将该服务器的公网IP设置为 42.51.67.93，并设置为固定的 mac 地址（修改源码的mac 数值）。

<font color=red>**以上配置在两台不同母机的云服务器测试通过的了，但后来我现在，由于 eth0 是公网，arp 包是通过公网传播的。虽然可以用，但当公网带宽用满了，可能会导致 arp 包传播问题，导致心跳出现问题。**</font>

### 新方案

改为用内网广播，eth0 为公网，eth1为内网
**云服务器1：**

```shell
# 当 vrrpd 切换为 master 时，执行的 shell
# vi Master.sh
#!/bin/bash
ip link set eth0 up
ip addr del 192.168.1.93/24 dev eth0
ip route del default
ip addr add 42.51.67.93/24 dev eth0
ip route add default via 42.51.67.1
ip link set eth0 address 00:9F:A4:4C:3B:37
exit 0
# 当 vrrpd 切换为 backup 时，执行的 shell

# vi Backup.sh
#!/bin/bash
ip addr add 192.168.1.93/24 dev eth0
ip addr del 42.51.67.93/24 dev eth0
ip route del default
ip link set eth0 address 00:0C:82:AD:B3:29
exit 0

### vrrpd 运行在内网网卡 eth1 上，10.51.67.93 为虚拟IP，并没有实际作用的
vrrpd -i eth1 10.51.67.93/24 -v 55 -U /root/Master.sh -D /root/Backup.sh
```

**云服务器2：**

```shell
# 当 vrrpd 切换为 master 时，执行的 shell
# vi Master.sh
#!/bin/bash
ip link set eth0 up
ip addr del 192.168.1.94/24 dev eth0
ip route del default
ip addr add 42.51.67.93/24 dev eth0
ip route add default via 42.51.67.1
ip link set eth0 address 00:9F:A4:4C:3B:37
exit 0
# 当 vrrpd 切换为 backup 时，执行的 shell

# vi Backup.sh
#!/bin/bash
ip addr add 192.168.1.94/24 dev eth0
ip addr del 42.51.67.93/24 dev eth0
ip route del default
ip link set eth0 address 00:f1:57:3d:c3:40
exit 0

### vrrpd 运行在内网网卡 eth1 上，10.51.67.93 为虚拟IP，并没有实际作用的
vrrpd -i eth1 10.51.67.93/24 -v 55 -U /root/Master.sh -D /root/Backup.sh
```



#### 参考文章：
[VRRP协议的工作机制介绍，Keepalived内部架构及其实现原理解析](https://blog.csdn.net/Hello_World_QWP/article/details/104224876)
[入木三分学网络第一篇--VRRP协议详解第一篇（转）](https://www.cnblogs.com/ajianbeyourself/p/4150233.html)
[VRRPd介绍及测试](https://www.jianshu.com/p/a96e1032c2c8)  

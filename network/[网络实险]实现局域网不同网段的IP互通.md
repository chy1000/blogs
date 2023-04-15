# [网络实险] 实现局域网不同网段的IP互通

> 一直以来我学到的网络知识都告诉我，当在一个局域网内，不同段的IP是没办法 ping 通的。我所在的公司的办公室有两个网段：192.168.7.0 和 192.168.8.0 ，如果我在本机搭建了一个测试网站，想给另一个网段的同事浏览，他们是没办法打开网站的。我突发奇想，可不可能通过设置机器或者路由，实现不同段的两台机器实现互通？以下是模拟的实验

### 实验步骤

1. ##### 在 ubuntu 系统机器上安装 mininet 

   为什么是使用 ubuntu 系统的机器，而不是我熟悉的 centos 系统的机器呢。因为我们测试的过程中需要使用到 mininet ，这个软件在 ubuntu 环境下安装比较简单。

   ```shell
   # 安装 mininet
   rm /var/lib/dpkg/lock-frontend
   rm /var/lib/dpkg/lock
   rm /var/cache/apt/archives/lock
   apt install git -y
   git clone git://github.com/mininet/mininet
   cd ./mininet/util/
   ./install.sh -a
   ```

   

2. ##### mininet 创建简单拓扑进行测试

   mininet 是 SDN 学习中用来创建各种拓扑的仿真软件，能够使用最小的消耗完成主机，交换机，控制器的模拟。使用 <font color=green>mn</font> 命令创建两个主机连接到一个交换机中的拓扑。

   ![image-20220327115349339]([网络实险]实现局域网不同网段的IP互通.assets/image-20220327115349339-16564890556916.png)

   当我们执行上面的命令后，自动创建了一个 s1 的 ovs 网桥，并且创建了两个节点：h1、h2，这两个节点连接到 s1 上。

   ![image-20220327115844486]([网络实险]实现局域网不同网段的IP互通.assets/image-20220327115844486-16564890556929.png)

   接着我们给 h1 和 h2 分别配置两个不同段的IP，并进行 ping 的测试，可以很明显看出两个节点是不能互通的。

   ```shsh
   h1 ifconfig h1-eth0 192.1.1.2 netmask 255.255.255.0
   h2 ifconfig h2-eth0 172.1.1.2 netmask 255.255.255.0
   h1 ping 172.1.1.2
   # Network is unreachable
   h1 route -n
   # Kernel IP routing table
   # Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   # 192.1.1.0       0.0.0.0         255.255.255.0   U     0      0        0 h1-eth0
   ```

   为什么不能互通呢？因为不同段的IP访问时，是用到IP协议，需要使用到网关。但很明显我们上面两个节点并没有设置到网关。

   <font color=orange>如果这时我创建一条指向 172.1.1.0 的路由，是否就可以实现将数据转发出去了？</font>

   ```shell
   # onlink 参数表明强制此网关是“在链路上”的(虽然并没有链路层路由),如果不加这个参数下面这条路由由于协议限制不同段的网关是添加不了的
   h1 ip route add 172.1.1.0/255.255.255.0 via 172.1.1.2 dev h1-eth0 onlink
   h1 route -n
   # Kernel IP routing table
   # Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   # 172.1.1.0       172.1.1.2       255.255.255.0   UG    0      0        0 h1-eth0
   # 192.1.1.0       0.0.0.0         255.255.255.0   U     0      0        0 h1-eth0
   ```

   从上面的命令可知，我们手动创建了一条 172.1.1.0 段的路由，并将网关设置为 172.1.1.2。设置为 172.1.1.2 ，是为了让以太网协议找到它的 MAC 的。这样所有 172.1.1.0 段的包都转发到 172.1.1.2。

   

3. ##### 使用 wireshark 抓包数据

   ![image-20220327222553013]([网络实险]实现局域网不同网段的IP互通.assets/image-20220327222553013-16564890556928.png)

   ![image-20220327222841042]([网络实险]实现局域网不同网段的IP互通.assets/image-20220327222841042-16564890556927.png)

   添加路由后，我们在 h1 上 ping h2，再打开 wireshark 抓包数据，从上图可以发现可以接收到 ping 请求的包了。但我们发现没有返回，原来我们只是弄通了单向，要双向弄通才会有返回的。继续设置：

   ```shell
   h2 ip route add 192.1.1.0/255.255.255.0 via 192.1.1.2 dev h2-eth0 onlink
   h2 route -n
   # Kernel IP routing table
   # Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   # 172.1.1.0       0.0.0.0         255.255.255.0   U     0      0        0 h2-eth0
   # 192.1.1.0       192.1.1.2       255.255.255.0   UG    0      0        0 h2-eth0
   ```

   这时我们再 ping ，终于有返回结果了。

   ![image-20220328094155237]([网络实险]实现局域网不同网段的IP互通.assets/image-20220328094155237-165648905569210.png)
   
   

------

## 实现不同网桥互通

使用`brctl`命令创建`br10`和`br11`

```shell
brctl addbr br10;
brctl addbr br11;
```

再创建`ns10`和`ns11`

```shell
ip netns add ns10;
ip link add ns10-veth0 type veth peer name ns10-veth1;
ip link set ns10-veth1 netns ns10;
ip netns exec ns10 ip addr add 172.16.18.101/24 dev ns10-veth1;
ip netns exec ns10 ip link set dev ns10-veth1 up;
ip netns exec ns10 ip route add default via 172.16.18.1;
brctl addif br10 ns10-veth0;
ip link set dev ns10-veth0 up;
ip link set br10 up;

ip netns add ns11;
ip link add ns11-veth0 type veth peer name ns11-veth1;
ip link set ns11-veth1 netns ns11;
ip netns exec ns11 ip addr add 192.168.10.101/24 dev ns11-veth1;
ip netns exec ns11 ip link set dev ns11-veth1 up;
ip netns exec ns11 ip route add default via 192.168.10.1;
brctl addif br11 ns11-veth0;
ip link set dev ns11-veth0 up;
ip link set br11 up;
```

在这两个`ns`执行`ping`，可以看到`ns`里的`ip`是不能互访的，但可以访问`br`的`ip`

```shell
ip netns exec ns10 ping 192.168.10.101
PING 192.168.10.101 (192.168.10.101) 56(84) bytes of data.
# 没返回
ip netns exec ns11 ping 172.16.18.101
PING 192.168.10.101 (192.168.10.101) 56(84) bytes of data.
# 没返回

ip netns exec ns10 ping 192.168.10.1
PING 192.168.10.1 (192.168.10.1) 56(84) bytes of data.
64 bytes from 192.168.10.1: icmp_seq=7567 ttl=64 time=0.057 ms
64 bytes from 192.168.10.1: icmp_seq=7568 ttl=64 time=0.057 ms
...
ip netns exec ns11 ping 172.16.18.1
PING 172.16.18.1 (172.16.18.1) 56(84) bytes of data.
64 bytes from 172.16.18.1: icmp_seq=1 ttl=64 time=0.129 ms
64 bytes from 172.16.18.1: icmp_seq=2 ttl=64 time=0.058 ms
...
```

> 这里的`FORWARD`我简单说一下，当`Linux`收到了一个 目的`ip`地址不是本地的包 ，`Linux`会把这个包丢弃，因为默认情况下，`Linux`的三层包转发功能是关闭的，如果要让我们的`Linux`实现转发，则需要打开这个转发功能，可以 改变它的一个系统参数，使用`sysctl net.ipv4.ip_forward=1`或者`echo 1 > /proc/sys/net/ipv4/ip_forward`命令打开转发功能。
> 好了，在这里我们让`Linux`允许转发，这个包的目的地址也不是本机，那么它将接着走入`FORWARD`链，在`FORWARD`链里面，我们就可以定义详细的规则，也就是是否允许它通过，或者对这个包的方向流程进行一些改变，这也是我们实现访问控制的地方。

`echo 1 > /proc/sys/net/ipv4/ip_forward` 开启允许转发后，可以看到有返回了，实现了不同网段的互通。

```shell
ip netns exec ns10 ping 192.168.10.101
PING 192.168.10.101 (192.168.10.101) 56(84) bytes of data.
64 bytes from 192.168.10.101: icmp_seq=1 ttl=63 time=0.150 ms
64 bytes from 192.168.10.101: icmp_seq=2 ttl=63 time=0.078 ms
...
ip netns exec ns11 ping 172.16.18.101
PING 172.16.18.101 (172.16.18.101) 56(84) bytes of data.
64 bytes from 172.16.18.101: icmp_seq=1 ttl=64 time=0.105 ms
64 bytes from 172.16.18.101: icmp_seq=2 ttl=64 time=0.058 ms
...
```

但这里还是有个疑问，未开启转发前，为什么可以`ping`通`br`本身的`ip`呢？
我认为是这样的，`br`本身的`ip`是属于本地的包，所以即使没开启转发，`Linux`也不会把这个包丢弃，并且会广播到其它的`br`上。而`ns`里的`ip`并不属于宿主机本地的包，所以未开启转发前，会把这个包丢弃。

-----

## 实现OVS的不同网桥互通

```
ovs-vsctl add-br br111;
ip netns add ns111;
ip link add ns111-veth0 type veth peer name ns111-veth1;
ip link set ns111-veth1 netns ns111;
ip netns exec ns111 ip addr add 10.1.1.101/24 dev ns111-veth1;
ip netns exec ns111 ip link set dev ns111-veth1 up;
ip netns exec ns111 ip route add default via 10.1.1.1;
ovs-vsctl add-port br111 ns111-veth0;
ip link set dev ns111-veth0 up;
ip addr add 10.1.1.1/24 dev br111;
ip link set br111 up;

ovs-vsctl add-br br222;
ip netns add ns222;
ip link add ns222-veth0 type veth peer name ns222-veth1;
ip link set ns222-veth1 netns ns222;
ip netns exec ns222 ip addr add 172.1.1.101/24 dev ns222-veth1;
ip netns exec ns222 ip link set dev ns222-veth1 up;
ip netns exec ns222 ip route add default via 172.1.1.1;
ovs-vsctl add-port br222 ns222-veth0;
ip link set dev ns222-veth0 up;
ip addr add 172.1.1.1/24 dev br222;
ip link set br222 up;
```

和上面“不同网桥实现互通”唯一不同的是，我把网桥换成了`ovs`，连接`ovs`和`ns`的还是`veth`，开启转发后，同样可以实现互通。

当不使用veth连接ovs和ns，而是使用ovs自身的port连接时，发现不一样的结果，不同网桥间并不能互通，下面是测试的例子：

```shell
ovs-vsctl add-br br111;
# 通过 internal port 可以实现容器直接连接到OVS（测试了下，其它类型的端口好像不行）
# 通过内部端口连接到 OVS 比通过 veth-pair实现更好的性能（https://arthurchiao.art/blog/ovs-deep-dive-6-internal-port/）
ovs-vsctl add-port br111 tap1 -- set Interface tap1 type=internal;
ip netns add ns111;
ip link set tap1 netns ns111;
ip netns exec ns111 ip addr add 10.1.1.101/24 dev tap1;
ip netns exec ns111 ip link set tap1 up;
ip netns exec ns111 ip link set lo up;
ip addr add 10.1.1.1/24 dev br111;
ip link set br111 up;

ovs-vsctl add-br br222;
ovs-vsctl add-port br222 tap2 -- set Interface tap2 type=internal;
ip netns add ns222;
ip link set tap2 netns ns222;
ip netns exec ns222 ip addr add 172.2.1.101/24 dev tap2;
ip netns exec ns222 ip link set tap2 up;
ip netns exec ns222 ip link set lo up;
ip addr add 172.2.1.1/24 dev br222;
ip link set br222 up;

ip netns exec ns111 ping 172.2.1.101
connect: Network is unreachable
ip netns exec ns111 ping 172.2.1.1
connect: Network is unreachable
```

暂时想不明白上面的情况为什么不能互通？如果要实现互通，可以添加`patch`实现。

```shell
ovs-vsctl add-port br111 patch-ovs-1 -- set Interface patch-ovs-1 type=patch options:peer=patch-ovs-2;
ovs-vsctl add-port br222 patch-ovs-2 -- set Interface patch-ovs-2 type=patch options:peer=patch-ovs-1;
```



----------

### 怎样安装  wireshark ？

上面安装了 mininet 是自动群安装 wireshark 的。如果我们在 centos 系统想安装 wireshark，可以按照下面的方法安装：

```shell
yum install -y wireshark wireshark-gnome
```



---

### 参考：

[openvswitch 实践一 创建patch port连接ovs两个桥](https://blog.csdn.net/u013743253/article/details/119601088)


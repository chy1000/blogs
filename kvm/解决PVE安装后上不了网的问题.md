# 解决PVE安装后上不了网的问题

我想体验一把PVE，刚好公司有一台架下闲置的母机可以拿来测试。这台机器是 Centos7 系统的，我重装为 PVE 了，却发现网络不通。/etc/network/interfaces 配置如下：

```shell
auto lo
iface lo inet loopback
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
        address 59.xx.xxx.74/25
        gateway 59.xx.xxx.1
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0

iface eno2 inet manual
iface eno3 inet manual
iface eno4 inet manual
```

从上面的配置文件我们可知，PVE 自动帮我们创建了一个网桥 vmbr0 ，物理网卡 eno1 连接到网桥，通过桥接的方式实现上网。这台母机原本是正常使用的，明显IP是没有问题的，上面的配置也问题，那是不是网桥的问题？去掉网桥测试下

```shell
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual
        address 59.xx.xxx.74/25
        gateway 59.xx.xxx.1

iface eno2 inet manual
iface eno3 inet manual
iface eno4 inet manual
```

去掉了网桥网络还是不通，上网查了相关的资源，反复确认了配置文件没错，那问题到底出在哪里呢？

后来想起原来的母机是有做 vlan 的，会不会是没配置 vlan 的问题？于是加上 vlan 的配置

```shell
auto lo
iface lo inet loopback
iface eno1 inet manual

auto eno1.33
iface eno1.33 inet manual
        address 59.xx.xxx.74/25
        gateway 59.34.xxx.1

iface eno2 inet manual
iface eno3 inet manual
iface eno4 inet manual
```

配置好后重启网络，发现网络终于通了。但我有点想不通的是，按照我一贯的理解，trunk端口只是用于交换机之间的连接的，我这里直接配置网卡，怎么也需要配置 vlan 号？

> Access类型的端口只能属于1个VLAN，一般用于连接计算机的端口；
> Trunk类型的端口可以允许多个VLAN通过，可以接收和发送多个VLAN的报文，一般用于交换机之间连接的端口；
> (https://www.cnblogs.com/sddai/p/8941757.html)

后来问了网管，原来是我之前的的理解有错了，当交换机开通了 trunk 端口，机器内的网卡也是需要配置 vlan 的。

> Access mode：(untag port)(一般電腦、設備使用)
> 1、收到一個封包。
> 2、判斷是否有VLAN tag；如果沒有則轉到第3步，有則轉到第4步。
> 3、打上Port的PVID，並進行交換轉發。
> 4、直接丟棄。
>
> Trunk mode：(tag port)(Switch交換使用)
> 1、收到一個封包。
> 2、判斷是否有VLAN tag；如果沒有則轉到第3步，有則轉到第4步。
> 3、打上Port的PVID，並進行交換轉發。
> 4、判斷該trunk Port是否允許該VLAN的封包進入；如果可以則轉發，否則丟棄。

<font color=red>从上面我们得知，当从机器的包进入交换机时，交换机是要判断 vlan 才进行转发的，所以从机器发送过来的包必须有 vlan 标记才能正常转发。</font>

我在那台机器上做了一个小实验，看看加上了 vlan ，包的数据格式是怎样的。

```shell
# 新开一台终端，ping 59.xx.xx.76
ping 59.xx.xxx.76

# 在那台机器使用 tcpdump 抓包数据
tcpdump -ni eno1 -v -e src host 59.xx.xxx.76
# 抓包输出的数据如下
tcpdump: listening on eno1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:07:08.010095 e0:xx:xx:xx:xx:80 > ec:xx:xx:xx:xx:60, ethertype 802.1Q (0x8100), length 102: vlan 33, p 0, ethertype IPv4 (0x0800), (tos 0x0, ttl 64, id 26651, offset 0, flags [none], proto ICMP (1), length 84)
```

我们从输出看到，包数据成功打上了 vlan 33 的标记。

最后，恢复网桥设置，再重新配置 vlan ，新开通的虚拟机也能正常上网了，问题解决。

```shell
auto lo
iface lo inet loopback
iface eno1 inet manual
iface eno1.33 inet manual

auto vmbr0
iface vmbr0 inet manual
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0

auto vmbr0v33
iface vmbr0v33 inet static
        address 59.xx.xxx.74/25
        gateway 59.xx.xxx.1
        bridge-ports eno1.33
        bridge-stp off
        bridge-fd 0

iface eno2 inet manual
iface eno3 inet manual
iface eno4 inet manual
```


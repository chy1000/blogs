# Ceph安装使用

### 安装

配置`yum`源

```shell
# 清空原来自带配置文件：
cd /etc/yum.repos.d/
mkdir /tmp/yum
mv * /tmp/yum/

curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
yum install wget -y
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum -y install yum-plugin-priorities.noarch

#配置阿里的 ceph 源：
cat << EOF | tee /etc/yum.repos.d/ceph.repo
[ceph]
name=Ceph packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/x86_64/
gpgcheck=0
priority=1

[ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/noarch/
gpgcheck=0
priority=1

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/SRPMS
gpgcheck=0
priority=1
EOF
```

关闭防火墙 & SELinux：

> 这里只是为了安装的简单，生产环境还是把防火墙打开吧，只开放使用到的端 口。

```shell
systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld
# 关闭 SELinux
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
```

所有节点配置主机名称：

```shell
hostnamectl --static set-hostname node1
hostnamectl --static set-hostname node2
hostnamectl --static set-hostname node3
```

所有节点配置hosts文件：

```shell
/etc/hosts
192.168.20.5    node1
192.168.30.5    node2
192.168.40.5    node3
```

配置`ntp`、`ntpdate`时间同步：

```shell
# node1
yum -y install ntp
# 注释原来的 server
vi /etc/ntp.conf
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server 127.127.1.0 # local clock
fudge  127.127.1.0 stratum 10

# node2、node3
yum -y install ntp
vi /etc/ntp.conf  # 注释server开头的前面四行
server node1
# node1、node2、node3
systemctl restart ntpd
systemctl enable ntpd
systemctl status ntpd
# 查看是否正确
ntpq -p
# 参考：https://blog.51cto.com/u_14249697/5040637 和 https://www.modb.pro/db/58827
```

配置`node1`免密访问其它节点：

```shell
ssh-keygen -t rsa
ssh-copy-id node2
ssh-copy-id node3
```

安装`ceph`软件：

```shell
# 在 node1 节点额外安装ceph-deploy
yum -y install ceph-deploy
# 使用 ceph-deploy 安装 ceph
ceph-deploy install node1 node2 node3 --release nautilus
# 部署MON节点，创建目录生成配置文件
mkdir cluster
cd cluster
ceph-deploy new ceph1 ceph2 ceph3
#ceph-deploy new --cluster-network 10.254.100.0/24 --public-network 10.254.100.0/24 node1 
# 初始化密钥
ceph-deploy mon create-initial
# 同步集群文件到所有节点
ceph-deploy --overwrite-conf admin node{1,2,3}
# 部署MGR节点
ceph-deploy mgr create node{1,2,3}
```

部署OSD节点：

>  部署osd节点时要注意，需要将使用到的磁盘取消挂载。如果取消挂载执行下面命令还是出错，最好使用 fdisk 命令装磁盘删除分区再操作。

```shell
ceph-deploy osd create --data /dev/vdb node1
```

创建成功后，查看是否正常：

```shell
ceph -s
```



### 使用 RDB

**创建`ceph`存储池**

```shell
ceph osd pool create <pool_name> <pg_num> [<pgp_num>]
-- <pool_name> 是要创建的存储池的名称
-- <pg_num> 是该存储池要使用的 PG 数量
-- <pgp_num> 是可选参数，用于指定主从 PG 数量（Primary Placement Group 数量），默认值等于 PG 数量
```

需要注意的是，在创建存储池之前，您需要先确保 Ceph 集群中已经有足够的 OSD 节点来存储存储池中的数据，否则存储池的数据无法得到正确的分布和备份。另外，PG 数量的大小是非常重要的，过少的 PG 数量可能导致数据热点和性能问题，而过多的 PG 数量则会导致管理复杂度的增加和存储空间的浪费，因此需要根据实际情况合理设置 PG 数量。

**创建`RBD`镜像**

```shell
rbd create <rdb_name> --size 20G --pool <pool_name> --image-format 2 --image-feature layering
--image-format 2: 指定 RBD 镜像的格式为 2，该格式支持特性比较全面，包括快照、克隆、分层等。
--image-feature layering: 指定 RBD 镜像支持分层功能，这意味着该镜像可以作为其他 RBD 镜像的父镜像，通过共享父镜像的数据来节省存储空间。
rbd ls --pool <pool_name>
rbd info <pool_name>/<rdb_name>
```

**挂载`RBD`磁盘使用**

使用 RBD 磁盘需要将其映射到本地主机上，然后像使用本地块设备一样来操作它。

```shell
[root@node1 ~]# rbd map ceph_pool/image01
/dev/rbd0
[root@node1 ~]# mkfs.ext4 /dev/rbd0
[root@node1 ~]# mount /dev/rbd0 /home
```

操作完成后，需要卸载 RBD 磁盘映射设备并取消映射，使用以下命令：

```shell
[root@node1 ~]# umount /dev/rbd0
[root@node1 ~]# rbd unmap /dev/rbd0
```

需要注意的是，每次操作 RBD 磁盘前，都需要先映射 RBD 磁盘到本地主机上，操作完成后再取消映射，否则会导致映射设备被占用而无法操作。此外，RBD 磁盘支持快照、克隆等高级功能，您可以使用 rbd snap、rbd clone 等命令来管理 RBD 磁盘的快照和克隆。

**速度测试**

```shell
[root@node1 ~]# dd if=/dev/zero of=/home/testfile bs=1M count=5210
5210+0 records in
5210+0 records out
5463080960 bytes (5.5 GB) copied, 54.1763 s, 101 MB/s
```


---------

### RBD 磁盘扩容

```shell
[root@node1 ~]# umount /home
[root@node1 ~]# rbd unmap ceph_pool/image01
# 执行 rbd resize 命令扩容
[root@node1 ~]# rbd resize --pool ceph_pool --image image01 --size 20G
[root@node1 ~]# e2fsck -f /dev/rbd0
[root@node1 ~]# resize2fs /dev/rbd0
[root@node1 ~]# rbd map ceph_pool/image01
[root@node1 ~]# mount /dev/rbd0 /home
```

--------

### OSD状态为down，采用以下步骤重新格式化硬盘并将其加入ceph集群中

```shell
# 步骤1：停止相应OSD服务
systemctl stop ceph-osd@1.service
# 步骤2：取消OSD挂载（安装OSD时，会将osd.1挂载至/var/lib/ceph/osd/ceph-1，因此，删除OSD时，需要首先取消OSD挂载）
umount /var/lib/ceph/osd/ceph-1
# 步骤3：设置OSD为OUT
ceph osd out osd.1
# 步骤4：删除OSD
ceph osd crush remove osd.1 #如果未配置Crush Map则不需要执行这一行命令
ceph auth del osd.1
ceph osd rm 1
```

----

### 为了使 Ceph 集群正常运行，以下端口必须在防火墙上打开：

Ceph Monitor (MON) 使用的 6789 端口
Ceph OSD 使用的 6800-7300 端口范围
Ceph Metadata Server (MDS) 使用的 6800 端口
您可以使用防火墙软件配置这些规则，以便 Ceph 服务可以在必要时使用网络端口。在配置防火墙规则时，建议仅开放必要的端口，并根据需要限制访问源。

如果您使用的是 CentOS/RHEL 系统，可以使用 firewalld 管理防火墙，您可以使用以下命令打开上述端口的流量：

```shell
firewall-cmd --add-port=6789/tcp --permanent
firewall-cmd --add-port=6800-7300/tcp --permanent
firewall-cmd --add-port=6800/tcp --permanent
firewall-cmd --reload
```

如果您使用的是 Ubuntu/Debian 系统，可以使用 ufw 管理防火墙，您可以使用以下命令打开上述端口的流量：

```shell
ufw allow 6789/tcp
ufw allow 6800:7300/tcp
ufw allow 6800/tcp
ufw reload
```

----

### KVM的XML配置使用Ceph存储

```xml
<disk type='network' device='disk'>
  <driver name='qemu' type='rbd'/>
  <source protocol='rbd' name='pool/image'>
    <host name='ceph-node'/>
    <host name='ceph-node'/>
  </source>
  <target dev='vda' bus='virtio'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
</disk>
```

----

### 参考：

[centos 7.7 安装ceph](https://developer.aliyun.com/article/761298)

[Ceph实战教程(一)让Ceph集群运行起来](https://blog.51cto.com/happylab/2474943)

[openstack 后端存储构建ceph集群为什么不建议和raid搭配使用？](https://www.talkwithtrend.com/Question/429929)

[配置 Ceph 内外网分离](https://www.jianshu.com/p/42ab1f6dc6de)

[【ceph相关】osd异常问题处理（lvm信息丢失）](https://blog.csdn.net/Micha_Lu/article/details/125563951)

[ceph 集群空间使用情况](https://bean-li.github.io/ceph-space/)

[安装Ceph集群](https://www.cnblogs.com/pengpengboshi/p/13324612.html)

[CentOS7 搭建ceph集群](https://blog.csdn.net/weixin_46051525/article/details/125122252)

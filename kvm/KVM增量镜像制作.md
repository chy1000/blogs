# KVM增量镜像制作

### Centos 镜像制作
```shell
### 创建派生镜像
qemu-img create -f qcow2 -o backing_file=centos6.5_32_v1.qcow2 /data/test/Images/centos6.5_32_2021_v1.qcow2

### 修改 xml 将系统盘的地址改为 /data/test/Images/centos6.5_32_2021_v1.qcow2
virsh edit qyi-00000000000-8065

### 修改密码为 qy123456
run python3 /home/qycloud/test3.py

### 挂载到本地修改网卡配置文件
run guestmount -a /data/test/Images/centos6.5_32_2021_v1.qcow2 -m /dev/VolGroup/lv_root /mnt1
vi /mnt1/etc/sysconfig/network-scripts/ifcfg-eth0

### 取消挂载
run guestunmount /mnt1

### 启动机器
virsh start qyi-00000000000-8065

### 取消 iptables ip6tables postfix 等服务
chkconfig iptables off
chkconfig ip6tables off
chkconfig postfix off

### 安装 qemu-guest-agent
yum install qemu-guest-agent -y

### 设置为中国时区
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

### 关闭 selinux
vi /etc/sysconfig/selinux
# 修改为 SELINUX=disabled

### 配置 sshd
vi /etc/ssh/sshd_config
# 添加如下两条参数：
UseDNS no
AddressFamily inet

### 修改内核配置文件
vi /etc/sysctl.conf
# 添加如下参数：
vm.swappiness = 0
net.ipv4.neigh.default.gc_stale_time=120
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce=2
net.ipv4.conf.all.arp_announce=2
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2

### 安装时间同步
yum install -y ntp ntpdate
# 设置为开机启动
chkconfig ntpd on # centos5、6
systemctl able ntpd # centos7、8

### 为确保时间同步正确，再安排另一种时间同步
yum -y install rdate
# crontab -e 增加如下定时任务
*/20 * * * * /usr/bin/rdate -s time.nist.gov > /dev/null 2>&1

### 编辑开机启动文件
vi /etc/rc.local
# 添加如下参数：
>/etc/udev/rules.d/70-persistent-net.rules
/usr/bin/rdate -s time.nist.gov

### 关闭并清空 root 邮箱提示
echo "unset MAILCHECK">> /etc/profile
source /etc/profile

### 清空历史和日志记录
cd /var/log;
>btmp ;>dmesg;>messages ;>lastlog;>secure ;>wtmp;
rm -f /etc/udev/rules.d/70-persistent-net.rules;
cat /dev/null > /var/spool/mail/root
history -c && history -w && poweroff;

### 使用 virt-sysprep 清理镜像（该命令可清理MAC和操作历史等一系列操作）
virt-sysprep -a centos6.5_32_2021_v1.qcow2

### 最后压缩镜像，减少文件大小，方便传输
qemu-img convert -c -B centos6.5_32_v1.qcow2 -O qcow2 centos6.5_32_2021_v1.qcow2 centos6.5_32_2021_v1_compress.qcow2
```



##### centos5.8 镜像升级 openssl 

```shell
mkdir /root/src/
cd /root/src

### 安装 openssl
wget https://www.openssl.org/source/old/1.0.2/openssl-1.0.2r.tar.gz --no-check-certificate
tar zxvf openssl-1.0.2r.tar.gz
cd openssl-1.0.2r
./config shared zlib-dynamic
make
make install
/usr/local/ssl/bin/openssl version -a
mv /usr/bin/openssl /usr/bin/openssl.old
mv /usr/include/openssl /usr/include/openssl.old
ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl
ln -s /usr/local/ssl/include/openssl/ /usr/include/openssl
echo "/usr/local/ssl/lib" >> /etc/ld.so.conf
ldconfig
openssl version -a

### 编译安装 curl
cd /root/src
wget -c ftp://121.10.143.221:21/Other/curl-7.42.1.tar.gz --ftp-user=qycloud --ftp-password=1q5urBB646;
tar -xzvf curl-*.tar.gz
cd curl-*
./configure --with-ssl=/usr/local/ssl --disable-ldap 
make
make install
curl -V

### 编译安装 wget
cd /root/src
wget -c ftp://121.10.143.221:21/Other/wget-1.14.tar.gz --ftp-user=qycloud --ftp-password=1q5urBB646;
tar -xzvf wget-*
cd wget-1.20.3
./configure --prefix=/usr --sysconfdir=/etc --with-ssl=openssl --with-libssl-prefix=/usr/local/ssl
make
make install
wget -V
```

参考：https://miteshshah.github.io/linux/centos/how-to-enable-openssl-1-0-2-a-tlsv1-1-and-tlsv1-2-on-centos-5-and-rhel5/



### windows 镜像制作

```shell
### 创建派生镜像
qemu-img create -f qcow2 -F qcow2 -o backing_file=windows2008_r2_sp1_data_x64_2020_v1.qcow2 /data/test/Images/windows2008_r2_sp1_data_x64_2021_v1.qcow2

### 修改 xml 将系统盘的地址改为 windows2008_r2_sp1_data_x64_2021_v1.qcow2
virsh edit qyi-00000000000-8065

# 如果不需要网络连接，请在 xml 文件中的下面内容删除
<interface type='user'>
  <mac address='00:16:3e:08:63:a5'/>
  <model type='rtl8139'/>
</interface>

### 挂载到本地
run guestmount -a /data/test/Images/windows2008_r2_sp1_data_x64_2021_v1.qcow2 -m /dev/sda2 /mnt1

### 清除更新的缓存 C:\Windows\SoftwareDistribution\

### 取消挂载
run guestunmount /mnt1

### 启动机器
virsh start qyi-00000000000-8065

### 启动机器后，在这里进行更新操作

### 压缩文件，注意这里加上 -B 参数，把派生镜像重新指向 windows2008_r2_sp1_data_x64_2020_v1
qemu-img convert -c -B windows2008_r2_sp1_data_x64_2020_v1.qcow2 -O qcow2 windows2008_r2_sp1_data_x64_2021_v1.qcow2 windows2008_r2_sp1_data_x64_2021_v1_compress.qcow2

### 设置IP
10.0.2.102
255.255.255.0
10.0.2.1
10,20,30,40

```



### ubuntu 跟 centos 的差别

```shell
### ubuntu 与 centos 还是有比较多的地方不一样的

### dns 是由 systemd-resolved 服务托管的，所以无论你怎样改 /etc/resolv.conf 这个文件，还是会被重置的。
### 可以停用 systemd-resolved ，停用方法如下：
systemctl disable systemd-resolved.service
systemctl stop systemd-resolved.service
### systemd-resolved 带来的好处，一个是统一了dns的管理；另外一个就是可以通过本地cache加速dns查询

### ubuntu 系统的防火墙不是使用 iptables 和 firewalld ，而是使用 ufw
### 查看防火墙状态
ufw status

### 查找软件包
apt-cache search package
apt-get install package
```

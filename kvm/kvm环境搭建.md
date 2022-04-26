# kvm环境搭建

##### 1. 检查CPU是否已开启虚拟化

   ```shell
   cat /etc/redhat-release
   egrep 'vmx|svm' /proc/cpuinfo
   dmesg |grep kvm
   ```

##### 2. 安装 qemu-kvm libvirt libvirt-devel virt-install bridge-utils virt-viewer

   ```shell
   #qemu-kvm用来创建虚拟机硬盘,libvirt用来管理虚拟机
   yum install -y qemu-kvm libvirt libvirt-devel virt-install bridge-utils 
   
   #如果这台母机还要安装制作镜像，则安装 virt-viewer
   yum install -y virt-viewer
   
   #一健安装所有虚拟化的软件
   yum -y group install virtualization-client
   ```

##### 3. 查看 libvirtd 服务是否已启动

```shell
systemctl status libvirtd.service
   
# 启动 libvirtd，并将它设为开机启动

systemctl enable libvirtd.service

systemctl start libvirtd.service
```

##### 4. 安装 python3 的环境

```shell
yum install openssl-devel -y 

wget https://www.python.org/ftp/python/3.4.0/Python-3.4.0.tgz

tar -xvzf Python-3.4.0.tgz
   
cd Python-3.4.0

./configure; make; make install
   
# ....

# 如果提示没有编译器，则 yum install gcc -y 安装
```

##### 5. 安装 libguestfs

```shell
axel -n 10 http://download.libguestfs.org/1.38-stable/libguestfs-1.38.0.tar.gz

tar -xvzf libguestfs-1.38.0.tar.gz

mv libguestfs-1.38.0 Libguestfs

cd Libguestfs

yum -y install yum-utils

yum-builddep -y libguestfs

yum -y install ocaml-hivex ocaml-hivex-devel gperf genisoimage flex bison ncurses-devel pcre-devel augeas augeas-devel file file-devel yajl yajl-devel hivex-devel qemu-kvm ocaml-findlib ocaml-findlib-devel supermin5

yum -y install xz

ln -s /usr/bin/supermin5 /usr/bin/supermin

wget ftp://ftp.pbone.net/mirror/ftp5.gwdg.de/pub/opensuse/repositories/home:/presbrey:/librdf/RedHat_RHEL-6/x86_64/yajl-2.0.1-11.4.x86_64.rpm

PYTHON=/usr/local/bin/python3 ./configure

# 如果安装完 libguestfs ，挂载不了 ntfs 的文件，需要安装 ntfs-3g 。千万不要源码安装，源码安装有问题

yum install -y epel-release
yum install -y ntfs-3g
yum install -y ntfs-3g-devel
yum install -y ntfsprogs # 记得安装这个，要不会出问题...

ln -s /home/qycloud/Libguestfs/run /usr/bin/run

# 测试是否可以挂载 windows 系统文件
run virt-resize --machine-readable

# 测试centos系统是否可以正常挂载
run virt-filesystems -a /home/qycloud/Images/centos7_X86_official.img -l
# 如果没有 /dev/centos/root
yum install -y libguestfs-tools

https://download.fedoraproject.org/pub/epel/7/SRPMS/Packages/n/ntfs-3g-2017.3.23-6.el7.src.rpm
```

##### 6. 安装 libvirt-python

```shell
pip3 install libvirt-python

pip3 install pycrypto

pip3 install redis

pip3 install requests

# 合并为一条命令：

pip3 install libvirt-python pycrypto redis requests
```


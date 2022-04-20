# 升级 qemu-kvm

> 如果版本太旧，虚拟机会有一些新的特性或者功能使用不了，所以我们要升级 qemu-kvm 。这里我们在 centos7 系统上使用 yum 进行安装，使用 yum 安装只能安装较为新的版本，而这个较为新的版本也已经是多年前的版本了....为什么我们不使用编译的方式来安装最新的版本呢？曾经使用过编译的方式来安装新的版本，但使用的过程中出现很多奇怪的问题，使用很不稳定。所以在这里我们只使用 yum 的方式来安装。以后有空再研究编译的安装方式。



##### 先升级内核的版本

```shell
uname -r
yum update curl
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
# 安装稳定的版本
yum --enablerepo=elrepo-kernel install kernel-lt
# 设置开机启动顺序
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
```

##### 升级 qemu-kvm

```shell
yum remove qemu-kvm-ev
yum install centos-release-qemu-ev -y
yum install qemu-kvm-ev -y
qemu-img -V
```

##### 删除旧内核

```shell
rpm -qa | grep kernel
yum remove kernel-xxx.el7.x86_64
yum remove kernel-xxx.el7.x86_64
yum remove kernel-xxx.el7.x86_64
```

到这里升级已经完成，并不需要重启机器就生效了的。

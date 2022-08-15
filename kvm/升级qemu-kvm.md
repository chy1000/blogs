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
# 在用的机器注意不要重启
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

---

##### 启动新内核报错：VFS:Unable to mount root fs on Unknown-block(0,0)

按 《[Kernal Panic - Not syncing : VFS: unable to mount root fs on unknown-block (0,0)](https://forums.centos.org/viewtopic.php?f=20&t=22425&sid=473f49b0d5b47b49fd0add12871142a5)》文章，应该是安装时未完整，导致进入不了内核，只需`yum remove`删除内核，再更新`yum update`后重新安装内核就行。

---


>  附：《[archlinux QEMU 文档](https://wiki.archlinux.org/title/QEMU#By_specifying_kernel_and_initrd_manually)》- 文档值得细看

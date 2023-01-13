# kvm遇到的问题整理

[TOC]



### 1. 怎样更改网络模式实现NAT上网？

将`xml`的网络模式更改为`user`模式后启动虚拟机，`dhcp`会获取一个 `10.0.2.xx`的内部IP，实现`NAT`上网。如果`dhcp`不能正确获取内部IP，可手动设置为 `10.0.2.xx/24`网关设置为`10.0.2.2`。

```xml
    <interface type='user'>
      <mac address='00:4A:DB:C3:61:CD'/>
      <model type='rtl8139'/>
    </interface>
```

如果还想在外面远程连接到内部的虚拟机，可配置`qemu:commandline`将虚拟机的3389的端口影射为宿主机的端口，从而实现外部远程访问。

```xml
  <qemu:commandline>
     <qemu:arg value='-redir'/>
     <qemu:arg value='tcp:55947::3389'/>
  </qemu:commandline>
```



### 2. 怎样在xml添加cdrom加载ISO?

```

```



### 3. guestmount 挂载 windows 系统的 qcow2 文件，复制文件出现 Read-only file system 的问题处理

```shell
[root@localhost ~]# run guestmount -a {qcow2文件} --rw -m /dev/sda2 /mnt1
cp /data/test/Dbs/qyi-04368232000-5921.db /mnt1/Windows/QyCloud/
cp: cannot create regular file ‘/mnt1/Windows/QyCloud/qyi-04368232000-5921.db’: Read-only file system
```

一般出现`Read-only file system`，由于 windows 系统未正常关机，导致文件的写保护。这时只需要将虚拟机重新开机，再手动关机，再执行挂载复制操作即可解决这个问题。



### 4. 怎样将宝塔的文件移动新的磁盘后，将新目录挂载到旧目录上，并开机自动挂载？

今天有个客户新增了`1T`的新磁盘，想把之前的两个`100G`的磁盘删掉。但需要保证他原在`100G`磁盘运行的宝塔程序，在删除磁盘后还可以正常运行。原磁盘目录结构如下：

```shell
/www    # 100G 宝塔的目录
/data1  # 100G
/data2  # 1T磁盘
```

首先我先在`/data2`目录下创建两个新的文件夹保存旧的两个磁盘的数据。

```shell
mkdir /data2/www
mkdir /data2/data1
cp -r /www/* /data2/www/
cp -r /data1/* /data2/data1
```

为了保证原宝塔可以正常运行，我需要将新的文件夹`/data2/www`挂载到原`/www`目录上，这样他原宝塔的任何文件我都不需要变动。在 `/etc/fstab`保存新的挂载：

```shell
/data2/www /www none bind 0 0
/data2/data1 /data1 none bind 0 0
```

注意上面挂载方式跟设备的挂载不一样，其中`none`指定文件系统类型为`none`, `bind`表示使用`bind`挂载，`0 0`分别代表用户和组的访问权限。

最后使用如下命令删除那两个`100G`的磁盘，再重启机器就可以正常挂载，宝塔也可以正常运行了。

```shell
virsh detach-disk {guid} vdb --persistent
virsh detach-disk {guid} vdd --persistent
```


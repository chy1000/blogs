# 怎样扩容 Centos 系统磁盘

当客户的`Centos`系统磁盘暴满后，我们有什么办法可以帮客户升级系统磁盘呢？


#### 第一种方法：使用`qemu-img resize`扩大磁盘，再进入系统使用`LVM`进行扩容

1. 首先云服务器关机。

2. 在母机备份好客户的磁盘文件，防止扩容出错文件损坏，导致客户机器数据丢失。再使用`qemu-img resize`对系统磁盘进行扩容

   ```shell
   qemu-img resize /data/qycloud/Disks/System/qyi-88736948001-5901-01.img +80G
   ```

3. 进入客户的机器使用`LVM`进行扩容

   ```shell
   lsblk
   fdisk -l
   pvcreate /dev/vda3
   vgextend centos /dev/vda3
   # 如果出现：Cannot archive volume group metadata for centos to read-only filesystem.
   mount -o remount rw /
   lvextend -l +100%FREE /dev/mapper/centos-root
   # 使用 xfs_growfs 扩展一个现存的 xfs 文件系统
   xfs_growfs /dev/mapper/centos-root
   ```

   

#### 第二种方法：新购一个磁盘，再进入系统使用`LVM`进行扩容

```shell
lsblk
fdisk -l
pvcreate /dev/vdb2
vgextend centos /dev/vdb2
lvextend -l +100%FREE /dev/mapper/centos-root
# 使用 xfs_growfs 扩展一个现存的 xfs 文件系统
xfs_growfs /dev/mapper/centos-root
```


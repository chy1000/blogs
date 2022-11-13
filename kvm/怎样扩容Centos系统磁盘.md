# 怎样扩容 Centos 系统磁盘

当客户的`Centos`系统磁盘暴满后，我们有什么办法可以帮客户升级系统磁盘呢？


#### 第一种方法：使用`qemu-img resize`扩大磁盘，再进入系统使用`LVM`进行扩容

1. 首先云服务器关机。

2. 在母机备份好客户的磁盘文件，防止扩容出错文件损坏，导致客户机器数据丢失。再使用`qemu-img resize`对系统磁盘进行扩容

   ```shell
   qemu-img resize /data/qycloud/Disks/System/{guid}-01.img +80G
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

----------

### 2022-11-11 更新

今天又有客户的`Centos`系统磁盘暴满，虚拟机的密码文件也丢失了。使用`qemu-img resize `扩大磁盘，再使用`guestmount -a `挂载到本地，打算创建`/etc/shadow`文件，发现还是提示空间不足。因为LVM的文件并没有添加增大，需进入系统进行扩容操作。那只能删除现在的系统盘的文件，磁盘有剩余空间了，才能创建密码文件进入系统，这样操作起来十分麻烦。

于是重新看`libguestfs`工具，发现居然有现成的工具可以实现，不需要开机进入系统操作

```shell
virt-filesystems -a {虚拟机文件} -l
/dev/sda1         filesystem  xfs  -      524288000    -
/dev/centos/root  filesystem  xfs  -      18798870528  -

qemu-img create -f qcow2 {扩容后的虚拟机文件} 40G

virt-resize --expand /dev/sda2 --LV-expand /dev/centos/root {虚拟机文件} {扩容后的虚拟机文件}
....
Resize operation completed with no errors.  Before deleting the old disk,
carefully check that the resized disk boots and works correctly.
```

执行上面的命令，命令运行一段时间后，磁盘扩容成功。我们再使用下面的命令，看看`LV`有没有增加

```shell
virt-filesystems -a {扩容后的虚拟机文件} -l
Name              Type        VFS  Label  Size         Parent
/dev/sda1         filesystem  xfs  -      524288000    -
/dev/centos/root  filesystem  xfs  -      40273707008  -
```

可以看到`/dev/centos/root`已经是`40G`了。

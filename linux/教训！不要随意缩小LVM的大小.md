# 教训！不要随意缩小LVM的大小

在一台已安装好系统并使用LVM分好区的物理服务器，我想从挂载在`home`目录的`/dev/sdb1`设备缩小点空间给`docker`用。

```shell
[root@localhost ~]# df -hT
Filesystem              Type      Size  Used Avail Use% Mounted on
/dev/mapper/centos-root xfs        50G  2.4G   48G   5% /
devtmpfs                devtmpfs  7.8G     0  7.8G   0% /dev
tmpfs                   tmpfs     7.8G     0  7.8G   0% /dev/shm
tmpfs                   tmpfs     7.8G  825M  7.0G  11% /run
tmpfs                   tmpfs     7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/sda1               xfs      1014M  145M  870M  15% /boot
tmpfs                   tmpfs     1.6G     0  1.6G   0% /run/user/0
/dev/mapper/centos-home xfs       500G   37G  463G   7% /home
```

根据我的半吊子`lvm`知识，我认为`lvm`是可以随意增加和缩小的，于是没有备份就直接对`/dev/sdb1`进行`lvreduce`操作了。

```shell
[root@localhost ~]# lvreduce -L 480G /dev/centos/home
[root@localhost ~]# xfs_growfs /dev/mapper/centos-home
[root@localhost ~]# lvs
  LV   VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home centos -wi-a----- 480.00g
  root centos -wi-ao----  50.00g
  swap centos -wi-ao----  <7.88g
[root@localhost ~]# df -hT
Filesystem              Type      Size  Used Avail Use% Mounted on
/dev/mapper/centos-root xfs        50G  2.4G   48G   5% /
devtmpfs                devtmpfs  7.8G     0  7.8G   0% /dev
tmpfs                   tmpfs     7.8G     0  7.8G   0% /dev/shm
tmpfs                   tmpfs     7.8G  825M  7.0G  11% /run
tmpfs                   tmpfs     7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/sda1               xfs      1014M  145M  870M  15% /boot
tmpfs                   tmpfs     1.6G     0  1.6G   0% /run/user/0
/dev/mapper/centos-home xfs       500G   37G  463G   7% /home
```

发现磁盘并没有减少到容量，于是我`unmount /dev/mapper/centos-home`，再想挂载回去时，出错了。

```shell
[root@localhost ~]# mount /dev/mapper/centos-home /home
mount: /dev/mapper/centos-home: can't read superblock
[root@localhost ~]# xfs_repair /dev/mapper/centos-home -L
Phase 1 - find and verify superblock...
xfs_repair: error - read only 0 of 512 bytes
```

这下悲剧了直接把`/dev/sdb1`搞崩了。这时我才上网搜索正确的缩小容量的方法，知道`xfs`的`lvm`不能直接在线缩减空间，一般只扩不缩。

> xfs文件系统只支持增大分区空间的情况，不支持减小的情况（切记！！）。
> 硬要减小的话，只能在减小后将逻辑分区重新通过mkfs.xfs命令重新格式化才能挂载上，这样的话这个逻辑分区上原来的数据就丢失了。如果有重要文件，那就歇菜喽～～～
> https://www.cnblogs.com/kevingrace/p/5825963.html

万不得意，要缩小空间，需备份好数据再操作，正确的步骤如下所示：

```shell
1. 备份数据
# xfsdump -f ~/home.dump /home -L data_dump -M data_dump
# L：xfsdump 会记录每次备份的 session 标头，这里可以填写针对此文件系统的建议说明
# M：xfsdump 可以记录存储媒体的标头，这里可以填写此媒体的建议说明
# l：小写的 L，指定等级。有 0~9 共 10 个等级，预设为 0 完整备份
2. 卸载分区
# umount /home
3. 减小分区大小
# lvreduce -L 480G /dev/mapper/centos-home
4. 使用 XFS 文件系统格式化分区
# mkfs.xfs -f /dev/mapper/centos-home
5. 重新安装分区
# mount /dev/mapper/centos-home /home
6. 恢复数据
# xfsrestore -f ~/home.dump /home
```

万幸的是，这台服务器并没有十分重要的数据，它的数据在其它服务器都是有的，只是要花点时间从别的服务器拉过来。

-------

如果仅是扩大，这是挺简单的

```
lvextend -l +100%FREE /dev/mapper/centos-home
xfs_growfs /dev/mapper/centos-home
```


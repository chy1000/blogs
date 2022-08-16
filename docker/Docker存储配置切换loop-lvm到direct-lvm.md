# Docker存储配置切换loop-lvm到direct-lvm

最近看文章才知道，`docker`有两种的存储模式，分别是`docker`默认的`loop volume`和`Direct LVM`。

#### 什么是`loop`设备

是一种伪设备，是使用文件来模拟块设备的一种技术，文件模拟成块设备后，就像一个磁盘或光盘一样使用。
在`docker`使用 `loop`设备，我们不需要额外的配置，程序自动化配置。但这种存储方式稳定性差、性能也差。运行`docker info`就可以看到提示，包括docker官方都不推荐使用loop设备的存储方式，而是推荐使用Direct LVM存储。

```shell
[root@QYSR058Z ~]# docker info
.....
WARNING: devicemapper: usage of loopback devices is strongly discouraged for production use.
         Use `--storage-opt dm.thinpooldev` to specify a custom block storage device.
```



#### 手动配置Direct LVM

```shell
# 查看磁盘是否已增加
fdisk -l
# 查看磁盘的树形结构
lsblk
# 创建物理卷
pvcreate /dev/vdc
# 创建逻辑卷组
vgcreate docker /dev/vdc
# 创建能够组成 thin-pool 的两个LV
lvcreate --wipesignatures y -n thinpool docker -l 95%VG
lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG
# 创建thin-pool
lvconvert -y --zero n -c 512K --thinpool docker/thinpool --poolmetadata docker/thinpoolmeta
# 配置thin-pool的自动扩展
vi /etc/lvm/profile/docker-thinpool.profile
activation {
  thin_pool_autoextend_threshold=80
  thin_pool_autoextend_percent=20
}

# 激活lv的配置文件LVM profile
lvchange --metadataprofile docker-thinpool docker/thinpool
# 对主机上的逻辑卷启用监视
lvs -o+seg_monitor
# 配置 /etc/docker/daemon.json
mkdir /etc/docker/
vi daemon.json
{
    "storage-driver": "devicemapper",
    "storage-opts": [
        "dm.thinpooldev=/dev/mapper/docker-thinpool",
        "dm.use_deferred_removal=true",
        "dm.use_deferred_deletion=true"
    ]
}

# 如果之前编辑过 docker.service，相应修改
vi /usr/lib/systemd/system/docker.service
cat /usr/lib/systemd/system/docker.service
# 重启 docker
systemctl daemon-reload
systemctl restart docker
# 查看配置是否已生效
docker info
....
Storage Driver: devicemapper
Pool Name: docker-thinpool
.....
```

当执行上面的配置时出现报错：Error starting daemon: error initializing graphdriver: devmapper: Unable to take ownership of thin-pool (docker-thinpool) that already has used data blocks，我们按下面的命令删除 thinpool ，再重新创建。

```
lvremove -f /dev/mapper/docker-thinpool
lvcreate --wipesignatures y -n Thinpool docker -l 95%VG
lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG
# 其余设置
```


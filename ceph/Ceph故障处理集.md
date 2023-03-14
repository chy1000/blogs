# Ceph故障处理集

## `rbd`出现`Input/output error`的问题

```shell
[root@node1 home]# ll
ls: cannot access testfile1: Input/output error
total 4997137
drwx------ 2 root root      12288 Feb 17 16:39 lost+found
-rw-r--r-- 1 root root          2 Feb 28 16:36 test
-rw-r--r-- 1 root root 2642411520 Feb 28 10:01 testfile
-????????? ? ?    ?             ?            ? testfile1
-rw-r--r-- 1 root root         17 Feb 17 16:41 test.log
```

应该是我使用`dd`做`ceph`的写入测试时，集群出现问题，导致文件没有完整创建，使用`dmesg | tail`查看详细的报错。

```shell
[root@node1 home]# dmesg | tail
[  707.807911] EXT4-fs error (device rbd0): ext4_lookup:1441: inode #2: comm ls: deleted inode referenced: 14
[  722.282029] EXT4-fs error (device rbd0): ext4_lookup:1441: inode #2: comm ls: deleted inode referenced: 14
[  730.594987] EXT4-fs error (device rbd0): ext4_lookup:1441: inode #2: comm rm: deleted inode referenced: 14
[  730.603129] EXT4-fs error (device rbd0): ext4_lookup:1441: inode #2: comm rm: deleted inode referenced: 14
[  994.720685] EXT4-fs (rbd0): error count since last fsck: 112
[  994.720701] EXT4-fs (rbd0): initial error at time 1677549679: ext4_mb_generate_buddy:757
[  994.720707] EXT4-fs (rbd0): last error at time 1677573092: ext4_lookup:1441: inode 2
[ 1056.250868] EXT4-fs error (device rbd0): ext4_lookup:1441: inode #2: comm ls: deleted inode referenced: 14
[21134.816916] libceph: mon0 192.168.20.5:6789 session established
[60630.246588] EXT4-fs error (device rbd0): ext4_lookup:1441: inode #2: comm ls: deleted inode referenced: 14
```

上面报错提示引用了已删除的索引。这时使用`fsck`检测文件系统的完整性并纠正任何错误，经修复重新挂载，终于正常了。

```
umount /dev/rbd0
fsck -s /dev/rbd0 -y
```



## `rbd remove`出现报错：`rbd: error: image still has watchers`的处理

```shell
[root@master ~]# rbd remove kubernetes/mysql.rbd
2023-03-03 15:38:43.255921 7f2838d3c700  0 monclient: hunting for new mon
2023-03-03 15:38:43.288808 7f285439ad80 -1 librbd: cannot obtain exclusive lock - not removing
Removing image: 0% complete...failed.
rbd: error: image still has watchers
This means the image is still open or the client using it crashed. Try again after closing/unmapping it or waiting 30s for the crashed client to timeout.
```

查看`rbd`的状态

```
[root@master ~]# rbd status kubernetes/mysql.rbd
2023-03-03 15:39:23.927804 7fb8fe4d4700  0 monclient: hunting for new mon
Watchers:
        watcher=192.168.30.3:0/3295563347 client.154237 cookie=18446462598732841054
```

该`image`仍旧被一个客户端（`192.168.30.3`）在访问，具体表现为该`image`中有`watcher`。进入被占用的节点，查看`rbd`的映射情况，取消映射。

```shell
[root@node1 ~]# rbd showmapped
id pool       image     snap device
0  kubernetes mysql.rbd -    /dev/rbd0
# 取消映射，出现错误
[root@node1 ~]# rbd unmap /dev/rbd0
rbd: sysfs write failed
rbd: unmap failed: (16) Device or resource busy
# 强制取消映射
[root@node1 ~]# rbd unmap -o force /dev/rbd0
```

再删除`images`

```shell
[root@master ~]# rbd remove kubernetes/mysql.rbd
2023-03-03 16:03:37.411726 7fb4e7b81700  0 monclient: hunting for new mon
Removing image: 100% complete...done.
```



### 一块 RBD 设备供多个 POD 使用，同时写入时会引发冲突问题

我一开始使用 helm 创建 mysql 主从集群时，使用 rbd 作为它的持久存储。我创建两个可用的`PersistentVolume`，具体配置如下所示：

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-mysql-pv1
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  rbd:
    monitors:
      - 192.168.20.5:6789
      - 192.168.30.5:6789
      - 192.168.40.5:6789
    pool: kubernetes
    image: mysql.rbd
    user: k8s
    secretRef:
      name: ceph-secret
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: ceph-rbd-storage
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-mysql-pv2
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  rbd:
    monitors:
      - 192.168.20.5:6789
      - 192.168.30.5:6789
      - 192.168.40.5:6789
    pool: kubernetes
    image: mysql.rbd
    user: k8s
    secretRef:
      name: ceph-secret
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: ceph-rbd-storage
```

`ceph-mysql-pv1`和`ceph-mysql-pv2`都关联到了`mysql.rbd`，主从数据都会对`mysql.rbd`进行读写操作，会引发冲突的问题。解决这个问题可以将这两个`pv`分别关联到不同的rbd块上，或者创建动态的StorageClass。



### “FailedScheduling  74s   default-scheduler  0/4 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 1 node(s) were unschedulable, 2 node(s) didn't match pod anti-affinity rules.” 的问题

```
[root@master ceph]# kubectl get pod -o wide -n ceph-csi-rbd
NAME                                        READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
ceph-csi-rbd-nodeplugin-mlg5c               3/3     Running   0          41s   192.168.50.3   node3    <none>           <none>
ceph-csi-rbd-nodeplugin-nlj8f               3/3     Running   0          41s   192.168.30.3   node1    <none>           <none>
ceph-csi-rbd-nodeplugin-nrhwm               3/3     Running   0          41s   192.168.40.3   node2    <none>           <none>
ceph-csi-rbd-provisioner-66c56c85cf-68hm4   7/7     Running   0          41s   10.244.3.6     node3    <none>           <none>
ceph-csi-rbd-provisioner-66c56c85cf-sp4zm   0/7     Pending   0          41s   <none>         <none>   <none>           <none>
ceph-csi-rbd-provisioner-66c56c85cf-vghhr   7/7     Running   0          41s   10.244.1.93    node1    <none>           <none>
```

```
[root@master ceph]# kubectl describe pod ceph-csi-rbd-provisioner-66c56c85cf-sp4zm -n ceph-csi-rbd
Name:                 ceph-csi-rbd-provisioner-66c56c85cf-sp4zm
Namespace:            ceph-csi-rbd
....
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  74s   default-scheduler  0/4 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 1 node(s) were unschedulable, 2 node(s) didn't match pod anti-affinity rules.
```

```
[root@master ceph]# kubectl get nodes
NAME     STATUS                     ROLES                  AGE   VERSION
master   Ready                      control-plane,master   26d   v1.22.4
node1    Ready                      <none>                 26d   v1.22.4
node2    Ready,SchedulingDisabled   <none>                 26d   v1.22.4
node3    Ready                      <none>                 26d   v1.22.4
```

SchedulingDisabled是禁止调试的标志，但很奇怪ceph-csi-rbd-nodeplugin-nrhwm还是调度到了node2上，网上搜索也有例子类似的[例子](https://blog.csdn.net/kozazyh/article/details/102925571)

```
kubectl patch node node2 -p '{"spec":{"unschedulable":false}}'
# 或者
kubectl uncordon node2
```


# Ceph故障处理

## `RBD`出现`Input/output error`的问题

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


# virsh managedsave 保存运行状态

在使用 PVE 时，发现它的快照功能特别好用。快照回滚时，机器可以直接恢复到原来的运行状态。

```shell
root@pve:~# lvs
  Logical Volume
  ==============
  LV                     VG  Attr       LSize   Pool Origin          Data%  Meta%  Move Log Cpy%Sync Convert
  base-100-disk-0        pve Vri-a-tz-k  40.00g data                 18.00
  data                   pve twi-aotz-- 147.12g                      5.65   1.29
  root                   pve -wi-ao----  57.75g
  snap_vm-101-disk-0_sn1 pve Vri---tz-k  40.00g data vm-101-disk-0
  snap_vm-101-disk-0_sn2 pve Vri---tz-k  40.00g data vm-101-disk-0
  snap_vm-101-disk-0_sn3 pve Vri---tz-k  40.00g data vm-101-disk-0
  swap                   pve -wi-ao----   8.00g
  vm-101-disk-0          pve Vwi-aotz--  40.00g data base-100-disk-0 18.00
  vm-101-state-sn1       pve Vwi-a-tz--  <8.49g data                 3.69
  vm-101-state-sn2       pve Vwi-a-tz--  <8.49g data                 4.27
  vm-101-state-sn3       pve Vwi-a-tz--  <8.49g data                 4.73
```

从上面的命令，我们可以看出，每次做快照时，会生成一个状态的快照，如 vm-101-state-sn 这样的。这个状态快照保存的是虚拟机的内存状态的。当快照回滚时，这个状态就会成到效果将机器恢复到原来的运行状态。在 centos 系统下，我们使用的是 libvirt 操作虚拟机，libvirt 是否也有类似的命令保存内存快照状态？经过搜索，发现果然 libvirt 也有类似的命令：

```shell
[root@kvm ~]# virsh managedsave --help
  NAME
    managedsave - managed save of a domain state

  SYNOPSIS
    managedsave <domain> [--bypass-cache] [--running] [--paused] [--verbose]

  DESCRIPTION
    Save and destroy a running domain, so it can be restarted from
    the same state at a later time.  When the virsh 'start'
    command is next run for the domain, it will automatically
    be started from this saved state.

  OPTIONS
    [--domain] <string>  domain name, id or uuid
    --bypass-cache   avoid file system cache when saving
    --running        set domain to be running on next start
    --paused         set domain to be paused on next start
    --verbose        display the progress of save
```

当我们执行 `virsh managedsave` 命令，会在 `/var/lib/libvirt/qemu/save/` 目录下生成一个状态文件，命令执行完成会将机器关闭。使用`virsh start` 启动机器，机器会恢复到我执行 `virsh managedsave` 命令前的状态。我使用的是 windows 系统进行测试的，桌面和打开的程序跟执行`virsh managedsave` 命令前是一模一样的。

```shell
[root@kvm ~]# virsh managedsave qyi-99118396001-5932 --verbose
Managedsave: [100 %]
[root@kvm ~]# ll -h /var/lib/libvirt/qemu/save/
-rw------- 1 root root 458M Jul 21 11:58 /var/lib/libvirt/qemu/save/qyi-99118396001-5932.save
[root@kvm ~]# virsh start qyi-99118396001-5932
Domain qyi-99118396001-5932 started
[root@kvm ~]# ll -h /var/lib/libvirt/qemu/save/
total 0
```

从上面的例子，我们可以看到当机器启动后，qyi-99118396001-5932.save 就被删除了。如果我们要实现像 PVE 那样，每个快照都保存机器状态，要怎样做呢？`cp /var/lib/libvirt/qemu/save/qyi-99118396001-5932.save {保存状态的目录}` 将状态文件保存到别的机器，当需要恢复时，再将文件拷贝回 `/var/lib/libvirt/qemu/save/` 重新启动机器就可以了。

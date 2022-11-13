# kvm遇到的问题整理

### 怎样更改网络模式实现NAT上网？

将`xml`的网络模式更改为`user`模式后启动虚拟机，`dhcp`会获取一个 `10.0.2.xx`的内部IP，实现`NAT`上网。如果`dhcp`不能正确获取内部IP，可手动设置为 `10.0.2.xx/24`网关设置为`10.0.2.2`。

```
    <interface type='user'>
      <mac address='00:4A:DB:C3:61:CD'/>
      <model type='rtl8139'/>
    </interface>
```

如果还想在外面远程连接到内部的虚拟机，可配置`qemu:commandline`将虚拟机的3389的端口影射为宿主机的端口，从而实现外部远程访问。

```
  <qemu:commandline>
     <qemu:arg value='-redir'/>
     <qemu:arg value='tcp:55947::3389'/>
  </qemu:commandline>
```



### 怎样在xml添加cdrom加载ISO?

```

```


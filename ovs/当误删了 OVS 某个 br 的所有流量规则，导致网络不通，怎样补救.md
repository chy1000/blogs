# 当误删了 OVS 某个 br 的所有流量规则，导致网络不通，怎样补救

当发现删错了规则，然后 ping IP 发现网络不通了。。。OMG，又做错事了。然后马上使用以下语句恢复：

```
ovs-ofctl del-flows br10 "in_port=5926"
ovs-ofctl add-flow br10 "ip, in_port=5926, table=0, priority=99, nw_src=125.xx.xx.122, actions=resubmit(,1)"
ovs-ofctl add-flow br10 "ip, in_port=5926, table=0, priority=98, actions=drop"
ovs-ofctl add-flow br10 "in_port=5926, table=1, actions=normal"
```

发现规则是补上去了，但IP还是不通，怎么回事？后来上网搜索到 ovs-appctl bridge/dump-flows br10 语句可以显示所以的规则。使用这条语句分别输入一个可以正常运行的br的规则和输出刚刚错删的规则：

```shell
# 正常的 br
ovs-appctl bridge/dump-flows br13
duration=7s, priority=98, n_packets=8, n_bytes=8, priority=98,ip,in_port=5909,actions=drop
duration=7s, priority=99, n_packets=3, n_bytes=3, priority=99,ip,in_port=5908,nw_src=IP,actions=resubmit(,1)
duration=5s, priority=0, n_packets=5588, n_bytes=15786082, priority=0,actions=NORMAL
table_id=1, duration=5s, priority=3, n_packets=8, n_bytes=18, in_port=5911,actions=NORMAL
table_id=254, duration=5s, priority=0, n_packets=0, n_bytes=0, priority=0,reg0=0x3,actions=drop
table_id=254, duration=5s, priority=0, n_packets=0, n_bytes=0, priority=0,reg0=0x1,actions=controller(reason=no_match)
table_id=254, duration=5s, priority=0, n_packets=0, n_bytes=0, priority=0,reg0=0x2,actions=drop

# 有问题的 br
ovs-appctl bridge/dump-flows br10
duration=7s, priority=98, n_packets=0, n_bytes=0, priority=98,ip,in_port=5927,actions=drop
duration=7s, priority=99, n_packets=158, n_bytes=15692, priority=99,ip,in_port=5927,nw_src=IP,actions=resubmit(,1)
table_id=1, duration=7476s, priority=32768, n_packets=0, n_bytes=0, in_port=5926,actions=NORMAL
table_id=1, duration=7514s, priority=32768, n_packets=0, n_bytes=0, in_port=5902,actions=NORMAL
table_id=1, duration=7904s, priority=32768, n_packets=158, n_bytes=15692, in_port=5927,actions=NORMAL
table_id=1, duration=1209s, priority=32768, n_packets=0, n_bytes=0, in_port=5914,actions=NORMAL
table_id=1, duration=7500s, priority=32768, n_packets=0, n_bytes=0, in_port=5900,actions=NORMAL
table_id=254, duration=15855262s, priority=0, n_packets=0, n_bytes=0, priority=0,reg0=0x3,actions=drop
table_id=254, duration=15855262s, priority=0, n_packets=57974, n_bytes=5033140, priority=0,reg0=0x1,actions=controller(reason=no_match)
table_id=254, duration=15855262s, priority=0, n_packets=0, n_bytes=0, priority=0,reg0=0x2,actions=drop

```

仔细对比，发现正常的多了一条规则：

```shell
duration=5s, priority=0, n_packets=5588, n_bytes=15786082, priority=0,actions=NORMAL
```

根据这个例子，这条语句表示当不符合 priority=98、priority=99 的数据包会通过这条规则。由于这条规则不是针对某个端口的，也就是这条规则是设置所有访问 br10 的规则，没有这条规则 ping 的包都进不入 br 自然不能正常流转了。

然后使用语句补回这一行：

```shell
ovs-ofctl add-flow br10 "table=0, priority=0, actions=normal"
```

最后惊喜的发现，问题终于解决了。

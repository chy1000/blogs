# 解决网站访问出现No route to host的问题

当我在一台服务器使用`go mod tidy`下载`golang`第三方包时，出现`github.com/docker/docker/api/types/events: module github.com/docker/docker/api/types/events: Get "https://goproxy.cn/github.com/docker/docker/api/types/eve: connect: no route to host`的报错。

我直接curl https://goproxy.cn ，看看是否可以正常访问。

```shell
[root@test DockerProtector]# curl https://goproxy.cn
curl: (7) Failed connect to goproxy.cn:443; No route to host
```

很明显`goproxy.cn`访问不了，`No route to host`按字面意思是没有路由，做个路由跟踪看看：

```shell
[root@test DockerProtector]# traceroute goproxy.cn
traceroute to goproxy.cn (61.136.173.41), 30 hops max, 60 byte packets
 1  122.xx.194.77 (122.xx.194.77)  3005.343 ms !H  3005.292 ms !H  3005.269 ms !H
```

看到路由跟踪的返回结果，我才幡然醒悟，这是台双线服务器，访问`goproxy.cn`时跑到联通的路由上去了，而这个双线服务器的联通IP不能正常访问了，导致了访问不可达。查看`route -n`路由记录表，印证我的猜测是对的。

```shell
[root@test DockerProtector]# route -n | grep 61.136
61.136.0.0      122.xx.194.1    255.255.0.0     UG    0      0        0 em1
```

解决问题的方法很简单，把这条联通的路由删除了，访问`goproxy.cn`就会使用到电信的路由了。

```shell
[root@test DockerProtector]# route del -net 61.136.0.0/16
```


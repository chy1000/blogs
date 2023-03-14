# K8S节点挂了，Pod状态长时间不更新的问题

使用下面的`yaml`创建`deployment`，部署`2`副本的`pod`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s-deployment
spec:
  strategy:
     rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
        - image: chy2046/hellok8s:healthz
          name: hellok8s-container
          livenessProbe:
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 3
          readinessProbe:
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 3
```

执行下面的命令可以看到pod已经正常创建运行：

```shell
[root@master k8s]# kubectl get pods -o wide
NAME                                   READY   STATUS        RESTARTS        AGE     IP             NODE    NOMINATED NODE   READINESS GATES
hellok8s-deployment-85556cff99-5lwnk   1/1     Running       0               45s     10.244.3.105   node3   <none>           <none>
hellok8s-deployment-85556cff99-62gzn   1/1     Running       0               17h     10.244.1.121   node1   <none>           <none>
```

这时我将`node3`节点关机，再观察`pod`的状态。可是等过了2、3分钟，pod还是之前的状态显示，按我认知，觉得节点出现问题，`pod`的状态应该要立即更新才对，这让我有点困惑。

为什么会出现这种情况呢？网上搜索相关资料，发现是污点和容忍度影响所致。下面是从`infoq`文章搜索到的关于污点和容忍度说明和参数值说明，摘录如下：

> `k8s`每个节点上都可以应用一个或多个`taint`，这表示对于那些不能容忍这些`taint`的`pod`，是不会被该节点接受的。如果将`toleration`应用于`pod`上，则表示这些`pod`可以（但不要求）被调度到具有相应`taint`的节点上。

> ##### Taint 的 effect 值：
> `NoSchedule`：一定不能被调度。
> `PreferNoSchedule`：尽量不要调度。
> `NoExecute`：不仅不会调度，还会驱逐`Node`上已有的`Pod`。

> ##### Toleration 的 effect 值：
>
> `NoSchedule`：如果一个`pod`没有声明容忍这个`Taint`，则系统不会把该`Pod`调度到有这个`Taint`的`node`上
> `PreferNoSchedule`：`NoSchedule`的软限制版本，如果一个`Pod`没有声明容忍这个`Taint`，则系统会尽量避免把这个`pod`调度到这一节点上去，但不是强制的。
> `NoExecute`：定义`pod`的驱逐行为，以应对节点故障。
>
> `NoExecute`这个`Taint`效果对节点上正在运行的`pod`有以下影响：
>
> - 没有设置`Toleration`的`Pod`会被立刻驱逐
> - 配置了对应`Toleration`的`pod`，如果没有为`tolerationSeconds`赋值，则会一直留在这一节点中
> - 配置了对应`Toleration`的`pod`且指定了`tolerationSeconds`值，则会在指定时间后驱逐

当`node3`挂了，集群会将`node3`节点标记为`node.kubernetes.io/not-ready`污点，表示如果没有容忍该污点的`pod`会被驱逐。

```shell
[root@master k8s]# kubectl describe node node2 | grep Taints
Taints:             node.kubernetes.io/not-ready:NoExecute
```

而`pod`在创建时默认设置了容忍度：

```shell
[root@master k8s]# kubectl get pod hellok8s-deployment-85556cff99-g9brq -o yaml | grep tolerations -A 10
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
...
```

上面定义的容忍度指定了`tolerationSeconds`值，表明在`300`秒（即`5`分钟）后，`pod`才会被驱逐。

于是我等了`5`分钟后，终于看到`node3`节点的`pod`被驱逐，状态变成`Terminating`了。由于`deployment`上我定义的是双副本，所以集群又重新调度在`node2`生成一个新的`pod`。

```shell
[root@master k8s]# kubectl get pods -o wide
NAME                                   READY   STATUS        RESTARTS        AGE     IP             NODE    NOMINATED NODE   READINESS GATES
hellok8s-deployment-85556cff99-5lwnk   1/1     Terminating   0               45s     10.244.3.105   node3   <none>           <none>
hellok8s-deployment-85556cff99-62gzn   1/1     Running       0               17h     10.244.1.121   node1   <none>           <none>
hellok8s-deployment-85556cff99-5lwnk   1/1     Running       0               4s      10.244.2.78    node2   <none>           <none>
```

另外需要补充说明的是，只有无状态的应用，节点挂了，集群才会生新调度生成新的`pod`，而有状态的应用（`StatefulSet`）是不会调度生成的。



### 参考：

[污点和容忍度](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/taint-and-toleration/)

[Kubernetes 污点与容忍详解](https://www.infoq.cn/article/0ycxb8vacb6pknzhudxy)

# K8S使用Ceph作为持久化存储

对于docker我算是比较熟悉的了，很早就使用docker应用到实际的生产环境。无论php网站，还是mysql数据库，抑或是redis缓存都是用docker搭建。在使用docker时，为了数据的持久化，都会将容器内部的目录映射到宿主机的目录上，这样即使容器崩了，重建容器也不会丢失数据。

## 使用本地持久化存储

k8s使用的也是容器，同样存在数据持久化存储的问题。k8s也可以像docker那样使用宿主机的磁盘存储空间，下面是helm安装的mysql主从集群，使用节点的本地磁盘持久化存储的例子：

```yaml
# local-storage.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv1
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /home/mysql
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv2
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /home/mysql
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node2
```

使用`kubectl apply -f local-storage.yaml`命令创建两个`Persistent Volume`(`PV`)，将卷绑定到`node1`和`node2`的本地目录，`storageClassName`都为`local-storage`。接下来使用这两个`pv`卷，安装`mysql`主从集群。

```shell
# 添加 helm 源
helm repo add bitnami https://charts.bitnami.com/bitnami
# 导出配置模版 values.yaml
helm show values bitnami/mysql > values.yaml
# 修改模版
vi values.yaml
# global.storageClass 设置为 local-storage，这里要和pv卷的storageClassName一致
# image.debug 修改为 true
# architecture 修改为 replication
# secondary.replicaCount 1 保持不变，只创建一个从数据库
# 安装主从集群
helm install mysql8 bitnami/mysql -f values.yaml
```

helm会自动创建pvc，并根据pvc申请找到合适的pv卷进行绑定。

```shell
[root@master k8s]# kubectl get pvc
NAME                         STATUS    VOLUME                  CAPACITY   ACCESS MODES   STORAGECLASS    AGE
data-mysql8-primary-0     Bound     mysql-pv2                  10Gi       RWO            local-storage   13m
data-mysql8-secondary-0   Bound     mysql-pv1                  10Gi       RWO            local-storage   13m
[root@master k8s]# kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                STORAGECLASS    REASON   AGE
mysql-pv1      10Gi       RWO            Delete           Bound    default/data-mysql8-secondary-0   local-storage            14m
mysql-pv2      10Gi       RWO            Delete           Bound    default/data-mysql8-primary-0     local-storage            14m
```

如果k8s启动集群过程中，没啥问题，过一会可以看到pod是Running状态并且READY都为1/1，集群就可以正常使用了。

```shell
[root@master k8s]# kubectl get all
NAME                     READY   STATUS    RESTARTS      AGE
pod/mysql8-primary-0     1/1     Running   0             21m
pod/mysql8-secondary-0   1/1     Running   0             21m

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/mysql8-primary              ClusterIP   10.105.188.147   <none>        3306/TCP   21m
service/mysql8-primary-headless     ClusterIP   None             <none>        3306/TCP   21m
service/mysql8-secondary            ClusterIP   10.106.85.95     <none>        3306/TCP   21m
service/mysql8-secondary-headless   ClusterIP   None             <none>        3306/TCP   21m

NAME                                READY   AGE
statefulset.apps/mysql8-primary     1/1     21m
statefulset.apps/mysql8-secondary   1/2     21m
```

开启端口映射在外部测试`mysql`集群是否可以正常连接和同步。

```shell
kubectl port-forward service/mysql8-primary 3306:3306 --address 0.0.0.0
kubectl port-forward service/mysql8-secondary 3307:3306 --address 0.0.0.0
```



## 使用`Ceph`持久化存储

```
rbd map kubernetes/mysql.rbd --id k8s --keyring /home/ceph/client.k8s.keyring
```

在`ceph`节点上创建资源池、用户和`rbd`块。

```shell
1.创建一个Pool资源池
[root@ceph-node1 ~]# ceph osd pool create kubernetes 128 128

2.创建认证用户
[root@ceph-node1 ~]# ceph auth get-or-create client.k8s mon 'allow r' osd 'allow class-read object_prefix rbd_children,allow rwx pool=kubernetes'

3.创建两个rbd块
[root@ceph-node1 ~]# rbd create -p kubernetes --image mysql.primary.rbd --size 10G
[root@ceph-node1 ~]# rbd create -p kubernetes --image mysql.secondary.rbd --size 10G
```

在`k8s`集群的所有节点中安装`ceph`命令

```
yum -y install ceph-common
```
创建`pod`资源使用`ceph`集群的`rbd`块存储进行数据持久化

```yaml
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
    image: mysql.primary.rbd
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
    image: mysql.secondary.rbd
    user: k8s
    secretRef:
      name: ceph-secret
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: ceph-rbd-storage
```




## 使用 helm 安装 mysql 主从

```
helm install mysql8 bitnami/mysql -f values.yaml
helm uninstall mysql8; kubectl delete pvc --all; kubectl delete pv --all
kubectl port-forward service/mysql8-primary 3306:3306 --address 0.0.0.0
kubectl port-forward service/mysql8-secondary 3307:3306 --address 0.0.0.0

rbd ls -p kubernetes
kubectl describe pv pvc-5f6f56a9-2cad-4c8c-9ed0-f3c15ccd11db | grep VolumeHandle

kubectl scale --replicas=0 deployment/<your-deployment>
```



## 使用 helm 安装 ceph csi

```
helm repo add ceph-csi https://ceph.github.io/csi-charts
kubectl create namespace "ceph-csi-rbd"
# 参考: https://artifacthub.io/packages/helm/ceph-csi/ceph-csi-rbd
```

导出`values.yaml`配置文件

```shell
helm show values ceph-csi/ceph-csi-rbd > ceph-csi-rbd.values.yaml
```

除了`quay.io/cephcsi/cephcsi`的镜像，其它镜像都改为`registry.aliyuncs.com`源

```
registry.aliyuncs.com/google_containers/csi-node-driver-registrar
registry.aliyuncs.com/google_containers/csi-provisioner
registry.aliyuncs.com/google_containers/csi-attacher
registry.aliyuncs.com/google_containers/csi-resizer
registry.aliyuncs.com/google_containers/csi-snapshotter
```

修改`values.yaml`配置文件

```xml
csiConfig:
    - clusterID: "1584b6f9-7a81-4c34-a735-3ff3c60320ba"
      monitors:
        - "192.168.20.5:6789"
        - "192.168.30.5:6789"
        - "192.168.40.5:6789"
```



```shell
[root@master ceph]# helm install --namespace ceph-csi-rbd ceph-csi-rbd ceph-csi/ceph-csi-rbd -f ceph-csi-rbd.values.yaml
NAME: ceph-csi-rbd
LAST DEPLOYED: Sat Mar  4 17:01:23 2023
NAMESPACE: ceph-csi-rbd
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Examples on how to configure a storage class and start using the driver are here:
https://github.com/ceph/ceph-csi/tree/v3.8.0/examples/rbd
# 查看安装有没有正确，如果不正确使用 describe 和 logs 命令查看具体出错信息
[root@master ceph]# kubectl get all -n ceph-csi-rbd
```

```shell
helm uninstall "ceph-csi-rbd" --namespace "ceph-csi-rbd" # 删除
```





### 参考：

[helm 部署mysql主从库](http://www.luckfu.com/post/2022-03-15_helm_deploy_mysql_master_slave/)

[k8s通过ceph-csi接入存储的概要分析](https://www.cnblogs.com/lianngkyle/p/14772121.html)

[Container Storage Interface 基本介紹](https://www.hwchiu.com/csi.html) -- 为什么要使用 csi？
# 模拟生产环境对 kubernetes 集群进行故障演练

### 1. 环境搭建
```shell script
[root@VM-18-7-centos ~]# kubectl  get node
NAME          STATUS   ROLES    AGE   VERSION
172.16.18.4   Ready    master   99m   v1.18.4
172.16.18.6   Ready    master   99m   v1.18.4
172.16.18.7   Ready    <none>   96m   v1.18.4
172.16.18.8   Ready    master   99m   v1.18.4
172.16.18.10  Ready    <none>   5m27s v1.18.4

[root@VM-18-7-centos ~]# kubectl create -f yamls/nginx-daemon-set.yaml
[root@VM-18-7-centos ~]# kubectl create -f yamls/nginx-deployment.yaml
[root@VM-18-7-centos ~]# kubectl create -f yamls/nginx-statefulset.yaml

[root@VM-18-7-centos ~]# kubectl  get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
nginx-daemon-set-8hlrs              1/1     Running   0          11m     172.22.1.2     172.16.18.10   <none>           <none>
nginx-daemon-set-hb5lk              1/1     Running   0          11m     172.22.0.199   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-8j7l8   1/1     Running   0          32m     172.22.0.195   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-ctplj   1/1     Running   0          32m     172.22.0.196   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-w8cqv   1/1     Running   0          32m     172.22.0.198   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-xwbzd   1/1     Running   0          32m     172.22.0.197   172.16.18.7    <none>           <none>
nginx-sts-0                         1/1     Running   0          3m17s   172.22.1.3     172.16.18.10   <none>           <none>
nginx-sts-1                         1/1     Running   0          3m1s    172.22.0.200   172.16.18.7    <none>           <none>
nginx-sts-2                         1/1     Running   0          2m50s   172.22.1.4     172.16.18.10   <none>           <none>
nginx-sts-3                         1/1     Running   0          2m41s   172.22.1.5     172.16.18.10   <none>           <none>
```


### 2. 故障模拟
#### 2.1 kubelet 停服5分钟,并观察集群状态,pod是否被删除

```shell script
[root@VM-18-7-centos ~]# systemctl stop kubelet
[root@VM-18-7-centos ~]# date
2020年 09月 03日 星期四 20:54:04 CST
[root@VM-18-7-centos ~]# kubectl  get node
172.16.18.7    NotReady   <none>   136m   v1.18.4
[root@VM-18-7-centos ~]# date
2020年 09月 03日 星期四 21:05:53 CST

查看pod状态
[root@VM-18-7-centos ~]# kubectl  get pod -o wide
NAME                                READY   STATUS        RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
nginx-daemon-set-8hlrs              1/1     Running       0          25m     172.22.1.2     172.16.18.10   <none>           <none>
nginx-daemon-set-hb5lk              1/1     Running       0          25m     172.22.0.199   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-4clw6   1/1     Running       0          5m46s   172.22.1.6     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-8j7l8   1/1     Terminating   0          46m     172.22.0.195   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-cb5sw   1/1     Running       0          5m46s   172.22.1.7     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-ctplj   1/1     Terminating   0          46m     172.22.0.196   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-dg997   1/1     Running       0          5m46s   172.22.1.8     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-hrssh   1/1     Running       0          5m46s   172.22.1.9     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-w8cqv   1/1     Terminating   0          46m     172.22.0.198   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-xwbzd   1/1     Terminating   0          46m     172.22.0.197   172.16.18.7    <none>           <none>
nginx-sts-0                         1/1     Running       0          17m     172.22.1.3     172.16.18.10   <none>           <none>
nginx-sts-1                         1/1     Terminating   0          17m     172.22.0.200   172.16.18.7    <none>           <none>
nginx-sts-2                         1/1     Running       0          17m     172.22.1.4     172.16.18.10   <none>           <none>
nginx-sts-3                         1/1     Running       0          16m     172.22.1.5     172.16.18.10   <none>           <none>

查看副本数
[root@VM-18-7-centos ~]# kubectl  get sts
NAME        READY   AGE
nginx-sts   3/4     27m

[root@VM-18-7-centos ~]# kubectl  get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           56m

查看service后端podIP
[root@VM-18-7-centos ~]# kubectl get endpoints nginx-sts
NAME        ENDPOINTS                                   AGE
nginx-sts   172.22.1.3:80,172.22.1.4:80,172.22.1.5:80   32s

[root@VM-18-7-centos ~]# kubectl describe endpoints nginx-deployment
Name:         nginx-deployment
Namespace:    default
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2020-09-03T21:07:09+08:00
Subsets:
  Addresses:          172.22.1.6,172.22.1.7,172.22.1.8,172.22.1.9
```

#### 测试结果

```text
1. deployment 的pod状态发生了变化，副本数不变，endpoints后端随之更新；
2. statefulset 的pod状态发生了变化，副本数收到影响,具体原因可以查看statefulset的特性，endpoints后端随之更新；
3. 长期处于 Terminating 状态的pod，是因为这些pod所在节点的kubelet停服了，无法执行删除动作，可以启动kubelet恢复；
```

-----

#### 2.2 controller-manager 停服5分钟,并观察集群状态,pod是否被删除
```shell script
[root@172-16-18-6 ~]# kubectl  get pod -n kube-system |grep kube-controller-manager
NAME                                  READY   STATUS    RESTARTS   AGE
kube-controller-manager-172.16.18.4   1/1     Running   0          176m
kube-controller-manager-172.16.18.6   1/1     Running   0          176m
kube-controller-manager-172.16.18.8   1/1     Running   0          176m

停止所有kube-controller-manager服务,根据每个用户的启动方式不同,操作可能不同

[root@VM-18-7-centos ~]# date
2020年 09月 03日 星期四 21:37:17 CST
...
[root@VM-18-7-centos ~]# date
2020年 09月 03日 星期四 21:47:00 CST

查看pod状态
[root@VM-18-7-centos ~]# kubectl  get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
nginx-daemon-set-8hlrs              1/1     Running   0          67m   172.22.1.2     172.16.18.10   <none>           <none>
nginx-daemon-set-hb5lk              1/1     Running   0          67m   172.22.0.199   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-4clw6   1/1     Running   0          47m   172.22.1.6     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-cb5sw   1/1     Running   0          47m   172.22.1.7     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-dg997   1/1     Running   0          47m   172.22.1.8     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-hrssh   1/1     Running   0          47m   172.22.1.9     172.16.18.10   <none>           <none>
nginx-sts-0                         1/1     Running   0          59m   172.22.1.3     172.16.18.10   <none>           <none>
nginx-sts-1                         1/1     Running   0          28m   172.22.0.201   172.16.18.7    <none>           <none>
nginx-sts-2                         1/1     Running   0          58m   172.22.1.4     172.16.18.10   <none>           <none>
nginx-sts-3                         1/1     Running   0          58m   172.22.1.5     172.16.18.10   <none>           <none>

删除pod
[root@VM-18-7-centos ~]# kubectl  delete pod nginx-deployment-74d794547c-4clw6
pod "nginx-deployment-74d794547c-4clw6" deleted
[root@VM-18-7-centos ~]# kubectl  get pod -o wide |grep  nginx-deployment
nginx-deployment-74d794547c-cb5sw   1/1     Running   0          49m   172.22.1.7     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-dg997   1/1     Running   0          49m   172.22.1.8     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-hrssh   1/1     Running   0          49m   172.22.1.9     172.16.18.10   <none>           <none>
[root@VM-18-7-centos ~]# kubectl  get deploy -o wide |grep  nginx-deployment
nginx-deployment   4/4     4            4           90m   nginx        nginx    app=nginx
[root@VM-18-7-centos ~]# kubectl  describe endpoints nginx-deployment
Name:         nginx-deployment
Namespace:    default
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2020-09-03T21:07:09+08:00
Subsets:
  Addresses:          172.22.1.6,172.22.1.7,172.22.1.8,172.22.1.9
```
#### 测试结果

```text
1. 所有pod状态没有发生变化；
2. 删除pod后，无法创建workload中的pod，但是需要注意的是endpoints后端并没有更新，所以此时对在线服务是有影响的；
```
进一步删除 deployment测试

```shell script
[root@VM-18-7-centos ~]# kubectl  delete deploy nginx-deployment
deployment.apps "nginx-deployment" deleted
查看pod状态
[root@VM-18-7-centos ~]# kubectl  get pod -o wide |grep nginx-deployment
nginx-deployment-74d794547c-cb5sw   1/1     Running   0          54m   172.22.1.7     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-dg997   1/1     Running   0          54m   172.22.1.8     172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-hrssh   1/1     Running   0          54m   172.22.1.9     172.16.18.10   <none>           <none>

启动controller-manager后
[root@VM-18-7-centos ~]# kubectl  get pod -o wide|grep nginx-deployment
无
```
测试结果总结：

```text
3. 删除deployment可以成功，但是无法自动删除pod；
4. controller-manager恢复启动后，将自动删除没有属主的pods；
```

#### PS: 通过该实验，要掌握 kube-controller-manager的工作原理

-----

#### 2.3 kube-scheduler 停服5分钟,并观察集群状态,pod是否被删除

```shell script
[root@172-16-18-6 ~]# kubectl  get pod -n kube-system |grep kube-scheduler
NAME                                  READY   STATUS    RESTARTS   AGE
kube-scheduler-172.16.18.4            1/1     Running   0          3h33m
kube-scheduler-172.16.18.6            1/1     Running   1          3h33m
kube-scheduler-172.16.18.8            1/1     Running   0          3h33m
```
停止所有kube-scheduler服务,根据每个用户的启动方式不同,操作可能不同

```shell script
[root@VM-18-7-centos ~]# date
2020年 09月 03日 星期四 22:28:18 CST
...
[root@VM-18-7-centos ~]# date
2020年 09月 03日 星期四 21:47:00 CST
```
查看pod状态

```shell script
[root@172-16-18-6 ~]# kubectl get pod -o wide |grep nginx-deployment
nginx-deployment-74d794547c-8njn2   1/1     Running   0          8d    172.22.0.203   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-wnrr8   1/1     Running   0          8d    172.22.1.11    172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-xj8lf   1/1     Running   0          8d    172.22.0.204   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-ztq48   1/1     Running   0          8d    172.22.1.10    172.16.18.10   <none>           <none>
```
删除pod

```shell script
[root@172-16-18-6 ~]# kubectl  delete pod nginx-deployment-74d794547c-8njn2
pod "nginx-deployment-74d794547c-8njn2" deleted
[root@172-16-18-6 ~]# kubectl get pod -o wide |grep nginx-deployment
nginx-deployment-74d794547c-khxvk   0/1     Pending   0          55s   <none>         <none>         <none>           <none>
nginx-deployment-74d794547c-wnrr8   1/1     Running   0          8d    172.22.1.11    172.16.18.10   <none>           <none>
nginx-deployment-74d794547c-xj8lf   1/1     Running   0          8d    172.22.0.204   172.16.18.7    <none>           <none>
nginx-deployment-74d794547c-ztq48   1/1     Running   0          8d    172.22.1.10    172.16.18.10   <none>           <none>

[root@172-16-18-6 ~]# kubectl  get deploy -o wide |grep  nginx-deployment
nginx-deployment   3/4     4            3           8d    nginx        nginx    app=nginx
```
查看服务后端列表

```shell script
[root@172-16-18-6 ~]# kubectl  describe endpoints nginx-deployment
Name:         nginx-deployment
Namespace:    default
Labels:       <none>
Annotations:  <none>
Subsets:
  Addresses:          172.22.0.204,172.22.1.10,172.22.1.11
```
#### 测试结果

```text
1. 所有 pod 状态没有发生变化；
2. 删除pod后，无法创建workload中的pod，但是 endpoints 后端发生了更新，所以此时对在线服务理论上是没有影响的；
```
进一步删除 deployment测试

```shell script
[root@VM-18-7-centos ~]# kubectl  delete deploy nginx-deployment
deployment.apps "nginx-deployment" deleted
查看pod状态
[root@VM-18-7-centos ~]# kubectl  get pod -o wide |grep nginx-deployment
无
```
测试结果总结：

```text
3. 删除 deployment 可以成功，pods 也被删除；
4. kube-scheduler 服务是否存活，不影响workloads或者pods的删除操作，只影响pod的创建；
```

#### 2.4 kube-apiserver 停服5分钟,并观察集群状态,pod是否被删除
```shell script
[root@172-16-18-6 ~]# kubectl get pod -n kube-system |grep kube-apiserver
kube-apiserver-172.16.18.4            1/1     Running   0          8d
kube-apiserver-172.16.18.6            1/1     Running   0          8d
kube-apiserver-172.16.18.8            1/1     Running   0          8d

[root@VM-18-7-centos ~]# kubectl create -f yamls/nginx-deployment.yaml

[root@172-16-18-8 ~]# kubectl  get svc|grep nginx-deployment
nginx-deployment   ClusterIP   172.22.253.18    <none>        80/TCP    8d

是否可以正常访问服务?
[root@172-16-18-8 ~]# curl -v 172.22.253.18:80
* About to connect() to 172.22.253.18 port 80 (#0)
*   Trying 172.22.253.18...
* Connected to 172.22.253.18 (172.22.253.18) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.22.253.18
> Accept: */*
>
< HTTP/1.1 200 OK

```
停止所有kube-apiserver, 根据每个用户的启动方式不同,操作可能不同

```shell script
[root@VM-18-7-centos ~]# date
2020年 09月 12日 星期六 16:22:01 CST
...
[root@172-16-18-6 ~]# date
2020年 09月 12日 星期六 16:39:34 CST
``` 
查看pod状态

```shell script
[root@172-16-18-6 ~]# kubectl  get pod
The connection to the server 172.16.18.6:60002 was refused - did you specify the right host or port?
``` 
访问业务服务是否正常

```shell script
# curl -v 172.22.253.18:80
* About to connect() to 172.22.253.18 port 80 (#0)
*   Trying 172.22.253.18...
* Connected to 172.22.253.18 (172.22.253.18) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.22.253.18
> Accept: */*
>
< HTTP/1.1 200 OK
```
恢复启动 kube-apiserver

```shell script
[root@172-16-18-6 ~]# kubectl get pod -n kube-system|grep kube-apiserver
kube-apiserver-172.16.18.4            1/1     Running   0          8d
kube-apiserver-172.16.18.6            1/1     Running   0          8d
kube-apiserver-172.16.18.8            1/1     Running   0          8d

[root@172-16-18-6 ~]# kubectl get pod -o wide |grep nginx-deployment
nginx-deployment-5dbbd985f5-4vxxr   1/1     Running   0          24m   172.22.0.205   172.16.18.7    <none>           <none>
nginx-deployment-5dbbd985f5-g8bsh   1/1     Running   0          24m   172.22.1.13    172.16.18.10   <none>           <none>
nginx-deployment-5dbbd985f5-k9rxg   1/1     Running   0          24m   172.22.0.206   172.16.18.7    <none>           <none>
nginx-deployment-5dbbd985f5-shjd6   1/1     Running   0          24m   172.22.1.12    172.16.18.10   <none>           <none>
```

#### 测试结果

```text
1. 恢复kube-apiserver服务后，发现所有 pod 状态没有发生变化；
2. kube-apiserver服务停止之后，应用curl命令，通过clusterIP访问nginx服务，服务可以正常访问，所以kube-apiserver故障时对存量在线服务理论上是没有影响的；
3. kube-apiserver只影响集群之间组件的以及外部访问 k8s api服务，管理员无法通过kubectl管理和访问k8s集群。
```

### 3. 对 etcd 集群停掉1台，2台，3台后，观察k8s 集群状态；

查看etcd 集群状态

```shell script
[root@172-16-18-4 ~]# kubectl  get pod -n kube-system |grep etcd
etcd-172.16.18.4                      1/1     Running   0          8d
etcd-172.16.18.6                      1/1     Running   0          8d
etcd-172.16.18.8                      1/1     Running   0          8d
```
停掉 1台 etcd 实例

```shell script
[root@172-16-18-4 ~]# kubectl  get pod -n kube-system |grep etcd
etcd-172.16.18.6                      1/1     Running   0          8d
etcd-172.16.18.8                      1/1     Running   0          8d
```
查看集群中pods状态

```shell script
[root@172-16-18-4 ~]# kubectl  get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5dbbd985f5-4vxxr   1/1     Running   0          43m
nginx-deployment-5dbbd985f5-g8bsh   1/1     Running   0          43m
nginx-deployment-5dbbd985f5-k9rxg   1/1     Running   0          43m
nginx-deployment-5dbbd985f5-shjd6   1/1     Running   0          43m
[root@172-16-18-4 ~]# kubectl run nginx --image=nginx
pod/nginx created
[root@172-16-18-4 ~]# kubectl  get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx                               1/1     Running   0          4s
```
停掉 2台 etcd 实例

```shell script
[root@172-16-18-4 ~]# kubectl  get pod -n kube-system |grep etcd
Error from server: etcdserver: request timed out

[root@172-16-18-6 ~]# kubectl run nginx-2 --image=nginx
Error from server (Forbidden): pods "nginx-2" is forbidden: etcdserver: request timed out
```
访问nginx服务

```shell script
curl -v 172.22.253.18:80
2020年 09月 12日 星期六 17:19:59 CST
* About to connect() to 172.22.253.18 port 80 (#0)
*   Trying 172.22.253.18...
* Connected to 172.22.253.18 (172.22.253.18) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.22.253.18
> Accept: */*
>
< HTTP/1.1 200 OK
```

停掉 3台 etcd 实例

```shell script
[root@172-16-18-4 ~]# kubectl  get pod -n kube-system |grep etcd
Error from server: etcdserver: request timed out

[root@172-16-18-6 ~]# kubectl run nginx-2 --image=nginx
Error from server (Forbidden): pods "nginx-2" is forbidden: etcdserver: request timed out
```
访问 nginx 服务

```shell script
[root@172-16-18-8 ~]# curl -v 172.22.253.18:80
* About to connect() to 172.22.253.18 port 80 (#0)
*   Trying 172.22.253.18...
* Connected to 172.22.253.18 (172.22.253.18) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.22.253.18
> Accept: */*
>
< HTTP/1.1 200 OK
```

#### 测试结果
```text
1. 3台etcd实例的集群，停止其中一台etcd实例，k8s集群操作不受影响，集群中的服务也不受影响；
2. 3台etcd实例的集群，停止其中两台或3台etcd实例，访问etcd集群超时，k8s集群操作受到影响，访问集群中的服务不受影响。
```

### 4. 对 etcd 存储数据进行毁坏性测试，比如把 etcd 数据存储目录删除，观察k8s 集群状态
查看etcd 存储目录的配置, 有可能有不同的配置,比如 --data-dir=/var/lib/etcd

```shell script
[root@172-16-18-4 ~]# cat /etc/kubernetes/manifests/etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  name: etcd
  namespace: kube-system
spec:
  containers:
  - args:
    - --listen-client-urls=https://172.16.18.4:2379
    - --name=172.16.18.4
    - --data-dir=/var/lib/etcd
```
查看当前etcd 实例
```shell script
[root@172-16-18-4 ~]# kubectl  get pod -n kube-system|grep etcd
etcd-172.16.18.4                      1/1     Running   1          14m
etcd-172.16.18.6                      1/1     Running   0          17m
etcd-172.16.18.8                      1/1     Running   0          9d
```

删除 etcd-172.16.18.4 这1台 etcd 存储目录:

```shell script
[root@172-16-18-4 ~]# mv /var/lib/etcd/member/ ./

[root@172-16-18-4 ~]# kubectl  run nginx-02 --image=nginx
pod/nginx-02 created
[root@172-16-18-4 ~]# kubectl  get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-02                            1/1     Running   0          3s
[root@172-16-18-4 ~]# kubectl delete pod nginx-02
pod "nginx-02" deleted
```
删除一个etcd实例的存储目录之后，k8s集群状态正常，可以正常创建和删除pod；

再次删除 etcd-172.16.18.6 这台 etcd 存储目录，共两个etcd实例的存储目录销毁:

```shell script
[root@172-16-18-6 ~]# mv /var/lib/etcd/member/ ./
[root@172-16-18-4 ~]# kubectl  run nginx-02 --image=nginx
Error from server (Forbidden): pods "nginx-02" is forbidden: etcdserver: request timed out
[root@172-16-18-6 ~]# kubectl  get pod
Error from server: etcdserver: request timed out
查看kube-apiserver的日志
[root@172-16-18-6 ~]# docker logs a48e087ce578
E0912 22:26:59.856579       1 status.go:71] apiserver received an error that is not an metav1.Status: rpctypes.EtcdError{code:0xe, desc:"etcdserver: request timed out"}
I0912 22:26:59.856645       1 trace.go:116] Trace[1506359465]: "GuaranteedUpdate etcd3" type:*core.Event (started: 2020-09-12 22:26:45.860928777 +0800 CST m=+2227.125466433) (total time: 13.995623531s):
Trace[1506359465]: [13.995623531s] [13.995623531s] END
E0912 22:26:59.856674       1 status.go:71] apiserver received an error that is not an metav1.Status: rpctypes.EtcdError{code:0xe, desc:"etcdserver: request timed out"}
E0912 22:26:59.856769       1 status.go:71] apiserver received an error that is not an metav1.Status: rpctypes.EtcdError{code:0xe, desc:"etcdserver: request timed out"}
访问k8s集群中的服务:
[root@172-16-18-6 ~]# curl -v 172.22.253.18:80
* About to connect() to 172.22.253.18 port 80 (#0)
*   Trying 172.22.253.18...
* Connected to 172.22.253.18 (172.22.253.18) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 172.22.253.18
> Accept: */*
>
< HTTP/1.1 200 OK
```
此时k8s集群访问超时，查看kube-apiserver日志发现有大量的超时请求，但是访问 k8s 集群中的存量服务，是可以正常访问的；

#### 测试结果
```text
1. 3台etcd实例的集群，毁坏其中一台etcd实例的存储目录，k8s集群操作不受影响，集群中的服务也不受影响；
2. 3台etcd实例的集群，毁坏其中两台或3台etcd实例，访问etcd集群超时，k8s集群操作受到影响，但是访问集群中的服务不受影响。
```

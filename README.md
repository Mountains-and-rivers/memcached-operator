# memcached-operator

## operator 概述



## 常见 Operator



### kubebuilder 与 operator-sdk

使用 SDK 创建一个新的 Operator 项目
通过添加自定义资源（CRD）定义新的资源 API
指定使用 SDK API 来 watch 的资源
定义 Operator 的协调（reconcile）逻辑
使用 Operator SDK 构建并生成 Operator 部署清单文件

#### kuberbuilder demo



#### operator-sdk demo

编译与安装 operator-framework/operator-sdk。具体步骤请参考 [install-the-operator-sdk-cli](https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md#install-the-operator-sdk-cli)，
此次试验是按照 [compile-and-install-from-master](https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md#compile-and-install-from-master) 方式安装的 operator-sdk。

```
git clone https://github.com/operator-framework/operator-sdk
cd operator-sdk
git checkout master
make tidy
make install
```

operator-sdk version

```
operator-sdk version: "v0.15.0-79-g11aa2af7", commit: "11aa2af7aaa202cb38b2417031e6af2035c7948c", go version: "go1.13.4 linux/amd64"
```

go version

```
go version go1.13.4 linux/amd64
```

使用 operator-sdk 创建 memcached-operator 项目：
```
mkdir -p $HOME/projects/memcached-operator

cd $HOME/projects/memcached-operator

operator-sdk init --domain example.com --repo github.com/Mountains-and-rivers/memcached-operatorr

cd memcached-operator
```

注意：在 $GOPATH/src 外部创建项目时，需要 --repo=<path> 参数。

使用 apiVersion `cache.example.com/v1alpha1` 和 kind `Memcached` 创建名为 `Memcached` 的自定义资源(CRD) API。

`operator-sdk add api --api-version=cache.example.com/v1alpha1 --kind=Memcached`

这将在 `pkg/apis/cache/v1alpha1/` 下创建 `Memcached` 资源 API go 文件。

修改 `pkg/apis/cache/v1alpha1/memcached_types.go` 文件，添加以下内容：

```
type MemcachedSpec struct {
	// Size is the size of the memcached deployment
	Size int32 `json:"size"`
}
type MemcachedStatus struct {
	// Nodes are the names of the memcached pods
	Nodes []string `json:"nodes"`
}
```

修改 `*_types.go` 文件后，运行以下命令更新该资源类型的自动生成代码：

`operator-sdk generate k8s`

更新 CRD 清单：

`operator-sdk generate crds`

添加一个新的 Controller 监视(watch)并协调(reconcile) `Memcached` 资源：

`operator-sdk add controller --api-version=cache.example.com/v1alpha1 --kind=Memcached`

Controller 主要通过 Watch 与 Reconcile loop(控制环) 实现资源控制。

更改 `memcached_controller.go` 相关代码，如[memcached_controller.go](https://github.com/tanjunchen/memcached-operator/blob/master/pkg/controller/memcached/memcached_controller.go)。

有两种方法可以运行 operator：

    作为 Kubernetes 集群内部的 Deployment
    作为集群外的 Go 程序

我这里部署到 Kubernetes 集群中。

执行以下步骤：

    operator-sdk build tanjunchen/memcached-operator:v1
    sed -i 's|REPLACE_IMAGE|tanjunchen/memcached-operator:v1|g' deploy/operator.yaml

构建相关 operator 镜像，注意目前 `deploy/operator.yaml` 中的 tanjunchen/memcached-operator:v1 镜像远程不能访问。需要自己替换镜像地址。

设置 RBAC 并部署 memcached-operator：

```
kubectl create -f deploy/service_account.yaml,deploy/role.yaml,deploy/role_binding.yaml,deploy/operator.yaml
```

kubectl get deployment  -owide | grep memcached-operator

```
memcached-operator   1/1     1            1           4h55m   memcached-operator   tanjunchen/memcached-operator:v1   name=memcached-operator
```

cat deploy/crds/cache.example.com_v1alpha1_memcached_cr.yaml

```yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  name: example-memcached
spec:
  # Add fields here
  size: 3
```

kubectl apply -f deploy/crds/cache.example.com_v1alpha1_memcached_cr.yaml

kubectl get deployment

```
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
example-memcached    3/3     3            3           4h58m
memcached-operator   1/1     1            1           4h58m
```

kubectl get pods

```
NAME                                  READY   STATUS    RESTARTS   AGE
example-memcached-7c4df9b7b4-5t8lb    1/1     Running   0          5h
example-memcached-7c4df9b7b4-6v9nm    1/1     Running   0          4h59m
example-memcached-7c4df9b7b4-h9fmd    1/1     Running   0          5h
memcached-operator-686f9b7c9b-2fwfs   1/1     Running   0          5h
```

kubectl get memcached/example-memcached -o yaml

```
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"cache.example.com/v1alpha1","kind":"Memcached","metadata":{"annotations":{},"name":"example-memcached","namespace":"default"},"spec":{"size":3}}
  creationTimestamp: "2020-03-17T09:31:21Z"
  generation: 8
  name: example-memcached
  namespace: default
  resourceVersion: "135559"
  selfLink: /apis/cache.example.com/v1alpha1/namespaces/default/memcacheds/example-memcached
  uid: 3740a34d-3321-4759-b7e9-5befce29b428
spec:
  size: 3
status:
  nodes:
  - example-memcached-7c4df9b7b4-8fcd4
  - example-memcached-7c4df9b7b4-h9fmd
  - example-memcached-7c4df9b7b4-5t8lb
  - example-memcached-7c4df9b7b4-6v9nm
  - example-memcached-7c4df9b7b4-f82z2
```

上面 size 大小与 node 节点数不一致是因为我先将 `deploy/crds/cache.example.com_v1alpha1_memcached_cr.yaml` 中的 size 改为 5,然后减为 3。而没有在 controller 中添加相关操作事件。

kubectl logs memcached-operator-686f9b7c9b-2fwfs

```
{"level":"info","ts":1584439298.7350085,"logger":"cmd","msg":"Operator Version: 0.0.1"}
{"level":"info","ts":1584439298.7378504,"logger":"cmd","msg":"Go Version: go1.13.4"}
{"level":"info","ts":1584439298.7378654,"logger":"cmd","msg":"Go OS/Arch: linux/amd64"}
{"level":"info","ts":1584439298.7378726,"logger":"cmd","msg":"Version of operator-sdk: v0.15.0+git"}
{"level":"info","ts":1584439298.7380009,"logger":"leader","msg":"Trying to become the leader."}
{"level":"info","ts":1584439299.2666407,"logger":"leader","msg":"No pre-existing lock was found."}
{"level":"info","ts":1584439299.269379,"logger":"leader","msg":"Became the leader."}
{"level":"info","ts":1584439299.774637,"logger":"controller-runtime.metrics","msg":"metrics server is starting to listen","addr":"0.0.0.0:8383"}
{"level":"info","ts":1584439299.7766497,"logger":"cmd","msg":"Registering Components."}
{"level":"info","ts":1584439300.7996862,"logger":"metrics","msg":"Metrics Service object created","Service.Name":"memcached-operator-metrics","Service.Namespace":"default"}
{"level":"info","ts":1584439301.3024964,"logger":"cmd","msg":"Could not create ServiceMonitor object","error":"no ServiceMonitor registered with the API"}
{"level":"info","ts":1584439301.3025157,"logger":"cmd","msg":"Install prometheus-operator in your cluster to create ServiceMonitor objects","error":"no ServiceMonitor registered with the API"}
{"level":"info","ts":1584439301.3025324,"logger":"cmd","msg":"Starting the Cmd."}
{"level":"info","ts":1584439301.3038416,"logger":"controller-runtime.controller","msg":"Starting EventSource","controller":"memcached-controller","source":"kind source: /, Kind="}
{"level":"info","ts":1584439301.303985,"logger":"controller-runtime.controller","msg":"Starting EventSource","controller":"memcached-controller","source":"kind source: /, Kind="}
{"level":"info","ts":1584439301.3040295,"logger":"controller-runtime.controller","msg":"Starting Controller","controller":"memcached-controller"}
{"level":"info","ts":1584439301.304537,"logger":"controller-runtime.manager","msg":"starting metrics server","path":"/metrics"}
{"level":"info","ts":1584439301.404253,"logger":"controller-runtime.controller","msg":"Starting workers","controller":"memcached-controller","worker count":1}
2020-03-17 10:01:41.404451307 +0000 UTC m=+2.726210870 ......Kubernetes Operator Framework Reconcile......
{"level":"info","ts":1584439301.4079845,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
{"level":"info","ts":1584439301.5083523,"logger":"controller_memcached","msg":"Creating a new Deployment","Request.Namespace":"default","Request.Name":"example-memcached","Deployment.Namespace":"default","Deployment.Name":"example-memcached"}
2020-03-17 10:01:41.580843209 +0000 UTC m=+2.902602770 ......Kubernetes Operator Framework Reconcile......
2020-03-17 10:01:41.580921058 +0000 UTC m=+2.902680627 ......Kubernetes Operator Framework Update Pod Name ......
{"level":"info","ts":1584439301.580856,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
2020-03-17 10:01:41.584898031 +0000 UTC m=+2.906657592 ......Kubernetes Operator Framework Reconcile......
{"level":"info","ts":1584439301.5849097,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
2020-03-17 10:02:35.634080794 +0000 UTC m=+56.955840354 ......Kubernetes Operator Framework Reconcile......
{"level":"info","ts":1584439355.6340961,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
2020-03-17 10:02:35.711503159 +0000 UTC m=+57.033262720 ......Kubernetes Operator Framework Reconcile......
2020-03-17 10:02:35.711948775 +0000 UTC m=+57.033708345 ......Kubernetes Operator Framework Update Pod Name ......
{"level":"info","ts":1584439355.711521,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
2020-03-17 10:02:35.728754702 +0000 UTC m=+57.050514262 ......Kubernetes Operator Framework Reconcile......
2020-03-17 10:02:35.72883959 +0000 UTC m=+57.050599159 ......Kubernetes Operator Framework Update Pod Name ......
{"level":"info","ts":1584439355.728768,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
2020-03-17 10:02:35.732398693 +0000 UTC m=+57.054158255 ......Kubernetes Operator Framework Reconcile......
{"level":"info","ts":1584439355.7324111,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
2020-03-17 10:03:40.896113823 +0000 UTC m=+122.217873384 ......Kubernetes Operator Framework Reconcile......
{"level":"info","ts":1584439420.8961298,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
2020-03-17 10:03:40.9053012 +0000 UTC m=+122.227060761 ......Kubernetes Operator Framework Reconcile......
2020-03-17 10:03:40.905427776 +0000 UTC m=+122.227187345 ......Kubernetes Operator Framework Update Pod Name ......
{"level":"info","ts":1584439420.9053237,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
2020-03-17 10:03:40.940342735 +0000 UTC m=+122.262102295 ......Kubernetes Operator Framework Reconcile......
2020-03-17 10:03:40.940425961 +0000 UTC m=+122.262185531 ......Kubernetes Operator Framework Update Pod Name ......
{"level":"info","ts":1584439420.940356,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
2020-03-17 10:03:40.970526685 +0000 UTC m=+122.292286246 ......Kubernetes Operator Framework Reconcile......
{"level":"info","ts":1584439420.9732163,"logger":"controller_memcached","msg":"Reconciling Memcached","Request.Namespace":"default","Request.Name":"example-memcached"}
```

清理资源：

```
kubectl delete -f deploy/crds/cache.example.com_v1alpha1_memcached_cr.yaml,deploy/operator.yaml,deploy/role_binding.yaml,deploy/role.yaml,deploy/service_account.yaml
```

**高级进阶与源码分析**

## 参考

https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md#compile-and-install-from-master

https://github.com/operator-framework


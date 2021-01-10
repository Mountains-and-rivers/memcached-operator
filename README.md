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
operator-sdk version: "v1.3.0", commit: "1abf57985b43bf6a59dcd18147b3c574fa57d3f6", kubernetes version: "1.19.4", go version: "go1.15.5", GOOS: "linux", GOARCH: "amd64"
```

go version

```
go version go1.15.6 linux/amd64
```

使用 operator-sdk 创建 memcached-operator 项目：
```
mkdir -p $HOME/projects/memcached-operator

cd $HOME/projects/memcached-operator

operator-sdk init --domain example.com --repo github.com/Mountains-and-rivers/memcached-operator

cd memcached-operator
```

注意：在 $GOPATH/src 外部创建项目时，需要 --repo=<path> 参数。

使用 apiVersion `v1alpha1` 和 kind `Memcached` 创建名为 `Memcached` 的自定义资源(CRD) API。

`operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller`

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

`make generate`

更新 CRD 清单：

`make manifests`

添加一个新的 api，Controller 监视(watch)并协调(reconcile) `Memcached` 资源：

operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller`

Controller 主要通过 Watch 与 Reconcile loop(控制环) 实现资源控制。

更改 `memcached_controller.go` 相关代码，如[memcached_controller.go](https://github.com/Mountains-and-rivers/memcached-operator/blob/master/controllers/memcached_controller.go)。

有两种方法可以运行 operator：

    作为 Kubernetes 集群内部的 Deployment
    作为集群外的 Go 程序

我这里部署到 Kubernetes 集群中。

执行以下步骤：

这里用docker官方仓库，先登录才能push成功
make docker-build docker-push IMG=memcached-operator:v0.0.1

部署 memcached-operator：

```
make deploy IMG=memcached-operator:v0.0.1
这里要修改deploy中的镜像名称
kubectl get deployment -n projects-system
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
projects-controller-manager   0/1     1            0           63s

修改如下仓库地址

搜索image字段修改
...
,"image":"mangseng/memcached-operator:v0.0.1"
...
...
image: memcached-operator:v0.0.1
imagePullPolicy: IfNotPresent
livenessProbe:
  failureThreshold: 3
...
```

 kubectl get deployment -o wide -n projects-system

```
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS                IMAGES                                                                         SELECTOR
projects-controller-manager   1/1     1            1           6m41s   kube-rbac-proxy,manager   gcr.io/kubebuilder/kube-rbac-proxy:v0.5.0,mangseng/memcached-operator:v0.0.1   control-plane=controller-mana
```

cat config/samples/cache_v1alpha1_memcached.yaml

```yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  name: memcached-sample
spec:
  size: 3
```

kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml

 kubectl get deployment --all-namespaces

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
NAMESPACE         NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
default           memcached-sample              3/3     3            3           102s
kube-system       kube-dns                      1/1     1            1           12m
projects-system   projects-controller-manager   1/1     1            1           12m
```

kubectl get pods

```
NAME                                  READY   STATUS    RESTARTS   AGE
memcached-sample-6c765df685-68s4p   1/1     Running   0          2m14s
memcached-sample-6c765df685-6ssgv   1/1     Running   0          2m14s
memcached-sample-6c765df685-ltjjx   1/1     Running   0          2m14s
```

kubectl get memcached/memcached-sample -o yaml

```
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"cache.example.com/v1alpha1","kind":"Memcached","metadata":{"annotations":{},"name":"memcached-sample","namespace":"default"},"spec":{"size":3}}
  creationTimestamp: "2021-01-10T04:05:32Z"
  generation: 1
  managedFields:
  - apiVersion: cache.example.com/v1alpha1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
      f:spec:
        .: {}
        f:size: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2021-01-10T04:05:32Z"
  - apiVersion: cache.example.com/v1alpha1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        .: {}
        f:nodes: {}
    manager: manager
    operation: Update
    time: "2021-01-10T04:05:46Z"
  name: memcached-sample
  namespace: default
  resourceVersion: "1071"
  uid: b738b193-129d-402c-b375-d7d2973d8412
spec:
  size: 3
status:
  nodes:
  - memcached-sample-6c765df685-ltjjx
  - memcached-sample-6c765df685-6ssgv
  - memcached-sample-6c765df685-68s4p
```

修改pod副本数
```
kubectl patch memcached memcached-sample -p '{"spec":{"size": 5}}' --type=merge
kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
memcached-sample   5/5     5            5           4m55s

```

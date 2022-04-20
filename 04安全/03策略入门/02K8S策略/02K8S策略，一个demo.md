# Kubernetes策略，一个demo

这个demo中包括一个前端服务和一个后端服务，还有一个客户端服务，都跑在Kubernetes中。然后给每个服务配置它们的网络策略。

## 前提

创建一个Kubernetes集群，支持Kubernets网络策略API，参照[入门指引](../../../02%E5%AE%89%E8%A3%85Calico/00%E5%AE%89%E8%A3%85Calico.md)。

## 运行小星星

### 1)创建前端、后端、客户端、管理ui。

```shell
kubectl create -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/manifests/00-namespace.yaml
kubectl create -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/manifests/01-management-ui.yaml
kubectl create -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/manifests/02-backend.yaml
kubectl create -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/manifests/03-frontend.yaml
kubectl create -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/manifests/04-client.yaml
```

等待所有Pod变成`Running`。

```shell
$ kubectl get pods --all-namespaces --watch
```

> 这里可能要多等几分钟，要下载相关的Docker镜像。

这里面的管理UI是一个`NodePort`类型的Service，它可以展示demo中各个服务的连通性。

可以在浏览器通过`http://<k8s-node-ip>:30002`地址访问这个UI。

所有Pod都起来之后，它们应该是完全互通的。可以通过UI来观察到这一点。图中每个服务都表示成了一个单独的节点。

- `backend` -> 节点“B”
- `frontend` -> 节点“F”
- `client` -> 节点“C”

### 2)开启隔离

执行下面的命令，拒掉所有对前端、后端、客户端服务的访问。

```shell
kubectl create -n stars -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/policies/default-deny.yaml
kubectl create -n client -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/policies/default-deny.yaml
```

#### 确认隔离

刷新管理UI（变更信息同步到UI界面上需要等一会儿，最多也就是10秒钟）。既然我们开启了隔离，UI就无法访问到Pod，所以就不会出现在UI中了。

### 3)通过网络策略对象放行UI对服务的访问

执行下面的YAML，允许来自管理UI的访问。

```shell
kubectl create -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/policies/allow-ui.yaml
kubectl create -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/policies/allow-ui-client.yaml
```

等个几秒，刷新UI——现在应该能看到服务了，但是它们之间应该是无法互通的。

### 4)创建backend-policy.yaml文件允许从前端到后端的流量

```shell
kubectl create -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/policies/backend-policy.yaml
```

刷新UI。你应该可以看到：

- 前端可以访问后端（只能访问TCP的6379端口）。
- 后端无法访问前端。
- 客户端无法访问前端，也不能访问后端。

### 5)将前端服务暴露到客户端的命名空间中

```shell
kubectl create -f https://projectcalico.docs.tigera.io/security/tutorials/kubernetes-policy-demo/policies/frontend-policy.yaml
```

客户端现在可以访问前端了，但是访问不了后端。前端和后端也都无法发起向客户端的连接。前端仍然可以访问后端。

如果用Calico为Pod添加egress策略，请看[K8S策略，高级教程](04K8S策略，高级教程.md)。

### 6)（可选）清理demo环境

直接删掉demo的命名空间：

```shell
$ kubectl delete ns client stars management-ui
```
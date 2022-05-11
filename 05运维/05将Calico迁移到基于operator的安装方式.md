# 将Calico迁移到operator模式

## 大面儿

将你的Calico从manifest模式迁移到operator。

## 价值

Calico的operator比一般的manifest模式更有优势，包括但不限于：

- 自动化平台及配置探测。
- 升级过程简化。
- 很好地区分了端用户配置和产品代码。
- 资源协调及生命周期管理。

## 概念

### Operator vs Manifest

以前大部分Calico都是基于manifest安装的，意味着将Calico直接作为Kubernetes的一系列资源通过`.yaml`文件进行安装。

Calico operator是一个Kubernetes应用，负责安装并管理Calico的生命周期，它创建并更新一系列的Kubernetes资源，包括Deployment、DaemonSet、Secret，无需用户介入。

有几个关键的地方需要知道，如果你已经熟悉了manifest安装并且正在想改成operator：

- 原来被manifest安装在`kube-system`命名空间中的Calico资源会被迁移到`calico-system`命名空间中。
- Calico资源不再是可以手动编辑的了，因为Calico operator会还原不应有的改动，维持它所期望的状态。
- 现在要通过`operator.tigera.io`的API来配置Calico资源。

### 迁移

对于新集群，直接按照[快速指引](../02%E5%AE%89%E8%A3%85Calico/01Kubernetes/01%E5%BF%AB%E9%80%9F%E5%BC%80%E5%A7%8B.md)中的说明安装operator即可。

对于老集群，使用了`calico.yaml`这种manifest的安装模式，当你安装operator时，它会探测到集群中已有的Calico资源并研究如何接管它们。如果可以支持，Operator会保持已有的定制化内容，并且针对那些无法支持的配置给出警告。

## 开始之前

- 确保你的Calico使用的是Kubernetes数据存储。如果是直连etcdv3的，必须在开始之前参照[迁移指南](04从etcd迁移到Kubernetes数据存储.md)进行操作。

## 怎么弄

### 迁移到operator

> 注意：在下面的操作进行过程中，不要编辑或删除任何`kube-system`命名空间中的资源，否则可能会导致升级失败。

1. 安装Tigera Calico operator以及CRD。

```shell
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
```

2. 创建一个`Installation`资源来触发operator迁移。Operator会自动探测已有的Calico设置并填入下面的spec中。

```yaml
kubectl create -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec: {}
EOF
```

3. 监控迁移过程：

```shell
kubectl describe tigerastatus calico
```

4. 现在已经迁移完了，你会发现Calico资源都被移动到了`calico-system`命名空间中。

```shell
kubectl get pods -n calico-system
```

看到类似下面的内容：

```text
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7688765788-9rqht   1/1     Running   0          17m
calico-node-4ljs6                          1/1     Running   0          14m
calico-node-bd8mc                          1/1     Running   0          14m
calico-node-cpbd8                          1/1     Running   0          14m
calico-node-jl97q                          1/1     Running   0          14m
calico-node-xw2nj                          1/1     Running   0          14m
calico-typha-57bf79f96f-6sk8x              1/1     Running   0          14m
calico-typha-57bf79f96f-g99s9              1/1     Running   0          14m
calico-typha-57bf79f96f-qtchs              1/1     Running   0          14m
```

现在，Operator会自动清理`kube-system`命名空间中的Calico资源。不需要手动干预。
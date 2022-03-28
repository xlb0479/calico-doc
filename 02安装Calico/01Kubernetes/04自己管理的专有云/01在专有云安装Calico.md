# 为专有云部署安装Calico网络及网络策略

## 大面儿

为自己管理的专有云部署安装Calico，提供网络及网络策略。

## 价值

**Calico网络**和**网络策略**是CaaS实现中的一把利器。如果你具备管理专有Kubernetes的网络设施和资源，安装完整的Calico产品可以提供最大限度的定制化和控制能力。

## 特性

本堂课我们使用以下Calico特性：

- **calico/node**
- **Typha**

## 概念

### **Calico manifests**

Calico为轻松定制提供了manifests。每个manifest包含了Kubernetes集群每个节点安装Calico必备的资源。在安装之前你可能想[定制Calico manifests](02%E8%87%AA%E5%AE%9A%E4%B9%89manifests.md)。

## 开始之前

- 确保你的Kubernetes集群满足[要求](../14%E7%B3%BB%E7%BB%9F%E8%A6%81%E6%B1%82.md)。如果你还没有集群，请参考[安装kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)。

## 怎么做

- [选择数据存储](#选择数据存储)
- [在节点上安装Calico](#在节点上安装Calico)

### **选择数据存储**

我们推荐的是**Kubernetes API数据存储**。

> 注意：对于全新安装不推荐使用**etcd**数据库。但如果你用Calico作为OpenStack和Kubernetes共同的网络插件，那么可以用。

### **在节点上安装Calico**

根据你的数据存储和节点数量，选择下面的一个链接来安装Calico。

> 注意：**Kubernetes API数据存储，大于50节点**这个选项用[Typha守护进程](../../../06%E5%8F%82%E8%80%83/08Typha/00Typha.md)提供伸缩性。用etcd的话不会包含Typha，因为etcd已经可以解决大量客户端的问题，用Typha就显得冗余了，所以不推荐。

- [Kubernetes API数据存储，小于等于50节点](#Kubernetes%20API数据存储，小于等于50节点)
- [Kubernetes API数据存储，大于50节点](#Kubernetes%20API数据存储，大于50节点)
- [etcd数据存储](#etcd数据存储)

#### **Kubernetes API数据存储，小于等于50节点**

1. 为Kubernetes API数据存储下载Calico网络manifest。
```shell
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
```
2. 如果你用的Pod CIDR是`192.168.0.0/16`，那么直接跳过这一步。如果用kubeadm安装时配置了不同的CIDR，不需要做改动——Calico会根据运行时配置自动探测到CIDR。对于其他平台，确保在manifest中打开CALICO_IPV4POOL_CIRD变量的注释，把它跟你的Pod CIDR设置成一样的。
3. 根据需要定制manifest。
4. 执行下面的命令来应用manifest。
```shell
kubectl apply -f calico.yaml
```

[Calico部署选项](/Calico%E9%83%A8%E7%BD%B2%E9%80%89%E9%A1%B9.md)

#### **Kubernetes API数据存储，大于50节点**

1. 为Kubernetes API数据存储下载Calico网络manifest。
```shell
curl https://projectcalico.docs.tigera.io/manifests/calico-typha.yaml -o calico.yaml
```
2. 如果你用的Pod CIDR是`192.168.0.0/16`，那么直接跳过这一步。如果用kubeadm安装时配置了不同的CIDR，不需要做改动——Calico会根据运行时配置自动探测到CIDR。对于其他平台，确保在manifest中打开CALICO_IPV4POOL_CIRD变量的注释，把它跟你的Pod CIDR设置成一样的。
3. 根据需要修改名为`calico-typha`的`Deployment`中的副本数。
```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: calico-typha
  ...
spec:
  ...
  replicas: <副本数>
```
我们建议每200个节点至少要有一个副本，最多不超过20个副本。在生产环境中我们建议至少使用三个副本，减少滚动更新和异常发生时的影响。副本的数量应当总是小于节点的数量，否则滚动更新时就会卡住，Typha只有在其实例数小于节点数时才能提供伸缩性。
> 警告：如果你设置了`typha_service_name`并且将Typha Deployment的副本数设置成了0，那么Felix就不会启动。
4. 根据需要定制manifest。
5. 应用manifest。
```shell
kubectl apply -f calico.yaml
```

[Calico部署选项](/Calico%E9%83%A8%E7%BD%B2%E9%80%89%E9%A1%B9.md)

#### **etcd数据存储**

> 注意：对于全新安装不推荐使用**etcd**数据库。但如果你用Calico作为OpenStack和Kubernetes共同的网络插件，那么可以用。

1. 为Kubernetes API数据存储下载Calico网络manifest。
```shell
curl https://projectcalico.docs.tigera.io/manifests/calico-etcd.yaml -o calico.yaml
```
2. 如果你用的Pod CIDR是`192.168.0.0/16`，那么直接跳过这一步。如果用kubeadm安装时配置了不同的CIDR，不需要做改动——Calico会根据运行时配置自动探测到CIDR。对于其他平台，确保在manifest中打开CALICO_IPV4POOL_CIRD变量的注释，把它跟你的Pod CIDR设置成一样的。
3. 在名为`calico-config`的`ConfigMap`中，将`etcd_endpoints`的设置为etcd服务的IP和端口。
> 提示：可以用逗号间隔的方式设置多个`etcd_endpoint`。
4. 根据需要定制manifest。
5. 应用manifest。
```shell
kubectl apply -f calico.yaml
```

[Calico部署选项](/Calico%E9%83%A8%E7%BD%B2%E9%80%89%E9%A1%B9.md)
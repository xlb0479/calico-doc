# Calico数据存储

Calico会把集群的运维和配置状态保存在一个中央数据存储中。如果这个数据存储崩了，Calico网络还能继续用，但是无法更新了（新的Pod拿不到网络，无法更新策略，等等。）。

Calico有两种数据存储驱动供你选择

- **etcd** - 直接连接到一个etcd集群
- **Kubernetes** - 连接到Kubernetes API server

## 使用Kubernetes作为数据存储

本文使用了Kubernetes API数据存储驱动。这么干的好处是

- 不需要额外的数据存储，管理简单
- 可以用Kubernetes RBAC来控制对Calico资源的访问
- 可以用Kubernetes审计日志记录Calico的资源变更

出于讲解的完整性，使用etcd驱动的好处是

- 可以在非Kubernetes平台上运行Calico（比如OpenStack）
- 分离Kubernetes和Calico资源的问题域，比如可以单独对数据存储进行扩缩
- 可以在一个Calico集群中运行多个Kubernetes集群，比如采用Calico主机防护的裸金属服务器，跟一个Kuberenetes集群进行交互；或者是多个Kubernetes集群之间进行交互。

## 自定义资源

使用Kubernetes API数据存储驱动时，大部分Calico资源都保存为[Kubernetes自定义资源](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)。

零星几个Calico资源没有保存为自定义资源，就是用原本的Kubernetes资源类型来实现的。比如[16WorkloadEndpoint](../../../06%E5%8F%82%E8%80%83/04资源定义/16WorkloadEndpoint.md)就是Kubernetes的Pod。

为了用Kubernetes作为Calico的数据存储，我们要定义Calico用到的自定义资源。

下载并研究Calico自定义资源列表，在文本编辑器中打开看看。

```
wget https://projectcalico.docs.tigera.io/manifests/crds.yaml
```

在Kubernetes中创建自定义资源。

```
kubectl apply -f crds.yaml
```

## calicoctl

要想直接操作Calico数据存储，要用`calicoctl`客户端工具。

### 安装

1. 将`calicoctl`二进制文件下载到一个能访问Kubernetes的主机上。

```shell
wget https://github.com/projectcalico/calicoctl/releases/download/v3.20.0/calicoctl
chmod +x calicoctl
sudo mv calicoctl /usr/local/bin/
```

2. 配置`calicoctl`访问Kubernetes。

```shell
export KUBECONFIG=/path/to/your/kubeconfig
export DATASTORE_TYPE=kubernetes
```

在大部分系统中，kubeconfig都是保存在`~/.kube/config`。你可以把`export`命令保存到`~/.bashrc`，这样就省的每次登录之后都要配。

### 测试

验证`calicoctl`访问数据存储

```
calicoctl get nodes
```

输出内容类似这样

```text
NAME
ip-172-31-37-123
ip-172-31-40-217
ip-172-31-40-30
ip-172-31-42-47
ip-172-31-45-29
```

节点都是Kubernetes的节点对象，所以这里的名字应该跟`kubectl get nodes`的结果能对应上。

尝试获取一个基于自定义资源的对象

```
calicoctl get ippools
```

会看到一个空的结果

```text
NAME   CIDR   SELECTOR


```
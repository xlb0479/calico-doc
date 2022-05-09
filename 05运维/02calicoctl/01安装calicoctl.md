# 安装calicoctl

## 大面儿

本文帮你安装`calicoctl`命令行工具，管理Calico资源并执行管理功能。

## 价值

`calicoctl`命令行工具是使用Calico众多特性的必备工具。它可以用来管理Calico策略及配置，并查看集群的详细状态。

## 概念

### API组

所有的Kubernetes资源都有一个API组。API组可以从资源的`apiVersion`中看出来。例如Calico使用`projectcalico.org/v3`这个API组中的资源进行配置，operator则使用`operator.tigera.io/v1`组中的资源。

详见[Kubernetes文档](https://kubernetes.io/docs/reference/using-api/#api-groups)。

### calicoctl和kubectl

应该使用`calicoctl`来管理`projectcalico.org/v3`API组中的Calico API。因为`calicoctl`为这些资源提供了重要的校验以及默认值，这些在`kubectl`中都没有。但`kubectl`依然能够用于管理其它Kubernetes资源。

> 注意：如果你想用`kubectl`管理`projectcalico.org/v3`组中的资源，可以使用[Calico API服务](../06%E8%AE%A9kubectl%E7%AE%A1%E7%90%86Calico%E7%9A%84API.md)。

> 警告：永远不要直接去修改`crd.projectcalico.org`组中的资源。这些东西都属于内部数据格式，改了的话可能会导致异常的行为。

除了资源管理，`calicoctl`还可以进行其它Calico管理任务，例如查看IP池使用情况以及BGP状态。

### 数据存储

Calico对象可以使用两种数据存储，etcd或Kubernetes。数据存储的选择在安装Calico的时候就定下来了。通常部署在Kubernetes上时默认就用的Kubernetes数据存储。

只要主机能够连接到Calico数据存储，就可以使用`calicoctl`，直接用二进制或者通过容器使用都可以。操作步骤可以在下面对应的环境中看到。

## 怎么弄

> 注意：确保你的`calicoctl`版本总是跟Calico的版本保持一致。

- [在单个主机上安装calicoctl二进制](#在单个主机上安装calicoctl二进制)
- [在单个主机上将calicoctl安装成kubectl的插件](#在单个主机上将calicoctl安装成kubectl的插件)
- [在单个主机上将calicoctl安装成一个容器](#在单个主机上将calicoctl安装成一个容器)
- [将calicoctl安装成一个Kubernetes的Pod](#将calicoctl安装成一个Kubernetes的Pod)

### 在单个主机上安装calicoctl二进制

#### Linux

1. 登录到主机上，打开一个终端，进入你想安装二进制的目录中。

> 提示：位置最好是在`PATH`中。比如`/usr/local/bin`。

2. 下载`calicoctl`二进制。

```shell
$ curl -L https://github.com/projectcalico/calico/releases/download/v3.22.2/calicoctl-linux-amd64 -o calicoctl
```

3. 增加可执行权限。

```shell
$ chmod +x ./calicoctl
```

> 注意：如果`calicoctl`不在`PATH`中，要么移动文件，要么把位置加入到`PATH`中。这样你就可以直接执行它了。

### 在单个主机上将calicoctl安装成kubectl的插件

#### Linux

1. 登录到主机上，打开一个中断，进入到你想安装二进制的目录中。

> 提示：位置最好是在`PATH`中。比如`/usr/local/bin`。

2. 下载`calicoctl`二进制。

```shell
$ curl -L https://github.com/projectcalico/calico/releases/download/v3.22.2/calicoctl-linux-arm64 -o kubectl-calico
```

3. 增加可执行权限。

```shell
$ chmod +x kubectl-calico
```

> 注意：如果`calicoctl`不在`PATH`中，要么移动文件，要么把位置加入到`PATH`中。这样你就可以直接执行它了。

确认插件正常工作。

```shell
   kubectl calico -h
```

现在可以通过`kubectl calico`子命令来执行`calicoctl`了。

> 注意：如果在你自己的电脑上执行这些命令（而不是在某个主机节点上），有些与节点相关的子命令不能用（例如node status）。

### 在单个主机上将calicoctl安装成一个容器

在主机上将`calicoctl`装成一个容器，登录到目标主机并执行下面的命令即可。

```shell
$ docker pull calico/ctl:v3.22.2
```

### 将calicoctl安装成一个Kubernetes的Pod

根据你的数据存储类型选择对应的YAML，将`calicoctl`容器部署到你的节点中。

- etcd

```shell
$ kubectl apply -f https://projectcalico.docs.tigera.io/manifests/calicoctl-etcd.yaml
```

> 注意：可以[在新标签页中查看它的YAML](https://projectcalico.docs.tigera.io/manifests/calicoctl-etcd.yaml)。

- Kubernetes API数据存储

```shell
$ kubectl apply -f https://projectcalico.docs.tigera.io/manifests/calicoctl.yaml
```

> 注意：可以[在新标签也中查看它的YAML](https://projectcalico.docs.tigera.io/manifests/calicoctl.yaml)。

现在可以通过kubectl来执行命令了。

```shelll
$ kubectl exec -ti -n kube-system calicoctl -- /calicoctl get profiles -o wide
```

返回结果示例如下。

```shell
NAME                 TAGS
kns.default          kns.default
kns.kube-system      kns.kube-system
```

我们建议你设置一个别名。

```shell
$ alias calicoctl="kubectl exec -i -n kube-system calicoctl -- /calicoctl"
```

> 注意：为了让`calicoctl`能够读取manifest，需要将文件重定向到stdin，例如：
> ```shell
> calicoctl create -f - < my_manifest.yaml
> ```
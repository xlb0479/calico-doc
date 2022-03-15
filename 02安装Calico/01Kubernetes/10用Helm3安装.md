# 用Helm安装

## 主旨

用Helm3在一个Kubernetes集群上安装Calico。

## 价值

Helm chart是一种打包Kubernetes应用的方法（类似操作系统的`apt`或`yum`）。ArgoCD等工具也都用Helm来管理集群应用，重点在于安装、升级（如果需要还有回滚），等等。

## 开始之前

### 必备条件

- 安装Helm 3
- Kubernetes集群满足下列条件：
    - Kubernetes未安装CNI插件**或者**集群运行着一个兼容性的CNI，允许Calico以policy-only模式运行。
    - x86-64、arm64、ppc64le、或s390x处理器
    - RHEL 7.x+，CentOS 7.x+，Ubuntu 16.04+，或Debian 9.x+
- 已配置集群的`kubeconfig`（执行`kubectl get nodes`确认一下）
- Calico可以管理主机的`cali`和`tunl`接口。如果主机有NetworkManager，参见[配置NetworkManager](../../05%E8%BF%90%E7%BB%B4/10排错/01排错及诊断.md#配置NetworkManager)。

## 概念

### 基于Operator安装

这里我们使用Helm 3安装Tigera operator和CRD。Operator为Calico提供整个生命周期的管理，通过Kubernetes API暴露出来，定义成了一个自定义资源（CRD）。

## 怎么做

### 下载Helm chart

1. 添加Calico helm repo：

```
helm repo add projectcalico https://projectcalico.docs.tigera.io/charts
```

### 自定义Helm chart

如果你是在一个用EKS、GKE、AKS或Mirantis Kubernetes Engine（MKE）安装的集群上进行安装，或者你本来就需要自定义TLS证书，那么**必须**创建一个`values.yaml`文件来定制这个Helm chart。否则你可以跳过这步。

1. 如果你是在一个用EKS、GKE、AKS或Mirantis Kubernetes Engine（MKE）安装的集群上进行安装，根据[安装指引](../../06%E5%8F%82%E8%80%83/02安装#Provider)中的描述，要设置`kubernetesProvider`。例如：

```
echo '{ installation: {kubernetesProvider: EKS }}' > values.yaml
```

2. 将其它的需要定制的内容添加到`values.yaml`中。你可能需要看一下[helm文档](https://helm.sh/docs/)，或者是执行

```
helm show values projectcalico/tigera-operator --version v3.22.1
```

来检查一下在这个chart中可以进行自定义的值。

### 安装Calico

1. 使用Helm chart安装Tigera Calico operator和CRD：

```
helm install calico projectcalico/tigera-operator --version v3.22.1
```

或者如果你已经按照上面说的创建了`values.yaml`：

```
helm install calico projectcalico/tigera-operator --version v3.22.1 -f values.yaml
```

2. 确认所有Pod都已经跑起来了。

```
watch kubectl get pods -n calico-system
```

等所有Pod的`STATUS`都变成`Running`。

> 注意：Tigera operator将资源安装在`calico-system`命名空间中。其他的安装方式可能会装到`kube-system`中。

恭喜！你已经用Helm 3 chart完成了Calico的安装。

## 下一步

## 必读

- [安装和配置calicoctl](../../05%E8%BF%90%E7%BB%B4/02calicoctl/01安装calicoctl.md)

### 推荐学习

- [使用Kubernetes NetworkPolicy API加固一个简单的应用](../../04%E5%AE%89%E5%85%A8/03策略入门/02K8S策略/03K8S策略，基础教程.md)
- [使用Kubernetes NetworkPolicy API控制ingress和egress流量](../../04%E5%AE%89%E5%85%A8/03策略入门/02K8S策略/04K8S策略，高级教程.md)
- [跑个例子，展示实时的阻断和允许的连接](../../04%E5%AE%89%E5%85%A8/03策略入门/02K8S策略/02K8S策略，一个demo.md)
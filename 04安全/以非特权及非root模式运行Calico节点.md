# 以非特权及非root模式运行Calico节点

## 大面儿

在非特权及非root容器中运行Calico组件。

## 价值

在非特权及非root模式下运行Calico，这对于那些想要尽可能加固Calico的用户来说是个不错的选择，或者说用户根本不需要Calico那些除了基本的网络与网络策略之外的特性。这其中就要在安全性及网络管理复杂性之间做出权衡。比如你不需要Calico对集群中其它组件做出的错误配置进行调整，并且只需要支持少数几个特性。

## 概念

如果要尽可能的加固Calico，持续性的Calico组件（比如calico/node）可以在各自容器无需特权及root权限。注意在部署这些组件时，初始化容器仍然需要以特权和root模式运行，但它对集群的安全风险影响已经最小化了，因为初始化容器就来那么一下子。

## 支持

- 仅限Operator安装模式。

## 不支持

- Calico企业版
- eBPF数据面

> 注意：不保证支持Calico v3.21之后增加的特性。

## 怎么弄

1. 参照Tigera Calico operator的[安装手册](../02%E5%AE%89%E8%A3%85Calico/01Kubernetes/01%E5%BF%AB%E9%80%9F%E5%BC%80%E5%A7%8B.md)。如果已经装了operator了，那就直接看下一步。
2. 编辑Calico的installation，将`nonPrivileged`设置为`Enabled`。

```shell
kubectl edit installation default
```

改完之后的installation资源差不多是这个样子：

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Enabled
    hostPorts: Enabled
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
    linuxDataplane: Iptables
    multiInterfaceMode: None
    nodeAddressAutodetectionV4:
      firstFound: true
  cni:
    ipam:
      type: Calico
    type: Calico
  controlPlaneReplicas: 2
  flexVolumePath: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/
  nodeUpdateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  nonPrivileged: Enabled
  variant: Calico
```

3. 位于`calico-system`命名空间中的`calico-node`Pod现在应该会重启。检查重启情况。

```shell
watch kubectl get pods -n calico-system
```

此时，Calico的`calico-node`就是在非特权及非root容器中运行了。
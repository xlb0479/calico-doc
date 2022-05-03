# 防御Dos攻击

## 大面儿

Calico可以自动的在数据包处理流水线的最早的位置设置特定类型的黑名单策略，还可以向NIC硬件卸压力。

## 价值

遭受DoS攻击时，集群会收到大量来自攻击者的连接请求。这些连接请求越快被丢弃掉，主机就可以越少遭受到洪流或超载。当你在Calico网络策略中定义了DoS缓和规则时，Calico会采用尽可能影响最小的方式实施这些规则。

## 特性

本文用到了Calico的以下特性：

- **HostEndpoint**作为策略的实施点
- **GlobalNetworkSet**用于管理黑名单CIDR
- **GlobalNetworkPolicy**用于拒绝那些属于全局网络集IP的ingress流量

## 概念

### 最早数据包处理

在数据包处理流水线中，最早可以将数据包丢弃的点，依赖于Linux的内核版本以及NIC驱动和NIC硬件的能力。Calico会自动使用给最先可用的选项。

|**由谁处理**|**何时会被Calico利用**|**性能**
|-|-|-
|NIC硬件|NIC支持**XDP offload**模式。|最快
|NIC驱动|NIC驱动支持**XDP native**模式。|较快
|内核|内核支持**XDP generic模式**，并且将Calico配置成使用该功能的模式。这种模式很少用到，而且相对于下面的iptable的raw模式，没有什么性能收益。要使用的话，详见[Felix配置](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/05Felix%E9%85%8D%E7%BD%AE.md)|快
|内核|如果上面的模式都不可用，则使用**iptables raw**模式。|快

> 注意：XDP模式需要Linux内核版本v4.16及以上。

## 怎么弄

抵御DoS攻击拢共分为以下几步：

- [1：创建主机端点](#1：创建主机端点)
- [2：在全局网络集中添加CIDR到黑名单](#2：在全局网络集中添加CIDR到黑名单)
- [3：创建拒绝入口流量的全局网络策略](#3：创建拒绝入口流量的全局网络策略)

### 最佳实践

下面会一步一步按照上面列出来的做，并且我们假设不会有其它的先决配置条件。一种最佳实践是在攻击发生前主动落实这些操作（创建主机端点、网络策略、全局网络集）。如果发生了DoS攻击，你只需要将CIDR添加到全局网络集中的黑名单即可，实现快速响应。

### 1：创建主机端点

首先，对于你想实施DoS缓和策略的网络接口，创建对应的HostEndpoint。在下面的例子中，HostEndpoint加固了**jasper**节点上IP为**10.0.0.1**且名为**eth0**的接口。

```yaml
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: production-host
  labels:
    apply-dos-mitigation: "true"
spec:
  interfaceName: eth0
  node: jasper
  expectedIPs: ["10.0.0.1"]
```

### 2：在全局网络集中添加CIDR到黑名单

然以后，创建一个**GlobalNetworkPolicy**，将期望的CIDR加入黑名单。下面的例子中，全局网络集会将CIDR**1.2.3.4/32**和**5.6.0.0/16**拉黑：

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: dos-mitigation
  labels:
    dos-deny-list: 'true'
spec:
  nets:
  - "1.2.3.4/32"
  - "5.6.0.0/16"

```

### 3：创建拒绝入口流量的全局网络策略

最后，创建GlobalNetworkPolicy并添加GlobalNetworkSet标签（上一步的**dos-deny-list**）作为ingress流量拒绝的选择器。为了更快速的在数据包层面拒绝被转发的流量，添加**doNotTrack**和**applyOnForward**参数。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: dos-mitigation
spec:
  selector: apply-dos-mitigation == 'true'
  doNotTrack: true
  applyOnForward: true
  types:
  - Ingress
  ingress:
  - action: Deny
    source:
      selector: dos-deny-list == 'true'
```
# 为Kubernetes的节点端口设置策略

## 大面儿

限制节点端口的访问。

## 价值

使用节点端口将Service暴露出去，这是Kubernetes的标准功能。但如果你想限制对其的访问，那就要使用Calico全局网络策略。

## 特性

本文用到了以下Calico特性：

- **GlobalNetworkPolicy**及其preDNAT属性
- **HostEndpoint**

## 概念

### 网络策略及其preDNAT属性

在一个Kubernetes集群中，节点IP和端口上的一个请求会被kube-proxy DNAT到后端的某个Pod上。如果在Calico全局网络策略中既想允许正常的集群ingress流量又想拒绝所有其它的ingress流量，那么就必须要在DNAT之前搞事情。为了实现这一点，你只需要在Calico全局网络策略中添加一个**preDNAT**字段。该字段：

- 在DNAT前生效
- 只用于ingress规则
- 作用于主机端点上所有的ingress流量，不管它的目的地是哪里。目的地可以是一个主机本地的Pod、另一个节点上的Pod、或者是主机自身的一个进程。

## 开始之前……

为了安全地将Kubernetes服务暴露到外面，你必须实现以下所有操作。

- [放行集群ingress流量，但是拒绝一般的ingress流量](#放行集群ingress流量，但是拒绝一般的ingress流量)
- [允许本地的egress流量](#允许本地的egress流量)
- [创建主机端点及适当的网络策略](#创建主机端点及适当的网络策略)
- [放行特定节点端口的ingress流量](#放行特定节点端口的ingress流量)

### 放行集群ingress流量，但是拒绝一般的ingress流量

下面的例子中，我们创建了一个全局网络策略放行集群ingress流量（**allow-cluster-internal-ingress**）：为节点IP地址（**1.2.3.4/16**），以及Kubernetes分配的Pod IP地址（**100.100.100.0/16**）。我们加上了preDNAT属性，那么这个Calico全局网络策略就会在DNAT之前生效。

在这个例子中，我们使用了**selector: has(kubernetes-host)**——因此该策略会应用于所有带**kubernetes-host**标签的端点（你也可以轻松地指定出特殊节点）。

最后，当你添加了preDNAT属性后，那么必须要加上**applyOnForward: true**。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-cluster-internal-ingress-only
spec:
  order: 20
  preDNAT: true
  applyOnForward: true
  ingress:
    - action: Allow
      source:
        nets: [1.2.3.4/16, 100.100.100.0/16]
    - action: Deny
  selector: has(kubernetes-host)
```

### 允许本地的egress流量

### 创建主机端点及适当的网络策略

### 放行特定节点端口的ingress流量
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

我们还需要一个全局网络策略，用于放行流经每个节点外部接口的egress流量。否则，当我们给这些接口定义了主机端点之后，本地进程的egress流量就会被拒掉（除了[安全失败规则](../../06%E5%8F%82%E8%80%83/13%E4%B8%BB%E6%9C%BA%E7%AB%AF%E7%82%B9/05%E5%AE%89%E5%85%A8%E5%A4%B1%E8%B4%A5%E8%A7%84%E5%88%99.md)放行的流量）。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-outbound-external
spec:
  order: 10
  egress:
    - action: Allow
  selector: has(kubernetes-host)
```

### 创建主机端点及适当的网络策略

此时我们假设你已经定义了Calico主机端点，以及适当的网络策略。（比如你并不希望主机端点的网络策略是“默认拒绝所有出入主机的流量”，因为这样就不是你要的特定流量管控策略。）参见[主机端点](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/08%E4%B8%BB%E6%9C%BA%E7%AB%AF%E7%82%B9.md)。

上面定义的所有全局网络策略，它们的选择器都会选中任何带有**kubernetes-host标签**的端点；因此我们要在定义中加上它。例如**node1**的**eth0**。

```yaml
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: node1-eth0
  labels:
    kubernetes-host: ingress
spec:
  interfaceName: eth0
  node: node1
  expectedIPs:
  - INSERT_IP_HERE
```

创建每一个主机端点时，需要把`INSERT_IP_HERE`替换成eth0上的IP地址。`expectedIPs`是必填项，这样ingress或egress规则中的选择器才能正确的匹配到主机端点上。

### 放行特定节点端口的ingress流量

现在我们可以创建一个全局网络策略，设置preDNAT，允许外部访问节点端口。本例中，主机端点上**port: 31852**的**ingress流量都会被放行**。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-nodeport
spec:
  preDNAT: true
  applyOnForward: true
  order: 10
  ingress:
    - action: Allow
      protocol: TCP
      destination:
        selector: has(kubernetes-host)
        ports: [31852]
  selector: has(kubernetes-host)
```

如果只想让部分节点的**NodePort**可访问，那么给这些节点加一个特殊的标签。例如：

```yaml
nodeport-external-ingress: true
```

然后在**allow-nodeport**策略中去掉**has(kubernetes-host)**，使用**nodeport-external-ingress: true**作为选择器。
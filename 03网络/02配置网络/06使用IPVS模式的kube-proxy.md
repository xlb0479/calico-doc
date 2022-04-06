# 使用IPVS模式的kube-proxy

## 大面儿

使用IPVS模式的kube-proxy为Pod间的流量弄负载均衡。

## 价值

不管你是不是在搞容器网络，iptables都是个好帮手。但如果服务的规模扩展到1000个往上，那么就值得试一下IPVS模式的kube-proxy带来的性能提升。

## 价值

本文使用Calico以下特性：

- **FelixConfiguration**的KubeNodePortRange

## 概念

### Kubernetes kube-proxy

Kube-proxy处理每个节点上跟Service相关的所有的事儿。它要保证到服务集群IP和端口的连接能够指到对应的Pod上。如果有多个Pod，kube-proxy负载在这些Pod之间进行负载均衡。

Kube-proxy有三种模式：**userspace**、**iptables**、**ipvs**。（Userspace很老了，不建议用。）这里快速介绍下iptables和ipvs模式。

|**kube-proxy模式**|**设计之初……**|**Linux内核钩子**|**连接处理开销……**
|-|-|-|-
|iptables|一个高效的防火墙|NAT pre-routing使用顺序的规则|随着集群大小而增加
|ipvs|一个负载均衡器，带有调度选项，例如round-robin，最小期望延迟，最小连接，等等。|优化了查询过程|保持稳定，与集群大小无关

如果你想知道iptables和ipvs的性能差异，答案当然不会那么直接。关于iptables（包括Calico自己对iptables的使用）和ipvs模式的对比，见[比较kube-proxy模式：iptables还是IPVS？](https://www.tigera.io/blog/comparing-kube-proxy-modes-iptables-or-ipvs/)

### IPVS模式与NodePort范围

Kube-proxy的IPVS模式支持NodePort服务和集群IP。Calico也会使用NodePort将流量路由到集群，包括跟Kubernetes一样的默认NodePort范围（30000:32767）。如果你在Kubernetes中修改了默认的NodePort范围，那么必须在Calico这边也改一下，好让ipvs保持它的覆盖面。

### iptables：何时修改标记位

基于IPVS模式做流量监管时，Calico使用额外的iptables标记位，位每个本地的Calico端点（endpoint）保存一个ID。如果在IPVS模式下你打算在每个节点运行的Pod数量超过1022个，那你需要使用Calico[FelixConfiguration](../../06%E5%8F%82%E8%80%83/07Felix/01%E9%85%8D%E7%BD%AE.md#iptables数据面配置)中的`IptablesMarkMask`参数，调整一下标记位的大小。

### Calico自动检测ipvs模式

当Calico发现kube-proxy运行在IPVS模式时（在安装过程中或安装之后），那么就会自动启用对IPVS的支持。这种探测在calico-node启动的时候进行，因此当你修改了一个正在运行中的集群的kube-proxy模式，那么就需要重启calico-node实例。

## 开始之前

**必要条件**

- kube-proxy配置成IPVS模式
- ipvs模式下的Service类型为NodePort

## 怎么弄

按照我们之前讲的，使用IPVS模式无需在Calico中做任何调整；如果开启了，那么模式会被自动探测到。但如果你修改了Kubernetes默认的NodePort范围，需要用下面的指令更新Calico的nodeport范围，保持同步。探测行为在calico-node启动时进行，如果是在集群运行时修改了kube-proxy的模式，那就需要重启calico-node实例。

### 修改Calico默认的nodeport范围

在FelixConfiguration资源中，修改默认节点端口范围的配置参数（`KubeNodePortRange`），跟Kubernetes的保持一致。详见[FelixConfiguration](../../06%E5%8F%82%E8%80%83/07Felix/01%E9%85%8D%E7%BD%AE.md)。
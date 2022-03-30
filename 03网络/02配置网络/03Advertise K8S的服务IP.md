# Advertise K8S的服务IP

## 大面儿

让Calico将Kubernetes服务IP发布（advertise）到集群外。Calico支持对服务的集群（cluster）IP和外部（external）IP进行发布。

## 价值

一般来说，Kubernetes服务的集群IP只能在集群内访问，外部要想访问服务的话需要专用的负载均衡器或者ingress控制器。当服务的集群IP无法路由时，可以使用服务的外部IP来访问。

跟Calico对**Pod IP**的BGP发布类似，同样支持对Kuberentes的**Service IP**使用BGP发布到集群外。这样就避免了使用专用的负载均衡器。这种特性同样支持在集群内的节点上进行等价路由（ECMP）负载均衡，而且如果你需要更丰富的控制，还可以为本地服务保留源地址。

## 特性

本文使用了以下Calico特性：

**发布服务的集群和外部IP地址：**

- **BGPConfiguration**资源配置了`serviceClusterIPs`和`serviceExternalIPs`字段。

## 概念

### BGP大显身手

在Kubernetes中，对服务的所有请求都会重定向到一个适当的端点（Pod）上。因为Calico有BGP这个武器，将Kubernetes的服务IP发布到BGP网络中之后，外部流量就可以直接路由到Kuberentes的服务上。

如果你的部署模式是跟集群外部的BGP路由器建立了对等，这些路由器（以及该路由器传播到的任何上游位置）可以将流量发送到Kubernetes的服务IP，进而继续路由到服务下某个可用的端点上。

### 发布服务IP：快速入门

Calico实现了Kubernetes的**externalTrafficPolicy**，用kube-proxy将入口流量指向正确的Pod。根据服务配置的具体类型，发布机制也会有不同的处理方式。

|**服务模式**|**集群IP发布**|**流量是……**|**源地址是……**
|-|-|-|-
|Cluster（默认）|集群中所有节点静态发布一条到服务CIDR的路由|用ECMP在集群的节点间进行负载均衡，然后用SNAT转发到服务中适当的Pod上。可能会往另一个节点再跳一次，但整体上达到了负载均衡的效果。|被SNAT给挡住了
|Local|服务Pod所在的节点会发布一个到服务IP的特定路由（/32或/128）|为服务的端点在多个节点间做负载均衡。对于LoadBalancer和NodePort类型的服务避免了一跳路由，可能会有不均衡的负载分配。（其它流量是在集群内节点进行负载均衡的。）|保留

如果你的部署模式是跟集群外部的BGP路由器建立了对等，这些路由器——以及该路由器传播到的任何上游位置——可以将流量发送到Kubernetes的服务IP，进而继续路由到服务下某个可用的端点上。

### 取胜之匙

- 通常我们基于以下原因建议使用“Local”模式：
    - 如果你的任何网络策略是根据特定源IP地址来做匹配的，那么使用Local就是显而易见的了，因为该模式下源地址没有被修改，策略可以正常执行。
    - 返回的流量可以直接路由到源IP，因为“Local”服务无需
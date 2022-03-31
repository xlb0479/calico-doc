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
    - 返回的流量可以直接路由到源IP，因为“Local”服务无需解决源地址NAT的问题（不同于“Cluster”模式）。
- Cluster IP的发布最适合支持ECMP的ToR。否则特定路由的所有流量都会指向同一个节点。

## 开始之前

**要求**

- 在你的网络设备和Calico之间[配置BGP对等](01配置BGP对等.md)
- 为了实现服务的ECMP负载均衡，上游的路由器必须使用BGP多路径。
- 至少要有一个集群外的节点来当路由器、路由反射器或ToR，跟集群内的Calico节点建立对等。
- 服务要配置成正确的服务模式（“Cluster”或“Local”）。对于`externalTrafficPolicy: Local`，服务必须是`LoadBalancer`或`NodePort`。

**限制**

- 都是关于OpenShift的，这里就不写了，我也不懂这东西。

## 怎么弄

- [发布服务的集群IP](#发布服务的集群IP)
- [发布服务的外部IP](#发布服务的外部IP)
- [发布服务的负载均衡IP](#发布服务的负载均衡IP)
- [在发布中剔除某些节点](#在发布中剔除某些节点)

### 发布服务的集群IP

1. 确定服务集群IP的范围。（如果集群是[dual stack](../03%E8%87%AA%E5%AE%9A%E4%B9%89IP%E5%9C%B0%E5%9D%80%E7%AE%A1%E7%90%86/03配置dual%20stack或仅IPv6.md)的。）

这些范围可以从Kubernetes API服务的`--service-cluster-ip-range`参数中看到。详见[Kubernetes API服务参考手册](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)

2. 看一下有没有默认的BGPConfiguration。

```shell
$ calicoctl get bgpconfig default
```

3. 根据上面命令的执行结果，更新或创建BGPConfiguration。

**更新默认的BGPConfiguration** 用下面的命令给BGPConfiguration来个Patch，记得把“10.0.0.0/24”替换成你自己的服务集群IP的CIDR：

```shell
$ calicoctl patch BGPConfig default --patch \
   '{"spec": {"serviceClusterIPs": [{"cidr": "10.0.0.0/24"}]}}'
```

**创建默认的BGPConfiguration** 用下面的例子创建一个默认的BGPConfiguration。把你的CIDR填到`serviceClusterIPs`，覆盖到要发布的集群IP，例如：

```yaml
$ calicoctl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
  - cidr: 10.96.0.0/16
  - cidr: fd00:1234::/112
EOF
```

详见[BGP配置资源](../../06%E5%8F%82%E8%80%83/04资源定义/02BGP配置.md)。

> 注意：在之前的Calico版本中，如果仅配置IPv4，服务集群IP的发布是通过环境变量CALICO_ADVERTISE_CLUSTER_IPS来配置的。这个环境变量的优先级高于任何默认BGPConfiguration中的serviceClusterIPs。我们建议不要再用废弃的CALICO_ADVERTISE_CLUSTER_IPS，都用BGPConfiguration。

### 发布服务的外部IP

1. 确认所有你要发布到集群外的服务外部IP的范围。

2. 看一下有没有默认的BGPConfiguration。

```shell
$ calicoctl get bgpconfig default
```

3. 根据上面命令的执行结果，更新或创建BGPConfiguration。

**更新默认的BGPConfiguration** 用下面的命令给BGPConfiguration来个Patch，把服务外部IP的CIDR加进去：

```shell
$ calicoctl patch BGPConfig default --patch \
   '{"spec": {"serviceExternalIPs": [{"cidr": "x.x.x.x"}, {"cidr": "y.y.y.y"}]}}'
```

**创建默认的BGPConfiguration** 用下面的例子创建一个默认的BGPConfiguration。把你的外部IP的CIDR填到`serviceExternalIPs`属性中。

```yaml
$ calicoctl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceExternalIPs:
  - cidr: x.x.x.x/16
  - cidr: y.y.y.y/32
EOF
```

详见[BGP配置资源](../../06%E5%8F%82%E8%80%83/04资源定义/02BGP配置.md)。

### 发布服务的负载均衡IP

下面的操作可以让Calico发布Service的`status.LoadBalancer.Ingress.IP`地址。

1. 确认Service LoadBalcner分配的IP地址范围。

2. 看一下有没有默认的BGPConfiguration。

```shell
$ calicoctl get bgpconfig default
```

3. 根据上面命令的执行结果，更新或创建BGPConfiguration。

**更新默认的BGPConfiguration** 用下面的命令给BGPConfiguration来个Patch，把负载均衡IP的CIDR加进去：

```shell
$ calicoctl patch BGPConfig default --patch '{"spec": {"serviceLoadBalancerIPs": [{"cidr": "x.x.x.x/16"}]}}'
```

**创建默认的BGPConfiguration** 用下面的例子创建一个默认的BGPConfiguration。把负载均衡IP的CIDR填到`serviceLoadBalancerIPs`属性中。

```yaml
$ calicoctl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceLoadBalancerIPs:
  - cidr: x.x.x.x/16
EOF
```

详见[BGP配置资源](../../06%E5%8F%82%E8%80%83/04资源定义/02BGP配置.md)。

关于Service的LoadBalancer地址分配超出了当前Calico的范围，但可以用外部控制器来实现。可以自己搞一个，也可以用第三方的实现，比如MetalLB。

如果要安装MetalLB控制器来分配地址，按照下面的操作进行。

1. 参考[MetalLB](https://metallb.universe.tf/installation/#installation-by-manifest)安装`metallb-system/controller`资源。

但是不要安装`metallb-system/speaker`组件。speaker组件会在节点上建立BGP会话，跟Calico有冲突。

2. 创建下面的ConfigMap配置MetalLB的地址分配，把`x.x.x.x/16`换成上面给Calico配的CIDR。

```yaml
kubectl create -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: bgp
      addresses:
      - x.x.x.x/16
EOF
```

### 在发布中剔除某些节点

有的时候需要在发布的服务地址中剔除某些节点。比如没有任何服务的控制面节点。

要想剔除某个节点，打一个`node.kubernetes.io/exclude-from-external-load-balancers=true`标签即可。

例如剔除`control-plane-01`节点：

```shell
kubectl label node control-plane-01 node.kubernetes.io/exclude-from-external-load-balancers=true
```

## 教程

关于Calico的服务发布是如何工作的，这里有篇博文[Kubernetes Service IP路由发布](/%E5%8D%9A%E6%96%87/Kubernetes%20Service%20IP路由发布.md)。
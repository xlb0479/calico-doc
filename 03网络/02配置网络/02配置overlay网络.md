# Overlay网络

## 大面儿

当网络无法感知工作负载的IP时，为工作负载之间建立通信机制。

## 价值

通常我们建议运行无overlay/封装的Calico。这样也是性能最好且最简单的网络；从你的工作负载中出来的数据包跟线路上传输的数据包保持完全一样。

但是吧，如果底层网络无法轻易感知到工作负载的IP，选择性的使用overlay/封装也是有意义的。一个常见的情况是在多VPC/子网的AWS中使用Calico网络。此时，Calico会选择性的为跨VPC/子网的流量进行封装，在每个VPC/子网内的流量则无需封装。你也可以选择让整个Calico网络都进行封装，弄成一整个overlay网络——在无需建立BGP对等或为底层网络添加路由信息的情况下快速入门。

## 特性

本文使用了以下特性：

**IPPool**资源：

- ipipMode字段（IP in IP封装）
- vxlanMode字段（VXLAN封装）

## 概念

### 路由工作负载的IP地址

网络感知工作负载的IP地址可以通过3层路由技术，例如静态路由或BGP路由分发，也可以通过2层地址学习。如此这般，它们就可以按照最终目的地的指示将流量路由到正确的主机上。但并不是所有的网络都可以路由工作负载的IP地址。比如在公有云上，硬件不归你管，又或者是在AWS上面跨子网边界，等等还有其它无法和底层通过BGP建立Calico对等的场景，或者连静态路由也没法很容易的进行配置。这就是为什么Calico要支持封装，这样你才能在无需底层网络感知工作负载IP的情况下在工作负载之间完成流量传输。

### 封装类型

Calico支持两种类型的封装：VXLAN和IP in IP。有些IP in IP用不了的场景（比如Azure）可以使用VXLAN。VXLAN带有一个微小的数据包开销，因为它的header更大，但除非你的工作负载对网络特别敏感，否则这种开销通常你也是注意不到的。另一点不同在于Calico实现VXLAN时没有使用BGP，实现IP in IP的时候在Calico节点间使用了BGP。

### 跨子网

流量封装通常是因为流量要跨不同的路由器，它自己的路由器无法路由工作负载的IP。Calico可以封装以下流量：所有流量，无流量，仅跨子网边界的流量。

## 怎么弄

可以为每个IP池设定不同的封装配置。但是在一个IP池内无法混用封装类型。

- [仅为跨子网流量配置IP in IP封装](#仅为跨子网流量配置IP%20in%20IP封装)
- [为所有工作负载间流量配置IP in IP封装](#为所有工作负载间流量配置IP%20in%20IP封装)
- [仅为跨子网流量配置VXLAN封装](#仅为跨子网流量配置VXLAN封装)
- [为所有工作负载间流量配置VXLAN](#为所有工作负载间流量配置VXLAN)

### IPv4/6地址支持

IP in IP和VXLAN都只能支持IPv4地址。

### 最佳实践

Calico可以配置成仅对跨子网边界的流量进行封装。我们建议使用**cross-subnet**选项，不管是IP in IP还是VXLAN，最小化封装带来的损耗。跨子网模式可以在AWS多AZ部署、Azure VNET，以及其它使用路由器为多个节点池建立L2连通性的网络中提供更好的性能。

切换封装模式会导致当前的网络连接中断。早做准备。

### 仅为跨子网流量配置IP in IP封装

IP in IP封装可以选择性的进行，可以仅对跨子网边界的流量进行。

要开启这种特性，需要将`ipipMode`设置成`Crosssubnet`。

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ippool-ipip-cross-subnet-1
spec:
  cidr: 192.168.0.0/16
  ipipMode: CrossSubnet
  natOutgoing: true
```

### 为所有工作负载间流量配置IP in IP封装

如果把`ipipMode`设置成`Always`，所有启用Calico的主机，连接所有Calico网络IP池中的容器和VM的流量都会使用IP in IP。

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ippool-ipip-1
spec:
  cidr: 192.168.0.0/16
  ipipMode: Always
  natOutgoing: true
```

### 仅为跨子网流量配置VXLAN封装

VXLAN封装可以选择性的进行，可以仅对跨子网边界的流量进行。

要开启这种特性，需要将`vxlanMode`设置成`Crosssubnet`。

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ippool-vxlan-cross-subnet-1
spec:
  cidr: 192.168.0.0/16
  vxlanMode: CrossSubnet
  natOutgoing: true
```

### 为所有工作负载间流量配置VXLAN

如果把`vxlanMode`设置成`Always`，所有启用Calico的主机，连接所有Calico网络IP池中的容器和VM的流量都会使用VXLAN。

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ippool-vxlan-1
spec:
  cidr: 192.168.0.0/16
  vxlanMode: Always
  natOutgoing: true
```

如果你只用了VXLAN池，那就不需要BGP了，你可以[自定义manifest](../../02%E5%AE%89%E8%A3%85Calico/01Kubernetes/04自己管理的专有云/02自定义manifests.md)，关闭BGP，减少集群中可变化的组件。将`calico_backend`设置成`vxlan`，并且关闭BGP的readiness检查。
# Calico部署选项

Calico灵活的模块化架构支持多种部署选项，这样你就可以选择最适合你环境的网络和网络策略选项。这就包括它可以跟各种不同的CNI、IPAM插件、底层网络选项一起运行的能力。

在[快速开始](02安装Calico/01Kubernetes/01%E5%BF%AB%E9%80%9F%E5%BC%80%E5%A7%8B.md)中，默认使用了在每个环境中最常用的选项，这样你暂时就无需深入了解它们的细节。

你可以在下面各个选项中了解更多细节。

## 策略

### Calico策略

Kubernetes网络策略是用网络插件实现的，而不是Kubernetes自己搞的。如果只创建了一个网络策略资源，但是没有实现它的网络插件，那就没什么卵用。

Calico插件实现了Kubernetes网络策略的全部特性。此外，Calico还支持Calico网络策略，提供超越Kubernetes网络策略的额外特性和功能。Kubernetes和Calico网络策略可以无缝配合，这样你就可以选择适合你的选项，混起来用，达到你要的效果。

## IPAM

### Calico IPAM

Kubernetes如何为Pod分配IP地址，由使用的IPAM（IP地址管理）插件决定。

Calico的IPAM插件按需为节点分配小块的IP地址，在整体上提供IP地址空间的可用性。此外，Calico IPAM支持高级特性，例如IP池，还可以为命名空间或Pod指定IP地址范围，甚至指定Pod的IP地址。

## CNI

### Calico CNI

Kubernetes使用CNI（容器网络接口）插件来决定Pod连接底层网络的具体细节。Calico CNI插件将Pod通过L3路由连接到主机网络，无需L2桥接。这种方式既简单又好理解，而且比一般其它的方法更高效，比如kubenet或flannel。

## Overlay

### VXLAN Overlay

Overlay网路允许Pod间进行跨节点通信，而且底层网络无需感知Pod或Pod的IP地址。

不同节点上Pod之间的数据包用VXLAN进行封装，将每一个原始数据包裹在一个外部数据包中，然后使用节点的IP，将Pod的IP隐藏在内部数据包中。Linux内核可以非常高效地完成这个事儿，但仍然带来了一个较小的开销，如果是运行一些对网络敏感的工作负载，你可能想要避免这种开销。

出于知识的完整性，作为对比，如果不使用overlay，那就能达到最高的网络性能。从你的Pod中出来的数据包就是网络上传输的数据包。

### 无Overlay

不使用Overlay可以提供最佳的网络性能。从Pod出去的数据包就是在网络上传输的数据包。

出于讲解的完整性，作为对比，用了Overlay的网络，不同节点间Pod传输的数据包被VXLAN或IPIP协议封装了，将每个原始数据包用节点IP封装在一个外部数据包之内，将Pod IP隐藏起来了。Linux内核可以非常高效的完成这个事儿，但仍然带来了一笔消耗，如果对网络很敏感，那你可能也想避免这部分的消耗。

## 路由

### Calico路由

Calico为Pod在节点间的流量进行路由分发和路由编程，用的是它的数据存储，无需BGP。Calico路由在单个子网内支持无封装流量，在多子网集群中也可以选择性的使用VXLAN封装。

### BGP路由

BGP（边界网关协议）用于对不同节点间Pod的流量进行动态编程路由。

BGP是用来构建因特网的一个基于标准的路由协议。它的伸缩性极强，即便是最大规模的Kubernetes集群，对BGP来说也是小菜一碟。

Calico可以用三种模式来运行BGP：

- **Full mesh** - 每个节点都跟其它节点进行BGP交互，可轻松扩展至100个节点，

## 数据存储

### kubernetes

Calico将集群的操作和配置状态统一保存在一个中央的数据存储中。如果这个数据存储崩了，Calico网络依然能够保持正常，但是无法更新了（新的Pod没网，无法更新策略，等）。

Calico有两种数据存储驱动供你选择：

- etcd - 直接连接etcd集群
- Kubernetes - 连接kubernetes的API server。

用Kubernetes做数据存储的优势在于：

- 不需要额外的数据存储，安装和管理都简单
- 可以用Kubernetes的RBAC对Calico资源进行访问控制
- 可以使用Kubernetes生成Calico资源变更的审计日志

出于完整性，使用etcd作为数据存储的优势在于：

- 可以在非Kubernetes平台运行Calico（比如OpenStack）
- 分离Kubernetes和Calico的问题域，比如可以单独对数据存储进行扩缩
- Calico集群中不光可以包含一个Kubernetes集群，比如在裸金属服务器上用Calico做主机防护，可以跟一个或多个Kubernetes集群进行交互。

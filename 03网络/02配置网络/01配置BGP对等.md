# 配置BGP对等

## 大面儿

在Calico节点间或者和网络设施之间配置BGP（边界网关协议）对等，分发路由信息。

## 价值

Calico节点可以基于BGP交换路由信息，开启Calico网络中工作负载（Kubernetes的Pod或OpenStackde VM）的可达性。在专有云部署中这种方法可以让你的工作负载变成整个网络中的一等公民。在公有云部署中，它可以提供集群内高效的路由分发机制，通常跟IPIP overlay或跨子网模式搭配使用。

## 特性

本文使用了Calico的以下特性：

- Node资源
- BGPConfiguration资源
- BGPPeer资源
    - 全局对等
    - 指定节点对等

## 概念

### BGP

**BGP**是在网络中的路由器之间交换路由信息的一个标准协议。每个运行BGP的路由器都有一个或多个**BGP对等** - 使用BGP通信的其他路由器。你可以认为Calico网络在每个节点上提供了一个虚拟路由器。你可以配置Calico的每个节点之间都进行对等，也可以跟路由反射器对等，或者跟top-of-rack（ToR）路由器对等。

### 常见BGP拓扑

根据你的环境，有很多种配置BGP网络的方法。这里给出Calico的几种常用方式。

### Full-mesh

打开BGP之后，Calico模式会创建一个**full-mesh**的内部BGP（iBGP），每个节点之间都会彼此建立对等。这可以让Calico在任何L2网络上运行，不管是公有云还是私有云，或者是[配置](02配置overlay网络.md)了IPIP，在不阻止IPIP流量的网络中运行overlay。Calico不会为VXLAN overlay使用BGP。

> 注意：大部分公有云都支持IPIP。值得注意的是Azure，它会阻止IPIP流量。所以如果是Azure的话，你必须[配置Calico使用VXLAN](02%E9%85%8D%E7%BD%AEoverlay%E7%BD%91%E7%BB%9C.md)。

少于100个节点的情况下，full-mesh都可以工作良好，一旦规模变得更大就会变得低效，此时我们建议使用路由反射器。
# 配置outgoing NAT

## 大面儿

配置Calico网络，为那些从Pod向集群外发起的连接进行出口NAT。Calico可选地将Pod IP做源地址转换到节点IP。

## 价值

Calico的出口NAT选项非常灵活；可以开启，可以关闭，可以应用到包含公网IP的Calico IP池，也可以应用到内网IP的Calico IP池，或者是特定的IP地址范围。本文介绍一些开启和关闭出口NAT的场景。

## 特性

本文使用了Calico的以下特性：

- **IPPool**资源的`natOutgoing`字段

## 概念

### Calico IP池和NAT

当一个池中IP对应的Pod发起了一个向Calico IP池外的IP地址连接时，发送出去的数据包通过SNAT（源地址转换）将源Pod IP修改为节点IP。在这个连接中返回的数据包在到达Pod之前会自动进行反向修改。

### 开启NAT：Pod IP无法在集群外路由

开启出口NAT的一个常见的场景是，允许overlay网络中的Pod连接到overlay之外的IP地址，或者是带有内网IP的Pod连接到集群/网络之外的公网IP地址（当然也要满足网络策略的要求）。开启NAT后，该池中Pod发起的向其它任意非Calico IP池地址的连接都会进行NAT。

### 关闭NAT：使用物理设施的专有云部署

如果你实现Calico网络时使用了[和物理网络设施建立BGP对等](01配置BGP对等.md)的方法，那么你可以使用你自己的设施对Pod向因特网的流量进行NAT。此时你应该关闭Calico的`natOutgoing`选项。如果你想让你的Pod有公网IP，那就应当：

- 让Calico跟你的物理网络设施建立对等
- 给这些Pod创建一个带公网IP的IP池，路由到你的网络中，并且关闭NAT（`nat-outgoing: false`）
- 确认没有其他的网络设备对Pod流量做NAT

## 咋弄

- [创建IP池并开启outgoing NAT](#创建IP池并开启outgoing%20NAT)
- [使用额外的IP池来声明无需NAT的地址](#使用额外的IP池来声明无需NAT的地址)

### 创建IP池并开启outgoing NAT

下面我们创建了一个Calico IPPool，开启了natOutgoing。池中工作负载所在的节点会为它们进行出口NAT。

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 192.168.0.0/16
  natOutgoing: true
```

### 使用额外的IP池来声明无需NAT的地址

因为Calico只会对那些目标IP不属于Calico IPPool的连接做出口NAT，那么你就可以创建额外的IPPool，这些池子不用来管理Pod IP，而是用来防止特定CIDR被NAT。当你想对跑到互联网的流量做NAT，但是在某些内网范围内的流量不想NAT，那么这招就有用了。比如发往10.0.0.0/8的流量不想NAT，那么就可以创建下面这种池子。必须确保集群和10.0.0.0/8之间的网络可以路由Pod的IP。

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: no-nat-10.0.0.0-8
spec:
  cidr: 10.0.0.0/8
  disabled: true
```
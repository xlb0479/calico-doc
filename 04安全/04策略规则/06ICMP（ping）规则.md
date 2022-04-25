# 在策略中使用ICMP/ping规则

## 大面儿

使用Calico网络策略允许或拒绝ICMP/ping消息。

## 价值

**因特网控制报文协议（ICMP）**提供了有价值的网络探测功能，但也会被恶意使用。攻击者可以用它来了解你的网络，或者做DoS攻击。使用Calico网络策略，你可以控制哪里可以使用ICMP。比如，你可以：

- 允许ICMP ping，但只能是工作负载、主机断点（或二者皆有）
- 允许运维人员做诊断时的专用Pod可以使用ICMP，但其它Pod不能用
- 临时允许ICMP用于问题诊断，研究明白后再禁用
- 拒绝/允许ICMPv4和/或ICMPv6

## 特性

本文使用了Calico的以下特性：

**GlobalNetworkPolicy**或**NetworkPolicy**及其：

- 匹配ICMPv4和ICMPv6的协议
- 根据ICMP的type和code匹配icmp/非ICMP
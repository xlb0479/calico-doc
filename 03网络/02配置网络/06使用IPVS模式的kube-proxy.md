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

Kube-proxy处理每个节点上跟Service相关的所有的事儿。它要保证到服务集群IP和端口的连接能够指到对应的Pod上。
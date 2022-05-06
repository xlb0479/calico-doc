# 将Typha调度到常用节点

如果你用的是Kubernetes，使用的是Kubernetes API数据存储，并且集群节点大于50个，那你可以已经部署了Typha了。Typha是在较大集群中作为一个扇出代理，起到提升性能的作用。Typha代理必须要在固定端口上接受来自其它代理的请求。

作为Calico基础套件的一部分，Typha必须要在Pod网络形成之前进入可用状态，而且使用主机网络。它会在对应节点上打开一个端口。默认情况下它可以被调度到任意节点，并且开启TCP的5473端口。

为了减少需要开启该端口的节点数量，可以考虑[将Typha调度到常用节点](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)。
# 配置Kubernetes的控制面使用IPv6

## 大面儿

如果你的节点之间和工作负载之间打通了IPv6，那么Kubernetes控制面的通信你可能也想搞成IPv6的，不用IPv4了。

## 怎么弄

要让Kubernetes组件只使用IPv6，设置以下选项。

|**组件**|**选项**|**值/内容**
|-|-|-
|**kube-apiserver**|`--bind-address`或`--insecure-bind-address`|设置成适当的IPv6地址或设置成`::`监听主机全部的IPv6地址。
||`--advertise-address`|设置成节点用来访问`kube-apiserver`的IPv6地址。
|**kube-controller-manager**|`--master`|设置成`kube-apiserver`的IPv6地址。
|**kube-scheduler**|`--master`|设置成`kube-apiserver`的IPv6地址。
|**kubelet**|`--address`|设置成适当的IPv6地址或设置成`::`使用全部的IPv6地址。
||`--cluster-dns`|DNS服务的IPv6地址；必须是落在`--service-cluster-ip-range`范围内的地址。
||`--node-ip`|节点的IPv6地址。
|**kube-proxy**|`--bind-address`|设置成适当的IPv6地址或设置成`::`使用全部的IPv6地址。
||`--master`|设置成`kube-apiserver`的IPv6地址。

对于dual stack设置，见[开启IPv4/IPv6 dual-stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/#prerequisites)。
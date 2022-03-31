# 配置MTU提升网络性能

## 大面儿

为Calico环境配置最大传输单元（MTU）。

## 价值

配置MTU来优化网络性能，跟底层网络形成最佳适配。

增大MTU可以提升性能，降低MTU可以减少丢包和碎片问题。

## 特性

本文使用了Calico的以下特性：

- **FelixConfiguration**资源

## 概念

### MTU与Calico默认值

最大传输单元（MTU）决定了网络中可以传输的数据包大小的最大值。MTU配置在每个工作负载的veth以及隧道设备（如果开启了IP in IP、VXLAN或WireGuard）上。

通常来讲，保证不导致碎片以及丢包的同时把MTU调到最大，便可以获得最佳的性能。在给定的流量速率下，最大带宽增长与CPU消耗都会有所下降。这种优化在流量有封装（IP in IP、VXLAN或WireGuard）的时候更显得尤为重要，无法将这种流量的切分和组合工作转嫁到网卡上。

默认情况下，Calico会根据集群节点的配置以及实用的网络模式自动判断出正确的MTU。本文教你如何覆盖这种自动判定的MTU，使用自己设定的MTU。

要确认MTU的自动判定是否正常工作，确保在[felix配置](../../06%E5%8F%82%E8%80%83/04资源定义/05Felix配置.md)中设置了正确的封装模式。在felix配置中关闭无用的封装（`vxlanEnabled`、`ipipEnabled`、`wireguardEnabled`），确保自动探测可以为集群选出最优MTU。

## 开始之前……

关于如何使用IP in IP和/或VXLAN，见[配置overlay网络](02%E9%85%8D%E7%BD%AEoverlay%E7%BD%91%E7%BB%9C.md)。

关于如何使用WireGuard加密，见[配置WireGuard加密](../../04%E5%AE%89%E5%85%A8/09加密集群内Pod流量.md)。

## 怎么弄

- [决定MTU大小](#决定MTU大小)
- [配置MTU](#配置MTU)
- [查看当前隧道MTU值](#查看当前隧道MTU值)

### 决定MTU大小

下表列出了Calico常用的MTU大小。因为对于网络路径的两端来说MTU是一个全局属性，所以应当将MTU设置成数据包可能经过的所有路径中MTU的最小值。

#### 常用MTU

|**网络MTU**|**Calico MTU**|**Calico MTU，IP-in-IP（IPv4）**|**Calico MTU，VXLAN（IPv4）**|**Calico MTU，WireGuard（IPv4）**
|-|-|-|-|-
|1500|1500|1480|1450|1440
|9000|9000|8980|8950|8940
|1500 (AKS)|1500|1480|1450|1340
|1460 (GCE)|1460|1440|1410|1400
|9001 (AWS Jumbo)|9001|8981|8951|8941
|1450 (OpenStack VXLAN)|1450|1430|1400|1390

#### overlay网络MTU建议

在IP in IP、VXLAN、WireGuard协议中存在额外的overlay header，根据header的大小，缩小了MTU的最小值。（IP in IP使用了20字节的header，VXLAN是50字节，WireGuard是[60字节](https://lists.zx2c4.com/pipermail/wireguard/2017-December/002201.html)）。

使用AKS的话，底层网络[MTU为1400](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-tcpip-performance-tuning#azure-and-vm-mtu)，尽管网卡上的MTU可能是1500。WireGuard会设置数据包中的不允许分片（DF）位，所以在AKS中为了避免底层网络丢弃数据包，WireGuard的MTU应当设置成比1400再小60字节。


### 配置MTU

### 查看当前隧道MTU值
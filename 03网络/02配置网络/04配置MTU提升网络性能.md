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

如果你的集群中混用了WireGuard和IP in IP或VXLAN，那么应该将MTU设置成所有类型的最小值。这其中的缘由是，只有两端都开启了WireGuard时才会使用WireGuard的封装，只有两端都没有启用WireGuard的时候才会使用IP in IP或VXLAN封装。这种情况可能发生在你正在给节点安装WireGuard的时候。（不过真是，听着就乱。）

因此我们给出以下建议：

- 如果Pod网络中的任意位置使用了WireGuard，那就将MTU设置成“物理网络MTU减60”。
- 如果没用WireGuard，但是Pod网络中有地方用了VXLAN，那就将MTU设置成“物理网络MTU减50”。
- 如果没用WireGuard，但只使用了IP in IP，MTU设置成“物理网络MTU减20”
- 将工作负载端MTU和隧道MTU设置成一样的（这样所有路径上的MTU就都一样了）

#### eBPF模式

NodePort实现时使用VXLAN隧道在节点间发送数据包，因此要拿VXLAN的MTU来设置工作负载（veth）的MTU，并且应当是“物理网络MTU减50”（同上）。

#### flannel网络的MTU

如果使用了flannel网络，网卡MTU应当和flannel接口的MTU匹配。

- 如果使用了flannel的VXLAN，此时的通用设置参照上面表格中的“Calico MTU，VXLAN（IPv4）”列。

### 配置MTU

> 注意：为Calico更新了MTU只会影响到新的工作负载。

选择适当的方式来配置MTU。根据安装方法的不同则有所差异：

- 基于Manifest的安装（如果你没看快速入门，那么大部分非OpenShift安装都属于这种类型）
- Operator

#### Manifest

如果是基于manifest的安装（没用operator），需要编辑`calico-config`ConfigMap。例如：

```shell
$ kubectl patch configmap/calico-config -n kube-system --type merge \
  -p '{"data":{"veth_mtu": "1440"}}'
```

更新之后，给所有calico/node来一次滚动重启。例如：

```shell
$ kubectl rollout restart daemonset calico-node -n kube-system
```

#### Operator

如果是Operator安装，编辑Calico operator的`Installation`资源，设置`spec`的`calicoNetwork`中的`mtu`字段。例如：

```shell
$ kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"mtu":1440}}}'
```

类似的，OpenShift：

```shell
$ oc patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"mtu":1440}}}'
```

### 查看当前隧道MTU值

使用下面的命令查看当前隧道大小：

`ip link show`

IP in IP隧道作为tunlx出现（例如tunl0），后面有MTU的大小。例如：

![img](https://projectcalico.docs.tigera.io/images/tunnel.png)
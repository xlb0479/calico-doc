# 开启eBPF数据面

## 大面儿

告诉你怎么开启eBPF数据面；这是一个相对于标准数据面（基于iptables）来说性能更高的数据面。

## 价值

eBPF数据面相较于标准Linux网络管道模型有以下优势：

- 吞吐量更高。
- 每GBit占用CPU更少。
- 原生支持Kubernetes的Service（不需要kube-proxy）：
    - 减少Service的首包延迟。
    - 保留源客户端IP。
    - 支持DSR（直接服务器返回，Direct Server Return）高效路由。
    - 保持数据面同步时比kube-proxy消耗的CPU更少。

如果需要了解详细的测试结果，见博文[Calico的eBPF数据面](https://www.tigera.io/blog/introducing-the-calico-ebpf-dataplane/)。

## 限制

当前eBPF模式相对于标准Linux管道模式存在以下限制：

- 只支持x86-64。（目前eBPF还没有为其它平台构建。）
- 尚未支持IPv6。
- 启用后，已有连接依然使用非eBPF数据路径；这些连接不应该被中断，但是它们也享受不到eBPF的优势。
- 不支持混合模式（有的节点用eBPF有的节点则用标准数据面）。（在此类集群中，从eBPF节点到非eBPF节点的NodePort流量会被拒掉。）也包括带Windows的节点。
- 不支持浮动IP（floating IP）。
- 不支持SCTP，无论是对策略还是对服务。
- 需要节点开启[IP探测](../../03%E7%BD%91%E7%BB%9C/03%E8%87%AA%E5%AE%9A%E4%B9%89IP%E5%9C%B0%E5%9D%80%E7%AE%A1%E7%90%86/02%E9%85%8D%E7%BD%AEIP%E8%87%AA%E5%8A%A8%E6%8E%A2%E6%B5%8B.md)，即便不使用Calico CNI和BGP。在eBPF模式中，节点IP用于将流量从外部源转发到服务时生成VXLAN数据包。
- 不支持策略中的“Log”动作。

## 特性

用到了Calico的以下特性：

- **calico/node**
- **eBPF数据面**

## 概念

### eBPF

eBPF（又称“extended Berkeley Packet Filter”），是一种将小型程序安全地加载到Linux内核的各种底层钩子上的技术。eBPF用途广泛，包括网络、安全、跟踪。你会发现有很多与网络无关的项目也用到了eBPF，但对于Calico来说我们就是关注网络方面，将Linux内核的网络能力推向极致。

## 开始之前……

eBPF模式需要具备以下先决条件：

- 受支持的Linux发行版：
    - Ubuntu 20.04（或Ubuntu 18.04.4+，它的内核更新了）。
    - Red Hat v8.2且内核版本大于等于v4.18.0-193（红帽把相关特性做了向后移植）。
    - 其它[受支持的发行版](../../02%E5%AE%89%E8%A3%85Calico/01Kubernetes/14%E7%B3%BB%E7%BB%9F%E8%A6%81%E6%B1%82.md)，且内核版本大于等于v5.3

如果Calico没有检测到一个可兼容的内核，会给出一个警告然后退回到标准的Linux网络模式。

- 在每个节点上，BPF文件系统必须被挂载到`/sys/fs/bpf`。这一点是必须的，这样Calico重启时BPF的文件系统可以持久化。如果文件系统无法持久化，那么Calico重启的话Pod会暂时性的断开连接，而且主机端点可能会失去安全防护（因为它们相关的策略程序被丢弃了）。
- 要实现最佳的Pod间网络性能，底层网络应该去overlay。例如：
    - 位于单个AWS子网内的集群。
    - 使用了可兼容的云服务商CNI（比如AWS VPC CNI插件）。
    - 配置了BGP对等的专有云集群。

如果必须要用overlay，我们建议使用VXLAN，而不是IPIP。在eBPF模式下，由于大量的内核优化，VXLAN要比IPIP性能高很多。

- 底层网络在Calico主机间需要配置成支持VXLAN数据包的传递（即便你用IPIP或者无overlay）。在eBPF模式中，VXLAN用于转发Kubernetes NodePort流量，并且保留源IP。eBPF模式遵守Felix的`VXLANMTU`设置（见[MTU配置](../../03%E7%BD%91%E7%BB%9C/02%E9%85%8D%E7%BD%AE%E7%BD%91%E7%BB%9C/04%E9%85%8D%E7%BD%AEMTU%E6%8F%90%E5%8D%87%E7%BD%91%E7%BB%9C%E6%80%A7%E8%83%BD.md)）。
- 提供稳定的到Kubernetes API服务的连接方式。因为eBPF模式替代了kube-proxy，Calico需要直接访问API服务。
- 满足基本[要求](../../02%E5%AE%89%E8%A3%85Calico/01Kubernetes/14%E7%B3%BB%E7%BB%9F%E8%A6%81%E6%B1%82.md)。

> 注意：EKS使用的默认内核不兼容eBPF模式。如果你想用，参考[基于eBPF模式的EKS集群](https://projectcalico.docs.tigera.io/maintenance/ebpf/ebpf-and-eks)，会告诉你咋弄。

## 怎么弄

- [验证集群是否支持eBPF模式](#验证集群是否支持eBPF模式)
- [让Calico直接连接API服务](#让Calico直接连接API服务)
- [配置kube-proxy](#配置kube-proxy)
- [配置数据接口](#配置数据接口)
- [开启eBPF](#开启eBPF)
- [尝试DSR模式](#尝试DSR模式)
- [回滚](#回滚)

### 验证集群是否支持eBPF模式

教你如何确认集群是否支持eBPF。

1. 检查内核版本

```shell
$ uname -rv
```

输出结果示例：

```text
5.4.0-42-generic #46-Ubuntu SMP Fri Jul 10 00:24:02 UTC 2020
```

这里的内核版本是5.4，可以用。

在红帽系的发行版中可看到的应该是类似这样的结果：

```text
4.18.0-193.el8.x86_64 (mockbuild@x86-vm-08.build.eng.bos.redhat.com)
```

这里是红帽v4.18的内核，最后构建版本号为193，这个内核也是可以用的。


2. 确认BPF文件系统挂载，执行：

```shell
mount | grep "/sys/fs/bpf"
```

如果BPF文件系统已挂载，会看到：

```text
none on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
```

如果没有任何输出，那么BPF文件系统就是没有挂载；参考你操作系统发行版的说明书，看看如何在启动时将它挂载到标准位置/sys/fs/bpf。这里面可能会涉及到`/etc/fstab`或者是需要添加`systemd`单元，看你的操作系统的具体情况了。如果文件系统没有挂载，那么eBPF要等到Calico重启后才会生效，工作负载的网络也会中断几秒。

如果你的系统用了`systemd`，可以参考以下配置：

```shell
cat <<EOF | sudo tee /etc/systemd/system/sys-fs-bpf.mount
[Unit]
Description=BPF mounts
DefaultDependencies=no
Before=local-fs.target umount.target
After=swap.target

[Mount]
What=bpffs
Where=/sys/fs/bpf
Type=bpf
Options=rw,nosuid,nodev,noexec,relatime,mode=700

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start sys-fs-bpf.mount
systemctl enable sys-fs-bpf.mount
```

### 让Calico直接连接API服务

### 配置kube-proxy

### 配置数据接口

### 开启eBPF

### 尝试DSR模式

### 回滚
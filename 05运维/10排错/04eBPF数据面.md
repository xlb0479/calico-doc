# eBPF模式

## Service无法访问

如果集群内的Pod或主机无法访问Service，按照下面的操作进行检查：

- Service对应的主机必须开启eBPF模式或`kube-proxy`。如果打开eBPF的同时关闭了`kube-proxy`，确认一下eBPF是不是真的生效了。如果Calico发现内核不支持，它会回退到标准的数据面模式（也就无法支持Service了）。

要确认eBPF是否正常，检查`calico-node`容器日志；如果eBPF模式不支持，会有一条`ERROR`日志

```text
BPF dataplane mode enabled but not supported by the kernel.  Disabling BPF mode.
```

如果BPF模式正常，会看到一条`INFO`日志

```text
BPF enabled, starting BPF endpoint manager and map manager.
```

- 在eBPF模式下，外部客户端是通过VXLAN访问Service（通常是NodePort）的。如果后端的Pod在另一个节点并且此时出现NodePort超时，检查一下底层网络是否允许节点间VXLAN流量。VXLAN是一个UDP协议；默认使用4789端口。
- 在DSR模式下，Calico需要底层网络允许一个节点扮演另一个节点来做出响应。
    - 在AWS中，要允许这一点，需要关闭节点NIC上的Source/Dest检查。但是DSR只能用于AWS内部；它无法跟经过负载均衡器的外部流量兼容。这是因为负载均衡器期望流量是从同一个主机返回。
    - 在GCP中，必须启用“允许转发”选项。和AWS一样，经过负载均衡器的流量无法使用DSR，因为从后端节点返回时不会考虑到负载均衡器。

## 检查程序是否丢包

要检查eBPF程序是否丢包，可以使用`tc`命令。例如你担心`eth0`上的一个eBPF程序出现了丢包，你可以执行下面的命令：

```shell
tc -s qdisc show dev eth0
```

返回结果应该是类似下面这样；找到`clsact`qdisc，也就是eBPF程序的挂载点。`tc`的`-s`选项可以显示当前丢包数，也就相当于eBPF出现的丢包数。

```text
...
qdisc clsact 0: dev eth0 root refcnt 2 
 sent 1340 bytes 10 pkt (dropped 10, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
...
```

## 高CPU

如果你发现`calico-node`的CPU占用过高：

- 看看`kube-proxy`是否还在运行。如果是，那么你要么关闭`kube-proxy`，要么确保Felix配置中的`bpfKubeProxyIptablesCleanupEnabled`是`false`。如果是`true`（默认），Felix会尝试删除`kube-proxy`的iptables规则。如果此时`kube-proxy`还在运行，它跟`Felix`就会干起来。
- 如果你的集群特别大，或者你的工作负载正处于狂暴模式，你可以调大Felix更新服务数据面的间隔，也就是调大`bpfKubeProxyMinSyncPeriod`。默认是1秒。这个值的增大也就会导致服务更新的会更慢。
- Calico支持端点切片，类似`kube-proxy`。如果Kubernetes集群支持端点切片并且启用了它，那么你必须在Calico的`bpfKubeProxyEndpointSlicesEnabled`配置中也启用它。

## eBPF程序的调试日志

Calico的eBPF程序可以打开详细的调试日志。尽管这些日志非常的啰嗦（因为会记录每一个数据包），但是对于诊断eBPF程序问题来说非常有用。要开启这个日志，需要将Felix配置中的`bpfLogLevel`设置为`Debug`。

> 警告！这个日志会严重影响eBPF程序的性能。

这个日志会发送到内核跟踪buffer中，可以通过下面的命令来查看：

```shell
tc exec bpf debug
```

日志格式如下：

```text
     <...>-84582 [000] .Ns1  6851.690474: 0: ens192---E: Final result=ALLOW (-1). Program execution time: 7366ns
```

解释如下：

- `<...>-84582`告诉你哪个程序（或内核进程）在处理这个数据包。如果是发送出去的数据包，这里通常是发送数据的程序PID。如果是接收的数据包，一般是一个内核进程，或者是一个无关的进程恰巧触发了这个流程。
- `6851.690474`日志时间戳。
- `ens192---E`是Calico日志标签。对于挂载到接口上的程序，第一部分包含了接口名的一部分字母。后缀则是`-I`或`-E`，代表“Ingress”或“Egress”。“Ingress”和“Egress”的含义跟策略中的一样：
    - 工作负载Ingress程序在主机网络空间通往工作负载的路上执行。
    - 工作负载Egress程序在工作负载通往主机的路上执行。
    - 主机端点Ingress程序在外部节点通往主机的路上执行。
    - 主机端点Egress程序在主机通往外部主机的路上执行。
- `Final result=ALLOW (-1). Program execution time: 7366ns`消息内容。这里打印的是程序执行的最终结果。注意其中日志记录消耗了大量的时间。

## `calico-bpf`工具

由于BPF映射包含了二进制数据，Calico团队搞了一个可以查看Calico的BPF映射的工具。这个工具被打包到了calico/node容器镜像中。可以这样操作：

- 查看主机上的calico/node Pod名

```shell
$ kubectl get pod -o wide -n calico-system
```

比如得到的是`calico-node-abcdef`

- 继续执行：

```shell
$ kubectl exec -n calico-system calico-node-abcdef -- calico-node -bpf ...
```

可以查看该工具的帮助信息：

```shell
$ kubectl exec -n calico-system calico-node-abcdef -- calico-node -bpf help

Usage:
  calico-bpf [command]
  
Available Commands:
  arp          Manipulates arp
  connect-time Manipulates connect-time load balancing programs
  conntrack    Manipulates connection tracking
  help         Help about any command
  ipsets       Manipulates ipsets
  nat          Nanipulates network address translation (nat)
  routes       Manipulates routes
  version      Prints the version and exits
  
Flags:
  --config string   config file (default is $HOME/.calico-bpf.yaml)
  -h, --help            help for calico-bpf
  -t, --toggle          Help message for toggle
```

（因为这个工具是嵌入到`calico-node`二进制中的，所以不能用`--help`，但是可以用`calico-node -bpf help`。）

可以把BPF的conntrack表dump出来：

```shell
$ kubectl exec -n calico-system calico-node-abcdef -- calico-node -bpf conntrack dump
...
```

## 性能问题

有多种原因可能导致eBPF数据面性能下降。

- 确认你已经给集群配置了最佳的网络模式。如果可能的话，避免使用overlay网络；无overlay的路由网络应该会更快。如果你必须要用到Calico的overlay模式，那么就用VXLAN，不要用IPIP。由于内核限制，IPIP在eBPF模式下表现极差。
- 如果你没有用overlay，确保[Felix配置参数](../../06%E5%8F%82%E8%80%83/07Felix/01%E9%85%8D%E7%BD%AE.md)中的`ipInIpEnabled`和`vxlanEnabled`都是`false`。这些参数是用来控制Felix是否允许IPIP或VXLAN的，即便你没有使用overlay的IP池。这些参数还可以关闭eBPF模式中针对IPIP和VXLAN做的兼容性优化。

查看配置：

```shell
$ calicoctl get felixconfiguration -o yaml
```

```yaml
apiVersion: projectcalico.org/v3
items:
- apiVersion: projectcalico.org/v3
  kind: FelixConfiguration
  metadata:
    creationTimestamp: "2020-10-05T13:41:20Z"
    name: default
    resourceVersion: "767873"
    uid: 8df8d751-7449-4b19-a4f9-e33a3d6ccbc0
  spec:
    ...
    ipipEnabled: false
    ...
    vxlanEnabled: false
kind: FelixConfigurationList
metadata:
  resourceVersion: "803999"
```

- 如果你的集群是运行在AWS上，云服务商可能会限制你的节点间通信带宽。比如大部分的AWS节点都会将每个连接限制到5GBit。
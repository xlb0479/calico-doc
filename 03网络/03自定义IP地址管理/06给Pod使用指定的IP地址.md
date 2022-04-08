# 给Pod使用指定的IP地址

## 大面儿

给Pod选一个IP地址，而不是让Calico自动选择。

## 价值

有的应用需要使用固定的IP地址。而且你可能想在外部的DNS服务上配置直接指向Pod的记录，这也需要静态IP。

## 特性

本文使用以下特性：

- Calico IPAM
- IPPool资源

## 概念

### Kubernetes Pod CIDR

这个东西就是Kubernetes期望给Pod分配的IP范围。它是定义在整个集群上的，Kubernetes的大量组件都会使用这个东西来判定某个IP是否属于一个Pod。例如kube-proxy就会根据流量是否来自一个Pod的IP做出不同的处理。要想一切正常，所有Pod的IP必须都落在这个CIDR内。

#### IP池子

IP池是Calico给Pod分配IP的范围。静态IP必须落在某个池子内。

## 开始之前

必须使用Calico的IPAM。

如果不确定，可以ssh登录到Kubernetes节点上检查CNI配置。

```shell
cat /etc/cni/net.d/10-calico.conflist
```

找到以下内容：

```json
         "ipam": {
              "type": "calico-ipam"
          },
```

如果找到了，那就是用的Calico的IPAM。如果IPAM设置成了别的值，或者是没有10-calico.conflist这个文件，那么你的集群中就用不了这些特性了。

## 怎么弄

给Pod打一个cni.projectcalico.org/ipAddrs注解，值是一个IP地址列表，用方括号括起来。例如：

```yaml
  "cni.projectcalico.org/ipAddrs": "[\"192.168.0.1\"]"
```

注意内部双引号要使用`\"`去义。

这个地址必须落在一个Calico的IP池中，而且当前没有被占用。这个注解必须在Pod创建的时候就打好；后面再加就不管用了。

注意目前每个Pod用这个注解只支持配置一个单独的IP。

### 为手动分配保留IP

`cni.projectcalico.org/ipAddrs`注解需要使用IP池中的IP地址。这就意味着默认情况下，Calico可能会使用你准备给另一个工作负载或内部隧道用的IP地址。为了避免这个问题，有以下方法：

- 将某个IPPool全部保留给手动分配，可以将它的[节点选择器](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/09IP%E6%B1%A0.md)设置为`!all()`。因为`!all()`是无法匹配任何节点的，这个IPPool就无法用于任何自动分配的场景。
- 要保留池子里的一部分，可以创建一个[`IPReservation`资源](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/10IP%E4%BF%9D%E7%95%99.md)。它可以让某些IP保留下来，这样Calico IPAM就不会自动使用它们了。但是手动分配的时候（使用注解）就可以使用这些“被保留”的IP了。
- 为了避免Calico将池中IP用于内部IPIP和/或VXLAN隧道地址，可以将[IPPool](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/09IP%E6%B1%A0.md)的`allowedUses`字段设置为`["Workload"]`。
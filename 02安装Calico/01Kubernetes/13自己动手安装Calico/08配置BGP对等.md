# 配置BGP对等（Peer）

我们已经配置Calico基于边界网关协议（BGP）进行了路由信息分发。这种可伸缩的协议强化了全球公共互联网的路由能力。

在许多专有云数据中心，每个服务器在IP层（3层）连接到了一个机架顶部路由器（ToR）。我们要在每个节点和对应的ToR路由器之间建立对等，这样ToR就会学习到容器的路由。这些配置超出了我们的讲解范围。

因为我们是跑在一个AWS VPC单个子网中，主机间具有以太网（2层）连接，也就是它们之间没有路由器。这样它们可以直接彼此建立对等。

在安装了`calicoctl`的节点上看一下状态。

```shell
sudo calicoctl node status
```

结果

```
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 172.31.40.217 | node-to-node mesh | up    | 17:38:47 | Established |
| 172.31.40.30  | node-to-node mesh | up    | 17:40:09 | Established |
| 172.31.45.29  | node-to-node mesh | up    | 17:40:20 | Established |
| 172.31.37.123 | node-to-node mesh | up    | 17:40:29 | Established |
+---------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

或者你可以创建一个[`CalicoNodeStatus`资源](../../../06%E5%8F%82%E8%80%83/04资源定义/04Calico节点状态.md)来获取节点的BGP会话状态。

注意一共有四个BGP会话，每个到集群中的其他节点。在一个小型集群中，这种形式工作良好且弹性也不错。但是随着节点规模的增加，BGP会话数量也会增加，在大型集群中会产生一笔不小的损耗。

本堂课中我们配置固定数量的*路由反射器（route reflector）*。路由反射器会宣告它自己的路由以及它从其它对等接收到的路由。这样就是说节点只需要和路由反射器建立对等就可以获取集群中的所有路由。这种对等排布就导致BGP会话的数量随着节点数量一同线性增长。

## 选择节点打标签

我们要建立三个节点反射器，这样我们就可以避免单点故障以及维护时的麻烦。在一个五节点集群中就意味着只有一个BGP会话是无用的，因为两个非反射器节点不需要彼此建立对等，在大型集群中这样可以减少相当大的损耗。

选择三个节点并执行下面的操作。

保存节点的YAML。

```shell
calicoctl get node <node name> -o yaml --export > node.yaml
```

编辑YAML并添加

```yaml
metadata:
  labels:
    calico-route-reflector: ""
spec:
  bgp:
    routeReflectorClusterID: 224.0.0.1
```

重新应用YAML

```shell
calicoctl apply -f node.yaml
```

## 配置对等

配置所有非反射器节点跟所有路由反射器的对等

```yaml
calicoctl apply -f - <<EOF
kind: BGPPeer
apiVersion: projectcalico.org/v3
metadata:
  name: peer-to-rrs
spec:
  nodeSelector: "!has(calico-route-reflector)"
  peerSelector: has(calico-route-reflector)
EOF
```

配置所有路由反射器之间的彼此对等

```yaml
calicoctl apply -f - <<EOF
kind: BGPPeer
apiVersion: projectcalico.org/v3
metadata:
  name: rrs-to-rrs
spec:
  nodeSelector: has(calico-route-reflector)
  peerSelector: has(calico-route-reflector)
EOF
```

关闭node-to-node mesh

```yaml
calicoctl create -f - <<EOF
 apiVersion: projectcalico.org/v3
 kind: BGPConfiguration
 metadata:
   name: default
 spec:
   nodeToNodeMeshEnabled: false
   asNumber: 64512
EOF
```

找一个非反射器节点，现在应该只能看到三个对等。

```shell
sudo calicoctl node status
```

结果

```
Calico process is running.

IPv4 BGP status
+---------------+---------------+-------+----------+-------------+
| PEER ADDRESS  |   PEER TYPE   | STATE |  SINCE   |    INFO     |
+---------------+---------------+-------+----------+-------------+
| 172.31.37.123 | node specific | up    | 21:52:57 | Established |
| 172.31.40.217 | node specific | up    | 21:52:57 | Established |
| 172.31.42.47  | node specific | up    | 21:52:57 | Established |
+---------------+---------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

或者你可以创建一个[`CalicoNodeStatus`资源](../../../06%E5%8F%82%E8%80%83/04资源定义/04Calico节点状态.md)来获取节点的BGP会话状态。
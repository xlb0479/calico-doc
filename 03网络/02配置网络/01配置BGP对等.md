# 配置BGP对等

## 大面儿

在Calico节点间或者和网络设施之间配置BGP（边界网关协议）对等，分发路由信息。

## 价值

Calico节点可以基于BGP交换路由信息，开启Calico网络中工作负载（Kubernetes的Pod或OpenStackde VM）的可达性。在专有云部署中这种方法可以让你的工作负载变成整个网络中的一等公民。在公有云部署中，它可以提供集群内高效的路由分发机制，通常跟IPIP overlay或跨子网模式搭配使用。

## 特性

本文使用了Calico的以下特性：

- Node资源
- BGPConfiguration资源
- BGPPeer资源
    - 全局对等
    - 指定节点对等

## 概念

### BGP

**BGP**是在网络中的路由器之间交换路由信息的一个标准协议。每个运行BGP的路由器都有一个或多个**BGP对等** - 使用BGP通信的其他路由器。你可以认为Calico网络在每个节点上提供了一个虚拟路由器。你可以配置Calico的每个节点之间都进行对等，也可以跟路由反射器对等，或者跟top-of-rack（ToR）路由器对等。

### 常见BGP拓扑

根据你的环境，有很多种配置BGP网络的方法。这里给出Calico的几种常用方式。

### Full-mesh

打开BGP之后，Calico模式会创建一个**full-mesh**的内部BGP（iBGP），每个节点之间都会彼此建立对等。这可以让Calico在任何L2网络上运行，不管是公有云还是私有云，或者是[配置](02配置overlay网络.md)了IPIP，在不阻止IPIP流量的网络中运行overlay。Calico不会为VXLAN overlay使用BGP。

> 注意：大部分公有云都支持IPIP。值得注意的是Azure，它会阻止IPIP流量。所以如果是Azure的话，你必须[配置Calico使用VXLAN](02%E9%85%8D%E7%BD%AEoverlay%E7%BD%91%E7%BB%9C.md)。

少于100个节点的情况下，full-mesh都可以正常工作，一旦规模变得更大就会变得低效，此时我们建议使用路由反射器。

### 路由反射器

要想构建超大规模内部BGP（iBGP），**BGP路由反射器**可以用于减少每个节点的BGP对等数量。这种模式下，部分节点作为路由反射器，彼此之间建立full mesh。其它节点跟这些路由反射器（一般连俩做冗余）的一部分建立对等，相对于full-mesh减少了BGP对等连接的整体数量。

### Top of Rack（ToR）

在**专有云部署**中，可以让Calico跟物理网络设施直接建立对等。通常这需要关闭Calico默认的full-mesh，然后让Calico跟你的L3 ToR路由器建立对等。专有云BGP网络构建的方法也有很多。怎么配置BGP那是你的事儿——Calico在iBGP和eBGP配置下都可以良好工作，你可以把Calico跟网络中的路由器看成是一回事儿。

根据你的网络拓扑，可能需要在每个机架中都使用BGP路由反射器。但一般来说只有每个L2域内节点数很大（>100）的情况下才这么干。

关于专有云常用的部署模型，详见[基于IP Fabrics](../../06%E5%8F%82%E8%80%83/14架构/03网络设计/01光纤以太网.md)。

## 开始之前

先得把[calicoctl](../../05%E8%BF%90%E7%BB%B4/02calicoctl/01安装calicoctl.md)弄好。

## 怎么弄

> 注意：如果要对Calico的BGP拓扑进行大改，例如从full-mesh改成ToR对等，在配置变更期间可能会导致Pod网络连通性的短暂不可用。建议只在运维窗口期间内进行这些操作。

- [配置全局BGP对等](#配置管局BGP对等)
- [配置每节点BGP对等](#配置每节点BGP对等)
- [将一个节点配置成路由反射器](#将一个节点配置成路由反射器)
- [关闭默认BGP节点到节点的mesh](#关闭默认BGP节点到节点的mesh)
- [不中断网络的情况下从节点到节点的mesh改成路由反射器模式](#不中断网络的情况下从节点到节点的mesh改成路由反射器模式)
- [查看节点的BGP对等状态](#查看节点的BGP对等状态)
- [修改默认的全局AS号](#修改默认的全局AS号)
- [修改指定节点的AS号](#修改指定节点的AS号)

### 配置全局BGP对等

全局BGP对等应用于集群中的所有节点。如果你的网络中的BGP发言者（Speaker）跟每个Calico节点都建立了对等，这种模式就是非常有用的。

下面的例子创建了一个全局BGP对等，让每个Calico节点在AS**64567**中跟**192.20.30.40**建立对等。

```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: my-global-peer
spec:
  peerIP: 192.20.30.40
  asNumber: 64567
```

### 配置每节点BGP对等

每节点BGP对等应用于集群中的一个或多个节点。可以通过指定节点的名字进行选择，或者使用标签选择器。

下面的例子创建了一个BGPPeer，将所有带**rack: rack-1**标签的Calico节点在AS**64567**中跟**192.20.30.40**建立对等。

```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: rack1-tor
spec:
  peerIP: 192.20.30.40
  asNumber: 64567
  nodeSelector: rack == 'rack-1'
```

### 将一个节点配置成路由反射器

可以把Calico节点配置成路由反射器。每个作为路由反射器的节点必须要有一个集群ID——一般是一个未使用的IPv4地址。

如果将一个节点配置成集群ID为244.0.0.1的路由反射器，执行下面的命令。

```shell
calicoctl patch node my-node -p '{"spec": {"bgp": {"routeReflectorClusterID": "244.0.0.1"}}}'
```

通常你可以给这个节点打个标签，表示它的路由反射器身份，在BGPPeer资源中选择的时候更加方便。例如使用kubectl：

```shell
kubectl label node my-node route-reflector=true
```

现在可以非常简单的将路由反射器和其它节点，以及非路由反射器节点建立对等，使用标签选择器就好。例如：

```yaml
kind: BGPPeer
apiVersion: projectcalico.org/v3
metadata:
  name: peer-with-route-reflectors
spec:
  nodeSelector: all()
  peerSelector: route-reflector == 'true'
```

> 注意：给节点定义中添加`routeReflectorClusterID`会立即将该节点从节点到节点的mesh中移除，中断已有的BGP会话。添加BGP对等时会创建新的BGP会话。在进行这些操作期间会导致这些节点上的工作负载到数据面的流量出现短暂的中断（2秒左右）。如果要避免这种问题，需要确保这些节点上没有正在运行的工作负载，可以添加新的节点或在这些节点上运行`kubectl drain`（由于清理操作可能也会导致工作负载的运行中断）。

### 关闭默认BGP节点到节点的mesh

可以关闭默认的**节点到节点BGP mesh**以便打造不同的BGP拓扑。需要修改默认的**BGP configuration**资源。执行下面的命令关闭BGP full-mesh：

```shell
calicoctl patch bgpconfiguration default -p '{"spec": {"nodeToNodeMeshEnabled": false}}'
```

> 注意：如果默认的BGP配置不存在那就得先创建一个。详见[BGP配置](../../06%E5%8F%82%E8%80%83/04资源定义/02BGP配置.md)。

> 注意：关闭节点到节点的mesh会中断Pod网络，直到/或者用BGPPeer资源定义了可替代的BGP对等。为了避免Pod网络中断，可以在关闭节点到节点mesh之前先配置好BGPPeer资源。

### 不中断网络的情况下从节点到节点的mesh改成路由反射器模式

从节点到节点mesh切换到BGP路由反射器模式会打断BGP会话并创建新的会话。对于集群中节点上的工作负载来说，这样做会导致短暂的数据面网络波动（2秒左右）。为了避免这个问题，在关闭节点到节点的mesh会话之前，可以先建好路由反射器节点并建立起BGP会话。

需要按照下面说的步骤来：

1. [创建新的节点作为路由反射器](#将一个节点配置成路由反射器)。这些节点[应当是不可调度的](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)，并且在节点定义中应当包含`routeReflectorClusterID`。它们不属于当前节点到节点的BGP mesh，当mesh关闭后会作为路由反射器。这些节点同时还应当有一个类似`route-reflector`这样的标签，便于在BGP对等时进行选择。[另一种方法是](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)，可以选择集群中已存在的节点，执行`kubectl drain <NODE>`抽干上面的工作负载，然后把这些节点作为路由反射器，但这样做会导致这些被抽干的节点上的工作负载运行中断。
2. 创建一个[BGPPeer](#将一个节点配置成路由反射器)定义，用标签选择器，为路由反射器和其它非路由反射器节点建立对等。
3. 等待对等建立。可以在节点上执行`sudo calicoctl node status`[检查](#查看节点的BGP对等状态)对等状态。或者你可以创建一个[`CalicoNodeStatus`资源](../../06%E5%8F%82%E8%80%83/04资源定义/04Calico节点状态.md)来获取节点的BGP会话状态。
4. [关闭节点到节点的BGP mesh](#关闭默认BGP节点到节点的mesh)。
5. 如果你是将已有节点抽干，或者是创建新的节点并指定它们不可调度，此时需要将节点恢复可调度状态（执行`kubectl uncordon <NODE>`）。

### 查看节点的BGP对等状态

可以使用`calicoctl`查看指定节点的BGP连接状态。可以帮助你确认你的配置是否工作正常。

登录到你想检查的节点并执行下面的命令：

```shell
sudo calicoctl node status
```

输出了所有的邻居列表以及它们的当前状态。成功建立对等的会标记成**Established**。

> 注意：该命令是跟本地的Calico agent通信，所以你必须登录到你想检查的节点上去执行这个命令。

或者，你可以创建一个[`CalicoNodeStatus`资源](../../06%E5%8F%82%E8%80%83/04资源定义/04Calico节点状态.md)来获取节点的BGP会话状态。

### 修改默认的全局AS号

默认情况下所有的Calico节点都使用了64512自治系统，除非给节点指定了AS。可以为所有节点修改全局的默认值，修改默认的**BGPConfiguration**资源即可。下面的命令将全局默认的AS号改为**64513**。

```shell
calicoctl patch bgpconfiguration default -p '{"spec": {"asNumber": "64513"}}'
```

> 注意：如果这玩意儿不存在那就要先创建一个。详见[BGP配置](../../06%E5%8F%82%E8%80%83/04资源定义/02BGP配置.md)。

### 修改指定节点的AS号

可以修改特定节点的AS，使用`calicoctl`修改节点对象即可。比如下面的命令将名为**node-1**的节点修改成了**AS 64514**。

```shell
calicoctl patch node node-1 -p '{"spec": {"bgp": {"asNumber": "64514"}}}'
```
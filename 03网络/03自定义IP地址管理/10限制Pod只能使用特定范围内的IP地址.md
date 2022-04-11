# 限制Pod只能使用特定范围内的IP地址

## 大面儿

把Pod IP的选择范围限制到一个特定的IP地址范围内。

## 价值

Kubernetes的Pod跟外部系统交互时如果需要根据IP范围来做决策（比如遗留的防火墙），那么可以定义多个IP范围，将Pod直接赋到这些范围中。使用Calico IP地址管理（IPAM），可以将Pod使用的IP地址限制到某个特定范围内。

## 特性

这里使用以下特性：

- **Calico IPAM**
- **IPPool资源**

## 概念

### Kubernetes Pod CIDR

**Kubernetes Pod CIDR**就是Kubernetes期望Pod分配IP地址的范围。它定义在整个集群范围，很多组件都是用它来判断某个IP是否属于一个Pod。例如kube-proxy根据流量是否来源于一个Pod会做出不同的处理。所有Pod的IP都必须要落在这个CIDR中。

### IP池子

**IP池子**是Calico用来为Pod分配IP地址的范围。默认情况下Calico为整个Kubernetes Pod CIDR创建一个IP池子，但是你可以把这个CIDR拆成多个不同的池子。你可以使用节点选择器、Pod注解或命名空间，来控制Calico给Pod分配地址时使用的池子。

## 开始之前……

我们需要以下特性：

- Calico IPAM

如果不确定有没有，登录到Kubernetes节点上检查CNI配置。

```shell
cat /etc/cni/net.d/10-calico.conflist
```

找到以下：

```json
         "ipam": {
              "type": "calico-ipam"
          },
```

如果有，那你就是用的Calico IPAM。如果IPAM不是Calico，或者10-calico.conflist这个文件都不存在，那你就用不了这个特性了。而且，集群管理员必须已经[配置了IP池子](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/09IP%E6%B1%A0.md)，定义了有效的IP范围来分配Pod IP。

## 怎么弄

### 限制一个Pod使用的IP地址范围

给Pod加注解`cni.projectcalico.org/ipv4pools`和/或`cni.projectcalico.org/ipv6pools`，值为IP池名字列表，用方括号括起来。例如：

`cni.projectcalico.org/ipv4pools: '["pool-1", "pool-2"]'`

注意在池子名两边用"括起来了。

### 限制某个命名空间下的所有Pod使用特定的IP地址范围

给命名空间加注解`cni.projectcalico.org/ipv4pools`和/或`cni.projectcalico.org/ipv6pools`，值为IP池名字列表，用方括号括起来。例如：

`cni.projectcalico.org/ipv4pools: '["pool-1", "pool-2"]'`

注意在池子名两边用"括起来了。

如果Pod和Pod的命名空间上都有注解，Pod上的注解的优先级更高。

必须在Pod创建时就加好注解。给已有Pod加这个注解没用。
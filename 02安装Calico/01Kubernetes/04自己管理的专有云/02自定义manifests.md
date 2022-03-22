# 自定义manifests

## 关于自定义manifests

我们提供了很多manifests，简化Calico的部署。在应用之前可以选择性的进行修改。或者你也可以在之后把想改的改一下重新应用也行。

各种修改对应的详细介绍。

- [定制Calico manifests](#定制Calico%20manifests)
- [定制应用层策略manifests](#定制应用层策略manifests)

## 定制Calico manifests

### 关于定制Calico manifests

每个manifest中都包含了所有安装Calico需要的资源。

它在Kubernetes中安装以下资源：

- 用DaemonSet在每个节点安装`calico/node`容器。
- 用DaemonSet在每个节点安装Calico CNI二进制和网络配置。
- 用Deployment安装`calico/kube-controllers`。
- `calico-etcd-secrets`Secret，可以提供etcd的TLS制品。
- `calico-config`ConfigMap，包含了用于配置安装过程的参数。

下面我们进一步讲一下这里面可配置的参数。

### 配置Pod的IP范围

Calico的IPAM从[IP池](../../../06%E5%8F%82%E8%80%83/04资源定义/09IP池.md)中分配IP地址。

要修改Pod的默认IP地址范围，就要改`calico.yaml`中的`CALICO_IPV4POOL_CIDR`。详见[配置calico/node](../../../06%E5%8F%82%E8%80%83/06calico-node.md)。

### 配置IP-in-IP

默认情况下，manifests中开启了跨子网时的IP-in-IP封装。许多用户可能想把这个IP-in-IP封装关了，例如下列场景。

- 集群[运行在一个配置好的AWS VPC中]()
- 所有Kubernetes节点都连到了同一个2层网络中。
- 打算用BGP对等，让底层设施了解Pod的IP地址。

要关闭IP-in-IP封装，改一下manifest中的`CALICO_IPV4POOL_IPIP`。详见[配置calico/node](../../../06%E5%8F%82%E8%80%83/06calico-node.md)。

### 把IP-in-IP改成VXLAN

默认Calico开启了IP-in-IP封装。如果你的网络中阻止了IP-in-IP，比如Azure，那么你可能就会想改成[Calico的VXLAN封装模式](../../../03%E7%BD%91%E7%BB%9C/02配置网络/02配置overlay网络.md)。要想在安装时做这种改动（这样Calico就会用VXLAN创建默认的IP池，不需要撤销已有的IP-in-IP配置）：

- 从[Calico策略和网络](#自定义manifests)的一个manifests开始搞。
- 把环境变量名`CALICO_IPV4POOL_IPIP`改为`CALICO_IPV4POOL_VXLAN`。改完之后它的值得是“Always”。
- 可选的，（如果是在一个只有VXLAN的集群中，为了节省一些资源）完全关闭Calico的基于BGP的网络：
    - 将`calico_backend: "bird"`改为`calico_backend: "vxlan"`。关闭BIRD。
    - 在calico/node的readiness/liveness检查中注释掉`- -bird-ready`和`- -bird-live`（否则关闭了BIRD会导致每个节点的readiness/liveness检查都会失败）：
    ```yaml
          livenessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-live
             # - -bird-live
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              # - -bird-ready
              - -felix-ready
    ```

关于calico/node的其它配置，包括其他的VXLAN配置，见[配置calico/node](../../../06%E5%8F%82%E8%80%83/06calico-node.md)。

> 注意：环境变量`CALICO_IPV4POOL_VXLAN`只有在第一个calico/node开始创建默认的IP池时才会起作用。如果池子已经建完了那这个环境变量也就不再起作用了。如果要在安装之后改成VXLAN模式，需要用calicoctl修改[IPPool](../../../06%E5%8F%82%E8%80%83/04资源定义/09IP池.md)资源。

### 配置etcd

默认情况下这些manifests不会配置到etcd的安全访问，并且假设每个节点上都有一个etcd代理。下面的选项允许你自定义etcd集群的endpoint，以及TLS。

下表中给出了用于支持etcd的`ConfigMap`选项：

|**选项**|**描述**|**默认值**
|-|-|-
|etcd_endpoints|逗号间隔的etcd endpoint列表|http://127.0.0.1:2379
|etcd_ca|这个文件中包含了签发etcd服务证书的CA的根证书。配置`calico/node`，CNI插件，以及Kubernetes控制器，让它们信任etcd服务提供的证书中的签名。|无
|etcd_key|这个文件中包含了`calico/node`的私钥，CNI插件和Kubernetes控制器的客户端证书。让这些组件加入到双向TLS认证，并且对etcd服务表明自己的身份。|无
|etcd_cert|这个文件中包含了签发给`calico/node`，CNI插件，以及Kubernetes控制器的客户端证书。让这些组件加入到双向TLS认证，并且对etcd服务表明自己的身份。|无

使用这些manifests的时候，如果etcd集群那边启用了TLS，那么必须按照下面的操作进行：

1. 根据你的安装方式下载对应的v3.22 manifest。

**Calico策略和网络**

```shell
curl https://projectcalico.docs.tigera.io/manifests/calico-etcd.yaml -O
```

**Calico策略和flannel网络**

```shell
curl https://projectcalico.docs.tigera.io/manifests/canal.yaml -O
```

## 定制应用层策略manifests
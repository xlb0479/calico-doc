# 给Pod添加一个浮动IP

## 大面儿

给Pod配置一个或多个浮动IP，作为Pod额外可用的IP地址。

## 价值

类似Kubernetes的Service，有的网络服务后端的Pod可能会发生变化，一个浮动的IP可以提供一个访问该服务的稳定的IP地址。对于Kubernetes的服务来说主要的优势就在于浮动IP可以用于所有的协议：不光是TCP、UDP和SCTP。不同于Kubernetes服务的地方在于，一个浮动IP一次只能代表一个Pod，无法用于负载均衡。

## 特性

本文使用Calico以下特性：

**Calico CNI配置文件**中启用floating_ips

## 概念

一个**浮动IP**就是给工作负载增加一个额外的IP地址。“浮动”的意思就是说可以在集群内各种调整，可以在不同时刻代表不同的Pod。工作负载本身通常是对浮动IP无感知的；主机对入口流量使用网络地址转换（NAT），将浮动IP改成工作负载的真实IP，然后再将数据包发送给工作负载。

一个Kubernetes Service可以拿到一个**集群IP**（当然也可能还会有一个nodePort和/或一个外部负载均衡器IP），网络中的其它端点可以用它来访问对应的Pod集合，这里面也用到了网络地址转换。在很多场景中，Kubernetes Service也可以用来解决浮动IP的用例，并且通常也都建议Kubernetes用户这么去用，因为它是Kubernetes中的一个原生概念。用Kubernetes Service搞不定的是无法使用UDP、TCP和SCTP之外的协议（这些协议极少用到）。

## 开始之前

需要用到：

- Calico CNI插件

如果想验证一下有没有，可以通过ssh登录到一个Kubernetes节点上，然后找一下CNI插件配置文件，通常是在`/etc/cni/net.d/`中。如果看到了`10-calico.conflist`这个文件，那就说明你正在使用Calico CNI插件。

## 怎么弄

- [开启浮动IP](#开启浮动IP)
- [给Pod配置一个浮动IP](#给Pod配置一个浮动IP)

### 开启浮动IP

默认情况下浮动IP是关闭的。要想启用就按照下面说的来。

修改kube-system命名空间中的calico-config ConfigMap。在`cni_network_config`中，给“calico”插件添加下面的片段。

```json
    "feature_control": {
         "floating_ips": true
     }
```

举个例子来说，改完之后的`cni_network_config`应该类似下面这样。

```json
cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.0",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "datastore_type": "kubernetes",
          "nodename": "__KUBERNETES_NODE_NAME__",
          "mtu": __CNI_MTU__,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "__KUBECONFIG_FILEPATH__"
          },
          "feature_control": {
              "floating_ips": true
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        }
      ]
    }
```

### 给Pod配置一个浮动IP

给Pod加一个注解`cni.projectcalico.org/floatingIPs`，值是用方括号括起来的IP地址列表。为了保证在集群内做出正确的广播，所有的浮动IP必须落在配置的[IP池](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/09IP%E6%B1%A0.md)范围内。

例如：

```yaml
"cni.projectcalico.org/floatingIPs": "[\"10.0.0.1\"]"
```

注意内部双引号要使用`\"`去义。
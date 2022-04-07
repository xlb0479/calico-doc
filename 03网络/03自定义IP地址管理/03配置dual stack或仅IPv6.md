# 配置dual stack或仅IPv6

## 大面儿

配置Calico IP地址分配，使用dual stack或仅IPv6。

## 价值

基于IPv6的工作负载通信需求日益增长，也有替代IPv4的势头。Calico支持：

- 仅IPv4（默认）<br/>每个工作负载得到一个IPv4地址，可以基于IPv4通信。
- Dual stack<br/>每个工作负载得到一个IPv4地址和一个IPv6地址，可以基于IPv4或IPv6地址通信。
- 仅IPv6<br/>每个工作负载得到一个IPv6地址，可以基于IPv6通信。

## 特性

本文使用Calico的以下特性：

- **CNI插件配置**中的`assign_ipv6`和`assign_ipv4`标记
- **IPPool**

## 开始之前

### Calico要求

- Calico IPAM

### Kubernetes版本要求

- dual stack需要大于等于1.16
- 单栈（IPv4或IPv6），任意版本即可

### Kubernetes IPv6主机要求

- 从其它节点可达的一个IPv6地址
- sysctl设置，`net.ipv6.conf.all.forwarding`设置成`1`。这样可以确保Kubernetes服务的流量和Calico的流量都能正确的转发。
- 一个默认的IPv6路由

### Kubernetes IPv4主机要求

- 从其它节点可达的一个IPv4地址
- sysctl设置，`net.ipv4.conf.all.forwarding`设置成`1`。这样可以确保Kubernetes服务的流量和Calico的流量都能正确的转发。
- 一个默认的IPv4路由

## 怎么弄

> 注意：下面的工作只用于新的集群。

- [启用仅IPv6](#启用仅IPv6)
- [启用dual stack](#启用dual%20stack)

### 启用仅IPv6

1. 搞一个新的Kubernetes集群，Pod的CIDR和Service的IP范围都要用IPv6的。
2. 根据[Calico Kubernetes安装指引](../../02%E5%AE%89%E8%A3%85Calico/01Kubernetes/04%E8%87%AA%E5%B7%B1%E7%AE%A1%E7%90%86%E7%9A%84%E4%B8%93%E6%9C%89%E4%BA%91/01%E5%9C%A8%E4%B8%93%E6%9C%89%E4%BA%91%E5%AE%89%E8%A3%85Calico.md)，为集群和数据存储的类型下载对应的Calico manifest。
3. 编辑CNI配置（manifest中的calico-config ConfigMap），关闭IPv4，启用IPv6。

```json
    "ipam": {
        "type": "calico-ipam",
        "assign_ipv4": "false",
        "assign_ipv6": "true"
    },
```

4. 为`calico-node`容器添加下面的环境变量，配置IPv6支持：

|**变量名**|**值**
|-|-
|`IP6`|`autodetect`
|`FELIX_IPV6SUPPORT`|`true`

5. 如果**不是**用kubeadm安装的集群（见下面的说明），为`calico-node`容器添加下面的环境变量，配置默认的IPv6的IP池：

|**变量名**|**值**
|-|-
|`CALICO_IPV6POOL_CIDR`|跟kube-controller-manager和kube-proxy中配置的值保持一致

> 注意：如果是使用kubeadm创建的集群，Calico可以自动探测IPv4和IPv6的CIDR，不需要配置。

6. 应用manifest，执行`kubectl apply -f`。

新的Pod就可以拿到IPv6地址了，它们之间的通信，以及跟外部的通信，就可以使用IPv6了。

#### （可选）更新主机并关闭IPv4地址查询

如果你想让工作负载只使用IPv6地址，因为你没有IPv4地址并且节点间也没有对应的连通性，那么可以完成下面的操作，让Calico不再查询IPv4的地址。

1. 关闭[IPv4的IP自动探测](02%E9%85%8D%E7%BD%AEIP%E8%87%AA%E5%8A%A8%E6%8E%A2%E6%B5%8B.md)，将`IP`设置成`none`。
2. 使用下面的方法之一，为Calico计算出IPv6的BGP路由器ID。
    - 为calico/node设置环境变量`CALICO_ROUTER_ID=hash`。这样可以让Calico根据主机名来计算出路由器ID。
    - 为每个节点分别指定一个唯一的`CALICO_ROUTER_ID`。

### 启用dual stack

1. 根据Kubernetes的[要求](https://kubernetes.io/docs/concepts/services-networking/dual-stack/#prerequisites)和[相关操作](https://kubernetes.io/docs/concepts/services-networking/dual-stack/#enable-ipv4-ipv6-dual-stack)，创建一个新的集群。
2. 根据[Calico Kubernetes安装指引](../../02%E5%AE%89%E8%A3%85Calico/01Kubernetes/04%E8%87%AA%E5%B7%B1%E7%AE%A1%E7%90%86%E7%9A%84%E4%B8%93%E6%9C%89%E4%BA%91/01%E5%9C%A8%E4%B8%93%E6%9C%89%E4%BA%91%E5%AE%89%E8%A3%85Calico.md)，为集群和数据存储的类型下载对应的Calico manifest。
3. 编辑CNI配置（manifest中的calico-config ConfigMap），把两个字段都设置成true。

```json
    "ipam": {
        "type": "calico-ipam",
        "assign_ipv4": "true",
        "assign_ipv6": "true"
    },
```

4. 为`calico-node`容器添加下面的环境变量，配置IPv6支持：

|**变量名**|**值**
|-|-
|`IP6`|`autodetect`
|`FELIX_IPV6SUPPORT`|`true`

5. 如果**不是**用kubeadm安装的集群（见下面的说明），为`calico-node`容器添加下面的环境变量，配置默认的IPv6的IP池：

|**变量名**|**值**
|-|-
|`CALICO_IPV6POOL_CIDR`|跟kube-controller-manager和kube-proxy中配置的值保持一致

> 注意：如果是使用kubeadm创建的集群，Calico可以自动探测IPv4和IPv6的CIDR，不需要配置。

6. 应用manifest，执行`kubectl apply -f`。

新的Pod就可以拿到IPv4和IPv6地址了，它们之间的通信，以及跟外部的通信，就可以使用IPv4或IPv6了。
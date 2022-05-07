# 加固Calico的Prometheus端点

## 关于Calico的metrics端点的安全访问

使用Calico时如果开启了Prometheus的metrics，我们建议通过网络策略限制以下对metrics端点的访问。

## 条件

- 安装Calico并开启Prometheus的metrics。
- `calicoctl`[安装在PATH中并且能够访问到数据存储](../../05%E8%BF%90%E7%BB%B4/02calicoctl/01%E5%AE%89%E8%A3%85calicoctl.md)。

## 选择一种方式

这里我们给了两种通过网络策略限制Calico的Prometheus metrics端点访问的例子。根据需要进行选择。

- [黑名单模式](#黑名单模式)

这种方式默认允许主机上的所有流量，但是你可以通过Calico的策略限制某个端口的访问。这种方式就是允许你限制特定端口的访问，同时又不影响其它的主机流量。

- [白名单模式](#白名单模式)

这种方式默认会拒绝主机上所有的出入流量，然后通过明确的网络策略定义放行期望的流量。这种方式更安全，因为只有明确定义的流量才会被放行，但是这样就需要你知晓主机上应该开放的所有端口

## 黑名单模式

### 简介

基本流程如下：

1. 创建默认的网络策略，放行主机的出入流量。
2. 为所有需要加固的节点创建主机端点。
3. 创建一个网络策略，拒绝对Calico metrics端点的非必要的访问。
4. 设置标签来放行对Prometheus metrics的访问。

### calico/node示例

这里展示了如何限制对calico/node的Prometheus metrics端点的访问。

1. 创建一个默认网络策略，放行主机流量

首先，创建一个默认放行的策略。首先弄这个是为了避免在后面添加主机端点的时候出现网络断连，因为如果主机端点上面没有策略的话默认是拒绝流量的。

因此我们创建一个`default-host-policy.yaml`文件，内容如下。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-host
spec:
  # Select all Calico nodes.
  selector: running-calico == "true"
  order: 5000
  ingress:
  - action: Allow
  egress:
  - action: Allow
```

然后使用`calicoctl`使策略生效。

```shell
$ calicoctl apply -f default-host-policy.yaml
```

2. 列出所有运行Calico的节点。

```shell
$ calicoctl get nodes
```

这里，我们的集群中一共有两个节点。

```shell
NAME
kubeadm-master
kubeadm-node-0
```

3. 为每个Calico节点创建主机端点。

创建一个`host-endpoints.yaml`文件，包含上面节点的主机端点。这里我们的示例内容如下。

```yaml
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: kubeadm-master.eth0
  labels:
    running-calico: "true"
spec:
  node: kubeadm-master
  interfaceName: eth0
  expectedIPs:
  - 10.100.0.15
---
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: kubeadm-node-0.eth0
  labels:
    running-calico: "true"
spec:
  node: kubeadm-node-0
  interfaceName: eth0
  expectedIPs:
  - 10.100.0.16
```

在这个文件中，可以把`eth0`替换为节点上真实的接口名称，并且将`expectedIPS`设置为接口对应的IP地址。

注意这里的标签表明这个主机端点上运行了Calico。这个标签跟第一步中创建的网络策略的选择器相匹配。

然后，使用`calicoctl`使主机端点生效。

```shell
$ calicoctl apply -f host-endpoints.yaml
```

4. 创建网络策略限制对calico/node的Prometheus metrics端口的访问。

现在我们来创建一个网络策略，限制一下对Prometheus metrics端口的访问，然后就只有带`calico-prometheus-access: true`标签的端点才能访问这些metrics。

我们创建一个`calico-prometheus-policy.yaml`文件，内容如下。

```yaml
# Allow traffic to Prometheus only from sources that are
# labeled as such, but don't impact any other traffic.
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: restrict-calico-node-prometheus
spec:
  # Select all Calico nodes.
  selector: running-calico == "true"
  order: 500
  types:
  - Ingress
  ingress:
  # Deny anything that tries to access the Prometheus port
  # but that doesn't match the necessary selector.
  - action: Deny
    protocol: TCP
    source:
      notSelector: calico-prometheus-access == "true"
    destination:
      ports:
      - 9091
```

这个策略会选中所有带`running-calico: true`标签的端点，并且应用了一条ingress的拒绝规则。这个ingress规则拒绝了9091端口的流量，除非流量的源端带有`calico-prometheus-access: true`标签，也就限制了所有的没有携带该标签的Calico工作负载端点，主机端点，全局网络集，以及那些Calico还不知道的网络端点。

然后使用`calicoctl`使策略生效。

```shell
$ calicoctl apply -f calico-prometheus-policy.yaml
```

5. 给需要访问metrics的端点添加标签。

现在，只有带`calico-prometheus-access: true`标签的端点才能访问每个节点上的Calico的Prometheus metrics端点。要做授权的话，只需要给端点打上标签即可。

下面我们给Kubernetes的一个Pod做下授权。

```shell
$ kubectl label pod my-prometheus-pod calico-prometheus-access=true
```

如果你想给特定IP网络做授权，可以用`calicoctl`创建一个[全局网络集](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/07%E5%85%A8%E5%B1%80%E7%BD%91%E7%BB%9C%E9%9B%86.md)。

比如给你的管理子网授权。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: calico-prometheus-set
  labels:
    calico-prometheus-access: "true"
spec:
  nets:
  - 172.15.0.0/24
  - 172.101.0.0/24
```

### 使用Typha时的额外步骤

如果你的Calico使用了Kubernetes API数据存储，并且节点数大于50个，那么你可能已经安装了Typha。本节告诉你如何使用额外的网络策略来加固Typha Prometheus端点。

按照上面讲的步骤操作完后，创建一个`typha-prometheus-policy.yaml`文件，内容如下。

```yaml
# Allow traffic to Prometheus only from sources that are
# labeled as such, but don't impact any other traffic.
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: restrict-calico-node-prometheus
spec:
  # Select all Calico nodes.
  selector: running-calico == "true"
  order: 500
  types:
  - Ingress
  ingress:
  # Deny anything that tries to access the Prometheus port
  # but that doesn't match the necessary selector.
  - action: Deny
    protocol: TCP
    source:
      notSelector: calico-prometheus-access == "true"
    destination:
      ports:
      - 9093
```

这个策略选中所有携带`running-calico: true`标签的端点，执行了一条ingress拒绝规则。这个ingress规则拒绝了9093端口的流量，除非流量的源端携带了`calico-prometheus-access: true`标签，也就限制了所有的没有携带该标签的Calico工作负载端点，主机端点，全局网络集，以及那些Calico还不知道的网络端点。

然后使用`calicoctl`使策略生效。

```shell
$ calicoctl apply -f typha-prometheus-policy.yaml
```

### kube-controllers的例子

如果你的Calico会暴露出kube-controllers的metrics，你也可以用下面的网络策略来限制一下对这些metrics的访问。

创建一个`kube-controllers-prometheus-policy.yaml`文件，内容如下。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: restrict-kube-controllers-prometheus
  namespace: calico-system
spec:
  # Select kube-controllers.
  selector: k8s-app == "calico-kube-controllers"
  order: 500
  types:
  - Ingress
  ingress:
  # Deny anything that tries to access the Prometheus port
  # but that doesn't match the necessary selector.
  - action: Deny
    protocol: TCP
    source:
      notSelector: calico-prometheus-access == "true"
    destination:
      ports:
      - 9094
```

> 注意：上面的策略是设定在calico-system命名空间中的。如果你把Calico安装到了kube-system命名空间中，那么改改命名空间就行了。

然后用`calicoctl`使策略生效。

```shell
$ calicoctl apply -f kube-controllers-prometheus-policy.yaml
```

## 白名单模式

### 简介

基本流程如下：

1. 给每个需要加固的节点创建主机端点。
2. 创建网络策略放行到Calico metrics端点的流量。
3. 设置标签。

### calico/node示例

1. 查看运行Calico的节点列表。

```shell
$ calicoctl get nodes
```

这里我们集群中有两个节点。

```shell
NAME
kubeadm-master
kubeadm-node-0
```

2. 给每个Calico节点创建主机端点。

创建一个`host-endpoints.yaml`文件，包含对应的主机端点信息。示例内容如下。

```yaml
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: kubeadm-master.eth0
  labels:
    running-calico: "true"
spec:
  node: kubeadm-master
  interfaceName: eth0
  expectedIPs:
  - 10.100.0.15
---
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: kubeadm-node-0.eth0
  labels:
    running-calico: "true"
spec:
  node: kubeadm-node-0
  interfaceName: eth0
  expectedIPs:
  - 10.100.0.16
```

在这个文件中，可以把`eth0`替换为节点上真实的接口名称，并且将`expectedIPS`设置为接口对应的IP地址。

注意这里的标签表明这个主机端点上运行了Calico。这个标签跟下面创建的网络策略的选择器相匹配。

然后，使用`calicoctl`使主机端点生效。

```shell
$ calicoctl apply -f host-endpoints.yaml
```

> 注意：执行这个策略后Calico依然能够放行安全失败流量。可以修改[FelixConfiguration资源](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/05Felix%E9%85%8D%E7%BD%AE.md)中的`failsafeInboundHostPorts`和`failsafeOutboundHostPorts`进行调整。

3. 创建网络策略放行对calico/node的Prometheus metrics端口的访问。

现在我们来创建一个网络策略，限制一下对Prometheus metrics端口的访问，然后就只有带`calico-prometheus-access: true`标签的端点才能访问这些metrics。

我们创建一个`calico-prometheus-policy.yaml`文件，内容如下。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: restrict-calico-node-prometheus
spec:
  # Select all Calico nodes.
  selector: running-calico == "true"
  order: 500
  types:
  - Ingress
  ingress:
  # Allow traffic from selected sources to the Prometheus port.
  - action: Allow
    protocol: TCP
    source:
      selector: calico-prometheus-access == "true"
    destination:
      ports:
      - 9091
```

这个策略会选中所有带`running-calico: true`标签的端点，并且应用了一条ingress的放行规则。这个ingress规则针对流量的源端带有`calico-prometheus-access: true`标签，放行了9091端口的流量，也就是所有携带该标签的Calico工作负载端点，主机端点，全局网络集，都可以访问这个端口。

然后使用`calicoctl`使策略生效。

```shell
$ calicoctl apply -f calico-prometheus-policy.yaml
```

4. 给需要访问metrics的端点添加标签。

现在，只有带`calico-prometheus-access: true`标签的端点才能访问每个节点上的Calico的Prometheus metrics端点。要做授权的话，只需要给端点打上标签即可。

下面我们给Kubernetes的一个Pod做下授权。

```shell
$ kubectl label pod my-prometheus-pod calico-prometheus-access=true
```

如果你想给特定IP网络做授权，可以用`calicoctl`创建一个[全局网络集](../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/07%E5%85%A8%E5%B1%80%E7%BD%91%E7%BB%9C%E9%9B%86.md)。

比如下面的网络集可以为172.15.0.101授权。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: calico-prometheus-set
  labels:
    calico-prometheus-access: "true"
spec:
  nets:
  - 172.15.0.101/32
```

### 使用Typha时的额外步骤

如果你的Calico使用了Kubernetes API数据存储，并且节点数大于50个，那么你可能已经安装了Typha。本节告诉你如何使用额外的网络策略来加固Typha Prometheus端点。

按照上面讲的步骤操作完后，创建一个`typha-prometheus-policy.yaml`文件，内容如下。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: restrict-typha-prometheus
spec:
  # Select all Calico nodes.
  selector: running-calico == "true"
  order: 500
  types:
  - Ingress
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: calico-prometheus-access == "true"
    destination:
      ports:
      - 9093

```

这个策略选中所有携带`running-calico: true`标签的端点，执行了一条ingress放行规则。这个ingress规则针对流量的源端带有`calico-prometheus-access: true`标签，放行了9093端口的流量，也就是所有携带该标签的Calico工作负载端点，主机端点，全局网络集，都可以访问这个端口。

然后使用`calicoctl`使策略生效。

```shell
$ calicoctl apply -f typha-prometheus-policy.yaml
```

### kube-controllers的例子

如果你的Calico会暴露出kube-controllers的metrics，你也可以用下面的网络策略来限制一下对这些metrics的访问。

创建一个`kube-controllers-prometheus-policy.yaml`文件，内容如下。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: restrict-kube-controllers-prometheus
  namespace: calico-system
spec:
  selector: k8s-app == "calico-kube-controllers"
  order: 500
  types:
  - Ingress
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: calico-prometheus-access == "true"
    destination:
      ports:
      - 9094
```

然后用`calicoctl`使策略生效。

```shell
$ calicoctl apply -f kube-controllers-prometheus-policy.yaml
```
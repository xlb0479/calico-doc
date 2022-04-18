# Calico网络策略入门

## 大面儿

使用Calico网络策略规则设置哪些流量被允许，哪些被拒绝。

## 价值

### 扩展Kubernetes的网络策略

Calico网络策略比Kubernetes提供更丰富的策略能力集合：策略排序/优先级，拒绝规则，更灵活的匹配规则。Kubernetes的网络策略只能加到Pod上，Calico的网络策略可以施加于多种端点，包括Pod、VM、以及主机接口。最后，如果搭配Istio服务网格，Calico网络策略支持加固5-7层应用层的匹配准则，支持加密身份。

### 一次编写，处处管用

不管你用的云服务商是啥，使用Calico网络策略就意味着你只需编写一次策略，它是可移植的。假如你换了云服务商，不需要重写Calico网络策略。使用Calico网络策略可以很好的避免绑死在某个云服务商上。

### 与Kubernetes网络策略无缝衔接

Calico网络策略可以跟Kubernetes的网络策略一起用，或者只用一种也行。比如，你可以允许开发者给他们的微服务定义Kubernetes的网络策略。对于开发者无权覆盖的那些更广更高层的访问控制，你可以让安全团队或者运维团队来定义Calico网络策略。

## 特性

Calico的**NetworkPolicy**支持以下特性：

- 策略可以应用于任何类型的端点：Pod/容器、VM、和/或主机接口
- 可以为ingress和egress定义策略
- 策略规则支持：
    - **动作**：允许、拒绝、记录、跳过
    - **源及目的地匹配准则：**
        - 端口：端口号、端口范围、Kubernetes的命名端口
        - 协议：TCP、UDP、ICMP、SCTP、UDPlite、ICMPv6、端口号（1-255）
        - HTTP属性（如果有Istio服务网格）
        - ICMP属性
        - IP版本（IPv4、IPv6）
        - IP或CIDR
        - 端点选择器（使用标签表达式选择Pod、VM、主机接口、和/或网络集合）
        - 命名空间选择器
        - Service Account选择器
    - **可选择数据包操作管控：**在DNAT之前，关闭连接跟踪，用于被转发的流量和/或本地范围内的流量

## 概念

### 端点

Calico网络策略应用于**端点**上。在Kubernetes中，每个Pod都是一个Calico的端点。但是Calico也可以支持其他类型的端点。一共两种类型的Calico端点：**工作负载端点**（比如Kubernetes的Pod或OpenStack的VM）和**主机端点**（主机上的一个或一组接口）。

### 命名空间与全局网络策略

**Calico网络策略**是一种带命名空间的资源，它应用于该命名空间下的Pod/容器/VM。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-tcp-6379
  namespace: production
```

**Calico全局网络策略**是一种非命名空间资源，可应用于任何类型的端点（Pod、VM、主机接口），不依赖命名空间。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-tcp-port-6379
```

因为全局网络策略用的是**kind: GlobalNetworkPolicy**，从分组上就跟**kind: NetworkPolicy**分开了。比如`calicoctl get networkpolicy`就不会返回全局网络策略，而是执行`calicoctl get globalnetworkpolicy`才会看到。

### kubectl vs calicoctl

Calico网络策略和Calico全局网络策略使用calicoctl进行管理。语法跟Kubernetes类似，只有少许不同。详见[calicoctl用户手册](../../../06%E5%8F%82%E8%80%83/03calicoctl/01简介.md)

### Ingress和Egress

每条网络策略规则都要应用于**ingress**或**egress**流量。从端点（Pod、VM、主机接口）的角度来看，**ingress**属于端点的入口流量，而**egress**则是端点的出口流量。在一个Calico网络策略中，你需要单独创建ingress和egress规则（egress、ingress、或两者都用）。

你可以通过**type**字段指明策略应用于ingress、egress还是两者都用。如果没有类型字段，Calico遵从以下规则。

|**有没有Ingress规则？**|**有没有Egress规则？**|**结果**
|-|-|-
|无|无|Ingress
|有|无|Ingress
|无|有|Egress
|有|有|Ingress，Egress

### 网络流量行为：允许和拒绝

Kubernetes网络策略规范定义了以下行为：

- 如果Pod身上没有网络策略，那么它的所有出入流量都被允许。
- 如果Pod身上的网络策略包含ingress规则，那么只有规则明确允许的ingress流量才会被放行。
- 如果Pod身上的网络策略有egress规则，那么只有规则明确允许的egress流量才会被放行。

为了跟Kubernetes兼容，**Calico网络策略**对Pod保持一样的行为。对于其他端点类型（VM、主机接口），Calico网络策略是默认拒绝的。也就是说，只有明确允许的流量才会被放行，即便端点上没有网络策略。

## 开始之前

`calicoctl`使用前要进行**安装**和**配置**。`calicoctl`默认使用etcd作为数据存储，但是许多Calico的安装清单都将Kubernetes配置成了数据存储。你可以在下面的连接中了解到更多关于配置`calicoctl`的知识：

- [配置`calicoctl`](../../../05%E8%BF%90%E7%BB%B4/02calicoctl/02%E9%85%8D%E7%BD%AEcalicoctl/01%E7%AE%80%E4%BB%8B.md)

## 怎么弄

- [控制命名空间中端点的出入流量](#控制命名空间中端点的出入流量)
- [控制端点的出入流量，不管命名空间](#控制端点的出入流量，不管命名空间)
- [使用IP地址或CIDR控制端点的出入流量](#使用IP地址或CIDR控制端点的出入流量)
- [按序指定网络策略](#按序指定网络策略)
- [为特定流量生成日志](#为特定流量生成日志)

### 控制命名空间中端点的出入流量

下面的例子中，放行了**namespace: production**中带有**color: red**标签的端点的入口流量，当且仅当流量来自同一命名空间下带**color: blue**标签的Pod，并且目标端口为**6379**。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-tcp-6379
  namespace: production
spec:
  selector: color == 'red'
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: color == 'blue'
    destination:
      ports:
        - 6379
```

如果要允许其它命名空间过来的流量，就要在策略规则中使用**namespaceSelector**。namespaceSelector根据命名空间的标签进行选择。下面的例子中，如果来源命名空间能够匹配**shape == circle**，则放行对应流量。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-tcp-6379
  namespace: production
spec:
  selector: color == 'red'
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: color == 'blue'
      namespaceSelector: shape == 'circle'
    destination:
      ports:
      - 6379
```

### 控制端点的出入流量，不管命名空间

下面的例子跟上面的差不多，但是用了**kind: GlobalNetworkPolicy**，所以它不参照命名空间，应用在所有的端点上。

下面的例子中，如果流量来自标签为**color: blue**的Pod，目标是标签为**color: red**的Pod，则拒绝所有的TCP流量。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-blue
spec:
  selector: color == 'red'
  ingress:
  - action: Deny
    protocol: TCP
    source:
      selector: color == 'blue'
```

类似**kind: NetworkPolicy**，你可以在策略规则中用namespaceSelector来允许或拒绝来自指定命名空间中的流量：

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-circle-blue
spec:
  selector: color == 'red'
  ingress:
  - action: Deny
    protocol: TCP
    source:
      selector: color == 'blue'
      namespaceSelector: shape == 'circle'
```

### 使用IP地址或CIDR控制端点的出入流量

除了用选择器来定义流量规则，还可以用CIDR来声明。

这个例子中，如果出口流量是来自带有标签**color: red**的Pod，并且目标IP落在**1.2.3.4/24**中，那么就放行这些流量。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-egress-external
  namespace: production
spec:
  selector:
    color == 'red'
  types:
    - Egress
  egress:    
    - action: Deny
      destination:
        nets:
        - 1.2.3.0/24
```

### 按序指定网络策略

如果要控制网络策略使用的顺序，可以使用**order**字段（值越小优先级越高）。如果**action: allow**和**action: deny**可能施加到同样的端点上，那么此时定义策略的**order**就显得尤为重要了。

在下面的例子中，**allow-cluster-internal-ingress**策略（order: 10）在**drop-other-ingress**策略（order: 20）之前执行。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: drop-other-ingress
spec:
  order: 20
  ...deny policy rules here...
```

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-cluster-internal-ingress
spec:
  order: 10
  ...allow policy rules here...
```

### 为特定流量生成日志

下面的例子中，拒绝了应用的入口TCP流量，并且在syslog中记录了每次连接尝试。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
Metadata:
  name: allow-tcp-6379
  namespace: production
Spec:
  selector: role == 'database'
  types:
  - Ingress
  - Egress
  ingress:
  - action: Log
    protocol: TCP
    source:
      selector: role == 'frontend'
  - action: Deny
    protocol: TCP
    source:
      selector: role == 'frontend'
```
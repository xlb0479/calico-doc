# 在策略中使用ICMP/ping规则

## 大面儿

使用Calico网络策略允许或拒绝ICMP/ping消息。

## 价值

**因特网控制报文协议（ICMP）**提供了有价值的网络探测功能，但也会被恶意使用。攻击者可以用它来了解你的网络，或者做DoS攻击。使用Calico网络策略，你可以控制哪里可以使用ICMP。比如，你可以：

- 允许ICMP ping，但只能是工作负载、主机断点（或二者皆有）
- 允许运维人员做诊断时的专用Pod可以使用ICMP，但其它Pod不能用
- 临时允许ICMP用于问题诊断，研究明白后再禁用
- 拒绝/允许ICMPv4和/或ICMPv6

## 特性

本文使用了Calico的以下特性：

**GlobalNetworkPolicy**或**NetworkPolicy**及其：

- 匹配ICMPv4和ICMPv6的协议
- 根据ICMP的type和code匹配icmp/非ICMP

## 概念

### ICMP的type和code

Calico网络策略可以根据ICMP的type和code进行管控。比如可以声明ICMP type5、code2，匹配特定的ICMP转发数据包。

详见[ICMP的type和code](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol#Control_messages)。

## 怎么弄

- [拒绝所有ICMP，所有的工作负载和主机端点](#拒绝所有ICMP，所有的工作负载和主机端点)
- [允许所有ICMP ping，所有工作负载和主机端点](#允许所有ICMP%20ping，所有工作负载和主机端点)
- [根据协议type和code放行ICMP，所有Pod](#根据协议type和code放行ICMP，所有Pod)

### 拒绝所有ICMP，所有的工作负载和主机端点

现在，我们来看一个“拒绝所有ICMP”的**GlobalNetworkPolicy**。

这个策略**选中了所有工作负载和主机端点**。它为所有工作负载和主机端点开启了一个默认拒绝的策略，并且还明确添加了ICMP的拒绝规则。

如果你还想让某些流量被放行，那就得让常规的“放行”策略在全局的拒绝所有ICMP流量策略之前生效。

在下面的例子中，所有的工作负载和主机端点都无法发送或接收**ICMPv4**和**ICMPv6**消息。

如果你的环境中不需要**ICMPv6**消息，那最好是按照下面的例子明确关掉它。

不论在什么样的“拒绝所有”类型的Calico网络策略中，确保给它设置了一个比较低的顺位（**order:200**），比普通的放行策略要低就行。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: block-icmp
spec:
  order: 200
  selector: all()
  types:
  - Ingress
  - Egress
  ingress:
  - action: Deny
    protocol: ICMP
  - action: Deny
    protocol: ICMPv6
  egress:
  - action: Deny
    protocol: ICMP
  - action: Deny
    protocol: ICMPv6
```

### 允许所有ICMP ping，所有工作负载和主机端点

本例中，工作负载和主机端点可以接收来自其它工作负载和主机端点的**ICMPv4 type 8**和**ICMPv6 type 128**ping请求。

其它的流量可以被其它策略做放行。如果流量没有被明确放行，那就被默认拒绝了。

这个策略只应用到了**ingress**流量。（egress流量不受影响，不会给egress做默认拒绝。）

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-ping-in-cluster
spec:
  selector: all()
  types:
  - Ingress
  ingress:
  - action: Allow
    protocol: ICMP
    source:
      selector: all()
    icmp:
      type: 8 # Ping request
  - action: Allow
    protocol: ICMPv6
    source:
      selector: all()
    icmp:
      type: 128 # Ping request
```

### 根据协议type和code放行ICMP，所有Pod

本例中，只有匹配到**projectcalico.org/orchestrator == 'kubernetes'**选择器的Pod，才会被允许接收ICMPv4的**code: 1 # host unreachable**消息。

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-host-unreachable
spec:
  selector: projectcalico.org/orchestrator == 'kubernetes'
  types:
  - Ingress
  ingress:
  - action: Allow
    protocol: ICMP
    icmp:
      type: 3 # Destination unreachable
      code: 1 # Host unreachable
```
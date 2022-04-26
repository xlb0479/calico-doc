# 保护Kubernetes节点

## 大面儿

加固Calico管理的Kubernetes节点的主机端点

## 价值

Calico可以为Kubernetes节点自动创建主机端点。也就是说随着集群的变化Calico可以自主管理主机端点的生命周期，确保每个节点总是处于策略的保护中。

## 特性

本文用到了Calico的以下特性：

- **HostEndpoint**
- **KubeControllersConfiguration**
- **GlobalNeworkPolicy**

## 概念

### 主机端点

每个主机都会有一个或多个跟外部通信的网络接口。在Calico中可以用主机端点来表达这些接口，然后在网络策略中进行加固。

Calico的主机端点可以打标签，跟工作负载的标签用法一样。然后在网络策略规则中使用标签选择器进行选择。

自动化主机端点可以加固主机的所有接口（例如在Linux中主机网络空间中的所有接口）。设置`interfaceName: "*"`即可。

### 自动化主机端点

Calico会为每个节点创建一个通配的主机端点，让主机端点的标签和IP地址跟所在节点的信息保持一致。Calico会定期把主机端点的标签和IP地址跟节点的信息进行同步。这就意味着针对这些自动化主机端点的策略只要选中了这些端点那么就会一直保持正确工作，即便节点的IP或标签发生了变化。

自动化主机端点跟其它的主机端点可以通过标签`projectcalico.org/created-by: calico-kube-controllers`进行区分。可以通过配置KubeControllersConfiguration资源来启用或禁用自动化主机端点。

## 开始之前……S

有一个Calico集群，装好`calicoctl`。

## 怎么弄

- [启用自动化主机端点](#启用自动化主机端点)
- [为自动化主机端点配置网络策略](#为自动化主机端点配置网络策略)

### 启用自动化主机端点

启用自动化主机端点，需要编辑默认的KubeControllersConfiguration实例，将`spec.controllers.node.hostEndpoint.autoCreate`设置为`true`。

```shell
$ calicoctl patch kubecontrollersconfiguration default --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```

ok的话，就会为集群的每个节点创建一个主机端点。

```shell
$ calicoctl get heps -owide
```

输出结果如下：

```shell
$ calicoctl get heps -owide
NAME                                                    NODE                                           INTERFACE   IPS                              PROFILES
ip-172-16-101-147.us-west-2.compute.internal-auto-hep   ip-172-16-101-147.us-west-2.compute.internal   *           172.16.101.147,192.168.228.128   projectcalico-default-allow
ip-172-16-101-54.us-west-2.compute.internal-auto-hep    ip-172-16-101-54.us-west-2.compute.internal    *           172.16.101.54,192.168.107.128    projectcalico-default-allow
ip-172-16-101-79.us-west-2.compute.internal-auto-hep    ip-172-16-101-79.us-west-2.compute.internal    *           172.16.101.79,192.168.91.64      projectcalico-default-allow
ip-172-16-101-9.us-west-2.compute.internal-auto-hep     ip-172-16-101-9.us-west-2.compute.internal     *           172.16.101.9,192.168.71.192      projectcalico-default-allow
ip-172-16-102-63.us-west-2.compute.internal-auto-hep    ip-172-16-102-63.us-west-2.compute.internal    *           172.16.102.63,192.168.108.192    projectcalico-default-allow
```

### 为自动化主机端点配置网络策略

如果要为所有Kubernetes节点设置网络策略，首先要给节点打一个标签。这个标签会被同步到它们的自动化主机端点上。

例如，给所有的节点和主机端点打一个**kubernetes-host**标签：

```shell
$ kubectl label nodes --all kubernetes-host=
```

策略示例：

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: all-nodes-policy
spec:
  selector: has(kubernetes-host)
  <rest of the policy>
```

如果只想选中一部分主机端点（机器对应的Kubernetes节点），那么就需要在策略中指定只属于这部分主机端点的标签。例如，我们给node1和node2打上**environment=dev**：

```shell
$ kubectl label node node1 environment=dev
$ kubectl label node node2 environment=dev
```


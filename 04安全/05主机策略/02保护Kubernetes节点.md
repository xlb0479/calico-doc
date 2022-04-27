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

打好标签之后，并且开启了自动化主机端点，那么node1和node2的主机端点就会自动加上**environment=dev**标签。我们可以组合一下选择方式，只选择这部分节点：

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: some-nodes-policy
spec:
  selector: has(kubernetes-host) && environment == 'dev'
  <rest of the policy>
```

## 教程

这里我们会拒掉Kubernetes节点的ingress流量，只允许SSH以及一些Kubernetes必需的端口。我们要创建两个策略：一个给master节点，一个给工作节点。

> 注意：这里我们的测试集群是在AWS上面用kubeadm v1.18.2创建的，使用了“stacked etcd”模式的[拓扑](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)。这种拓扑就是说etcd的Pod运行在master节点上。kubeadm默认都是用这种模式安装的。<br/>如果你的Kubernetes集群用的是不一样的平台，比如是Kubernetes的某种衍生品，或者是使用了外部的etcd集群，那么需要审查master和worker节点需要的端口，根据需要调整你的策略。

首先，我们限制master节点的ingress流量。下面给出的ingress策略包含了三条规则。第一条是允许从其它任意位置访问API server的端口。第二条是允许所有连接localhost的流量，也就是允许Kubernetes访问其控制面进程。这些控制面进程包括etcd服务器客户端API、调度器、以及controller-manager。这条规则还同时允许访问本地的kubelet API和calico/node安全检查。最后一条规则是允许etcd Pod之间建立对等，并且允许master互相访问各自的kubelet API。

如果你没有改过安全失败（failsafe）端口，那么执行该策略后应该依然可以正常SSH连接到这些节点上。现在我们为Kubernetes的master节点执行下这个策略：

```yaml
calicoctl apply -f - << EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: ingress-k8s-masters
spec:
  selector: has(node-role.kubernetes.io/master)
  # This rule allows ingress to the Kubernetes API server.
  ingress:
  - action: Allow
    protocol: TCP
    destination:
      ports:
      # kube API server
      - 6443
  # This rule allows all traffic to localhost.
  - action: Allow
    destination:
      nets:
      - 127.0.0.0/8
  # This rule is required in multi-master clusters where etcd pods are colocated with the masters.
  # Allow the etcd pods on the masters to communicate with each other. 2380 is the etcd peer port.
  # This rule also allows the masters to access the kubelet API on other masters (including itself).
  - action: Allow
    protocol: TCP
    source:
      selector: has(node-role.kubernetes.io/master)
    destination:
      ports:
      - 2380
      - 10250
EOF
```

注意上面的策略选择的是标准的**node-role.kubernetes.io/master**标签，是kubeadm给master节点打的。

下一步，我们需要设置工作节点的策略。在创建策略之前我们先给所有工作节点打个标签，然后它会更新到自动化主机端点上。这里我们使用**kubernetes-worker**。示例命令如下：

```shell
$ kubectl get node -l '!node-role.kubernetes.io/master' -o custom-columns=NAME:.metadata.name | tail -n +2 | xargs -I{} kubectl label node {} kubernetes-worker=
```

这个策略有两条规则。第一条放行所有对localhost的访问。和master一样，工作节点也需要访问它们localhost上的kubelet API和calico/node健康检查。第二条规则是允许master访问工作节点的kubelet API。开搞：

```yaml
calicoctl apply -f - << EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: ingress-k8s-workers
spec:
  selector: has(kubernetes-worker)
  # Allow all traffic to localhost.
  ingress:
  - action: Allow
    destination:
      nets:
      - 127.0.0.0/8
  # Allow only the masters access to the nodes kubelet API.
  - action: Allow
    protocol: TCP
    source:
      selector: has(node-role.kubernetes.io/master)
    destination:
      ports:
      - 10250
EOF
```
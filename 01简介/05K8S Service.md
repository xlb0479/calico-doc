# K8S Service

> 这里包含了一些相关的背景知识，不仅限于Calico。

在这里你能学习到：

- 什么是K8S的Service？
- 三种Service之间的区别是什么，分别用来干啥？
- Service和网络策略是如何联动的？
- Service的一些优化方法。

## 什么是K8S的Service？

K8S [Service](https://kubernetes.io/docs/concepts/services-networking/service/)将一组Pod的访问抽象成了一个网络服务。每个服务后面的这组Pod通常使用[标签选择器](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)来定义。

当一个客户端连接到一个K8S Service时，连接在对应的多个Pod上进行负载均衡，如下概念图所示：

![img](https://projectcalico.docs.tigera.io/images/k8s-service-concept.svg)

一共三种K8S的Service：

- Cluster IP - 常用于集群内部对Service的访问
- Node port - 从集群外访问Service的最基本的方法
- Load balancer - 使用外部负载均衡器，是一种更复杂的从外部访问集群Service的方法。

## Cluster IP

默认的Service类型就是`ClusterIP`。这种方式可以让Service在集群中通过一个虚拟IP地址进行访问，也就是一个Service的Cluster IP。Service的Cluster IP时可以通过K8S的DNS进行发现的。比如，`my-svc.my-namespace.svc.cluster-domain.example`。DNS名和Cluster IP在Service的生命周期中保持不变，即便后端的Pod可能会增加或销毁，数量随时变化。

在一个典型的K8S部署中，kube-proxy运行在每个节点上，拦截对Cluster IP地址的连接，在Service后端的Pod上进行负载均衡。作为这个过程的一部分，[DNAT](02网络.md#nat)用来实现目标IP地址映射，将Cluster IP映射到某个选中的Pod上。在该链接上响应的数据包经过反向NAT原路返回到发起该连接的Pod。

![img](https://projectcalico.docs.tigera.io/images/kube-proxy-cluster-ip.svg)

划重点，网络策略是施行在Pod上的，而不是Cluster IP上。（比如egress网络策略就是在DNAT将连接的目标IP改为选中的Pod之后才开始发挥作用的。因为连接中只改变了目标IP，对于后端Pod的ingress网络策略看到的连接的源地址是原始的客户端Pod的地址。）

## Node port

从集群外访问Service的最基础的方法就是使用`NodePort`类型的Service。一个Node Port就是指集群中每个节点都要保留的一个端口，通过它就可以访问服务。在一个典型的K8S部署中，kube-proxy要负责拦截对Node Port发起的连接，然后在Service后端对应的Pod上进行负载均衡。

作为这个过程中的一部分，[NAT](02网络.md#nat)用来将目标IP和端口从节点IP和Node Port映射到选中的Pod和服务端口上。而且还要把源IP从客户端IP映射成节点IP，这样在该连接上返回的包就可以回到最初的节点上，然后就可以做反向NAT了。（只有负责NAT的节点才有用来做反向NAT的连接跟踪状态。）

![img](https://projectcalico.docs.tigera.io/images/kube-proxy-node-port.svg)

注意因为连接的源IP地址都被SNAT成了节点的IP地址，在Service后端的Pod上的ingress网络策略看不到原始的客户端IP。这种情况通常意味着任何此类策略只能限制目标协议和端口，但是不能根据客户端/源IP地址进行限制。这种限制在某些场景中可以用[externalTrafficPolicy](#externalTrafficPolicy:local)来进行规避，或者使用Calico的eBPF数据面[原生Service处理](#Calico的eBPF原生Service处理)（而不是用kube-proxy），它可以保留源IP地址。

## Load balancer

`LoadBalancer`类型的Service通过一个外部网络负载均衡器（NLB）来暴露服务。具体的网络负载均衡器类型依赖于公有云供应商，或者如果是在专有云中，要看你的集群集成了什么样的硬件负载均衡设备。

可以通过网络负载均衡器上的指定IP，从集群外部访问Service，然后会在各个节点上通过Service的节点端口进行负载均衡。

![img](https://projectcalico.docs.tigera.io/images/kube-proxy-load-balancer.svg)

大部分网络负载均衡器都可以保留客户端的源IP，但是由于还要走一层节点端口，后端Pod看不到客户端IP，同样会影响到网络策略。因为用了节点端口，这个问题在某些场景下可以通过[externalTrafficPolicy](#externaltrafficpolicylocal)，或者用Calico的eBPF数据面[原生Service处理](#Calico的eBPF原生Service处理)（而不是用kube-proxy）来进行规避，它们可以保留源IP地址。

## Advertise服务IP

除了Node port和网络负载均衡器，还可以选择通过BGP来对服务IP进行advertise。这就需要集群所在的底层网络上支持BGP，一般就是要在专有部署场景中具备标准的ToR路由器。

Calico支持对Service的Cluster IP，或者配置的External IP进行advertise。如果你没用Calico当网络插件，那么[MetalLB](https://github.com/metallb/metallb)也可以提供类似的能力，支持各种不同的网络插件。

![img](https://projectcalico.docs.tigera.io/images/kube-proxy-service-advertisement.svg)

## externalTrafficPolicy:local

默认情况下，不管你是用`NodePort`、`LoadBalancer`还是通过BGP进行advertise，从集群外部访问Service时都会在后端的Pod上进行负载均衡，不管这个Pod在哪个节点上。这种行为可以通过配置Service的`externalTrafficPolicy:local`来调整，改完之后，连接只能在节点本地的Pod上进行负载均衡。

如果此时使用了`LoadBalancer`类型的Service，或者是用Calico来对Service的IP地址进行advertise，流量只会指向那些至少拥有一个该Service后端Pod的节点上。这样可以减少额外的节点间hop，而且，更重要的是，可以一直把源IP一路保留直到Pod，这样就可以通过网络策略来限制特定的外部客户端访问了。

![img](https://projectcalico.docs.tigera.io/images/kube-proxy-service-local.svg)

注意这里如果使用了`LoadBalancer`类型的Service，并不是所有的负载均衡器都支持这种模式。而如果用了advertise，负载均衡的平衡性就依赖于拓扑。此时可以用Pod的反亲和性让后端Pod在你的拓扑中均衡分布，但这也为服务的部署增加了复杂性。

## Calico的eBPF原生Service处理

作为K8S标准kube-proxy的替代方案，Calico的[eBPF 数据面](../05运维/07eBPF数据面/02开启eBPF数据面.md)支持原生服务处理。它可以保留源IP来简化网络策略，提供DSR（服务端直接返回）来减少返回时的网络hop，提供不依赖于拓扑的负载均衡，而且比kube-proxy的CPU占用、延迟都更小。

![img](https://projectcalico.docs.tigera.io/images/calico-native-service-handling.svg)
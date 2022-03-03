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

## Calico的eBPF原生Service处理

## externalTrafficPolicy:local
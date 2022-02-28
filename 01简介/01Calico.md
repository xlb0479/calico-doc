# 什么是Calico？

![img](https://projectcalico.docs.tigera.io/images/felix_icon.png)

Calico是为容器、虚拟机以及普通主机上的工作负载搞的一套网络及网络安全的解决方案。Calico支持的平台很多，包括K8S，OpenShift，MKE，OpenStack，以及裸金属服务。

不管你是用Calico的eBPF数据面还是标准的Linux网络模式，Calico都能在带来完全云原生扩展能力的同时提供强大的性能保障。不管是在公有云、专有云、单节点，还是上千个节点的集群中，Calico都能为开发者和集群管理者提供一致的体验和功能。

# 为什么要用Calico？

![img](https://projectcalico.docs.tigera.io/images/intro/multiple-dataplanes.png)

## 多种数据面

Calico有多种数据面，包括纯Linux eBPF数据面、标准Linux网络数据面、Windows HNS数据面。不管你是想使用最前沿的eBPF，还是使用系统管理员们已经熟知的那些标准，Calico都可以满足你。

不管你用哪种，你都会获得相同的、同样简单的基础网络以及网络策略和IP地址管理能力，这也使得Calico成为了对于任务密集型的云原生应用最受信赖的网络及网络策略解决方案。

## 网络安全的最佳实践

![img](https://projectcalico.docs.tigera.io/images/intro/best-practices.png)

Calico的富网络策略模型让它能够轻松的封锁通信，这样就只存在被你允许的网络流量。加上内置的Wireguard加密支持，对于pod间网络流量的加固变得非常简单。

Calico的策略引擎可以强制在主机网络层和（如果上了Isito和Envoy）服务网格层使用相同的策略模型，不管是脆弱的基础设施还是脆弱的工作负载，都可以得到很好的保护，不会互相影响。

## 性能

![img](https://projectcalico.docs.tigera.io/images/intro/performance.png)

根据你的选择，Calico会使用Linux eBPF或Linux内核的高度优化的标准网络管道来实现高性能网络。Calico的网络选项十分灵活，在绝大多数场景中都无需overlay，避免了数据包的拆解带来的损耗。Calico的数据面以及策略引擎都已经过多年的生产环境优化，将整体CPU的占用率降到了最低。

## 伸缩性

![img](https://projectcalico.docs.tigera.io/images/intro/scale.png)

Calico的核心设计原则采用了云原生设计模式的最佳实践，结合了全世界大型网络运营商都信赖的基础网络协议标准。这样就带来了优异的伸缩性，已经在大规模生产环境中使用了多个年头。Calico的开发测试闭环包含了常规的上千节点集群测试。不管你是10个，100个，还是更多个节点，大型K8S集群所需的高性能以及伸缩性都可以很好的满足。

## 互操作性

![img](https://projectcalico.docs.tigera.io/images/intro/interoperability.png)

Calico可以让K8S应用、非K8S应用、或遗留系统进行无缝且安全的通信。在你的网路中，K8S的pod是一等公民，能够跟网络中的其他任意系统进行通信。而且Calico还能无缝的将你已有的基于主机的系统
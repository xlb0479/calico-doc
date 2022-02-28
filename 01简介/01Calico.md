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

Calico可以让K8S应用、非K8S应用、或遗留系统进行无缝且安全的通信。在你的网路中，K8S的pod是一等公民，能够跟网络中的其他任意系统进行通信。而且Calico还能无缝地将它的安全能力扩展到你已有的和K8S协同的基于主机的系统（不管是在公有云、专有云、VM、还是裸金属服务器上）。所有的工作负载都会满足同样的网络策略模型，所以能出现的网络流量都是你所期待的流量。

## 真实的生产环境应用

![img](https://projectcalico.docs.tigera.io/images/intro/deployed.png)

很多大企业，包括SaaS供应商、金融服务企业、以及制造商，都选择相信Calico并在生产环境中使用了Calico。最大的公有云提供商选择Calico来为他们的K8S服务（Amazon EKS、Azure AKS、Google GKE、IBM IKS）提供网络安全。

## 完全支持K8S的网络策略

![img](https://projectcalico.docs.tigera.io/images/intro/policy.png)

在开发API的时候，Calico的网络策略引擎就实现了K8S的网络策略。Calico的卓越之处在于，这些API定义时所设想的那些功能以及灵活性，Calico全都把它们实现出来了。如果用户想要更多功能，Calico可以支持扩展的网络策略能力，可以无缝地跟K8S的API一起工作，甚至可以提供比用户定义的网络策略更好的灵活性。

## 贡献者

![img](https://projectcalico.docs.tigera.io/images/intro/community.png)

Calico开源项目能有今天，离不开来自各个公司200+的贡献者。此外，Calico背靠Tigera，它是由最初的Calico工程师团队建立的，致力于保持Calico在K8S网络策略这块领先的标准地位。

## Calico Cloud的兼容性

Calico Cloud基于开源的Calico打造，提供以下K8S安全及可观察性的特性及能力：

- Egress访问控制（DNS策略、egress网关）
- 将防火墙扩展到K8S
- 层次分级
- 基于FQDN/DNS的策略
- 在主机/VM/容器间的微分段（Micro-segmentation）
- 合规报告与告警
- K8S入侵检测与防御（IDS/IPS）
- SIEM集成
- 应用层（L7）可观测性
- 动态数据包捕获
- DNS面板
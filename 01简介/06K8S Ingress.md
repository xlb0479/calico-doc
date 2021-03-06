# K8S Ingress

> 这里包含了一些相关的背景知识，不仅限于Calico。

你将在这里学习到：

- 什么是K8S的Ingress？
- 为什么要用Ingress？
- 不同Ingress实现间的区别是什么？
- Ingress和网络策略是如何协作的？
- Ingress和Service底层是如何协作的？

## 什么是K8S的Ingress？

K8S的Ingress构建于K8S的[Service](05K8S%20Service.md)之上，在应用层提供负载均衡，根据特定域名或URL将HTTP和HTTPS请求映射到K8S的Service上。Ingress还可以用来在Service的负载均衡之前终结SSL/TLS。

Ingress的具体实现依赖于你用的[Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)。Ingress Controller用来监控K8S的[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)资源，提供/配置一个或多个Ingress负载均衡器，实现所需的负载均衡动作。

不同于K8S的Service，它是工作在网络层（L3-4），Ingress负载均衡是工作在应用层（L5-7）。进来的连接会终结在负载均衡设备器这里，这样它就可以判别每一个HTTP/HTTPS请求。然后这些请求通过不同的连接从负载均衡器转发到选中的Service后端Pod上。结果就是，后端Pod的网络策略可以限制只允许从负载均衡器过来的连接，但是无法根据特定的原始客户端进行限制。

## 为什么要用K8S的Ingress？

既然K8S的[Service](05K8S%20Service.md)已经提供了一个将集群外访问Service的负载均衡机制，为啥你还要用K8S的Ingress呢？

主要的出发点是假设你有多个HTTP/HTTPS的服务，然后向通过一个单独的外部IP暴露出去，每个服务只配置不同的URL路径，或者是不同的域名。从客户端配置的角度来看，这样要比用K8S Service将每个服务单独暴露到集群外部的方式配置起来更简单，后者每个服务还会有一个单独的外部IP。

另一方面，假设你的应用架构有一个总的前端微服务扛在最前面，用K8S Service可能已经满足你的需要了。此时你可能觉得不需要再引入Ingress了，既考虑了整体的简单性，也可以通过网络策略来根据特定客户端进行访问控制。结果就是，你的前端扛把子已经扮演了K8S Ingress的角色，类似于我们后面讲到的[集群内Ingress](#集群内Ingress)的方案。

## 各种Ingress方案

广义上讲，分为两类：

- 集群内Ingress - Ingress负载均衡由集群内的Pod负责。
- 外部Ingress - Ingress负载均衡由集群外的小玩意儿或者云服务商的能力提供。

### 集群内Ingress

集群内Ingress使用集群内Pod中的软件负载均衡。有好多不同的Ingress控制器实现了这种模式，比如NGINX Ingress Controller。

这种模式的优点在于你可以：

- 水平扩展你的Ingress方案直到K8S的上限
- 选择最适合你的Ingress控制器，比如需要特殊的负载均衡算法，或者安全考量。

要想将ingress流量发到集群内的ingress的Pod上，这些ingress的Pod可以暴露成普通的K8S Service，这样就可以按照普通的Service从集群外进行访问。一种常用的方法是使用外部网络负载均衡器或者Service IP Advertisement，并且设置`externalTrafficPolicy:local`。这样最小化了网络hop，并且保留了客户端的源IP地址，可以让网络策略根据特定客户端来限制对ingress Pod的访问。

![img](https://projectcalico.docs.tigera.io/images/ingress-in-cluster.svg)

### 外部Ingress

外部Ingress的方案使用了集群外的应用负载均衡器。具体细节依赖于你选用的Ingress Controller，大部分云服务商都包含了一个Ingress Controller，可以自动提供并管理云服务商的应用负载均衡，提供Ingress。

这种类型的优点在于，Ingress的复杂性交给云服务商了。缺点就是可能没有集群内Ingress方案那么多的特性，而且能够通过Ingress暴露出来的服务的数量也被云服务商限制住了。

![img](https://projectcalico.docs.tigera.io/images/ingres-external.svg)

注意了，大部分的应用负载均衡都支持基本的通过[Node Port](05K8S%20Service.md#Node%20port)的方式将流量转发都选中的后端Pod上。

除了这种基本的Node Port的方式，一些云服务商还提供第二种应用层负载均衡的方法，可以直接在后端Pod上进行负载均衡，无需通过node-port或kube-proxy服务的处理。这种方式可以在对跨界点Pod负载均衡时消除二次网络hop。潜在的缺点就是如果你的规模特别大，比如一个Service后面几百个Pod，可能会超出应用层负载均衡可以操作的最大IP数量。此时换成集群内Ingress的方案可能更合适。

## 深挖一下！

上面的图关注的都是Ingress和Service连接（L5-7）的表现形式。可以在[K8S Service](05K8S%20Service.md)中学习到更多关于处理连接时涉及到的网络（L3-4）交互细节，包括什么时候能将客户端源IP保留下来。

如果你已经准备好了解Service底层的工作细节了，这里有一些更详细的图，展示了Service在负载均衡时网络层（L3-4）上的细节。

> 注意：即便不了解这个层面的知识你也可以使用Ingress把一切搞定！如果不想了解底层的这些知识，跳过就行了。

**集群内Ingress方案，暴露成了`LocalBalancer`类型的Service，并且设置了`externalTrafficPolicy:local`**

![img](https://projectcalico.docs.tigera.io/images/ingress-in-cluster-nlb-local.svg)

**集群外Ingress方案，使用Node Port**

![img](https://projectcalico.docs.tigera.io/images/ingress-external-node-ports.svg)

**集群外Ingress方案，直达Pod**

![img](https://projectcalico.docs.tigera.io/images/ingress-external-direct-to-pods.svg)
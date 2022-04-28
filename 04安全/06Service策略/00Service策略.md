# Service策略

为Kubernetes的节点端口创建Calico策略，以及那些通过集群IP暴露到外面的Service。

## [为Kubernetes的节点端口设置策略](01为Kubernetes的节点端口设置策略.md)

使用Calico全局网络策略限制对Kubernetes节点端口的访问。一步一步加固主机，节点端口，以及集群。

## [为暴露到集群外的集群IP服务设置策略](02为暴露到集群外的集群IP服务设置策略.md)

使用Calico基于BGP将集群IP服务暴露出去，并且通过Calico网络策略限制对它的访问。
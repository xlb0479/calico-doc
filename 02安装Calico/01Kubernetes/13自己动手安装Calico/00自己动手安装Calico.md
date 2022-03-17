# 自己动手安装Calico

准备好接受挑战啦？这里带你了解Calico安装过程中每一步的细节。

## [介绍](01介绍.md)

简单介绍下先。

## [准备Kubernetes](02准备Kubernetes.md)

搞起一个Kubernetes集群。

## [Calico数据存储](03Calico数据存储.md)

为集群运维和配置状态准备的中央数据存储。

## [配置IP池](04配置IP池.md)

快速了解集群的IP池定义（IP地址范围）。

## [安装CNI插件](05安装CNI插件.md)

安装Calico Container Network Interface（CNI）

## [安装Typha](06安装Typha.md)

了解规模化部署用到的Typha。

## [安装calico/node](07安装calico/node.md)

以daemon set方式配置并安装calico/node。

## [配置BGP对等](08配置BGP对等.md)

BGP对等配置一览。

## [测试网络](09测试网络.md)

测试网络是否正常。

## [测试网络策略](10测试网络策略.md)

验证网络策略。

## [用户RBAC](11用户RBAC.md)

生产环境集群用到的角色和访问控制一览。

## [Istio集成](12Istio集成.md)

为Istio服务网格应用施加Calico网络策略。
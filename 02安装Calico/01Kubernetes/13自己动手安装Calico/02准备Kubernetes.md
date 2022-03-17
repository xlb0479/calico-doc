# 准备Kubernetes

我们要在一个Kubernetes集群上安装Calico。为了展示一个高可用的Calico控制面，本文我们要使用五个节点。本场实验要教你在AWS上用kubeadm创建一个Kubernetes集群。

## 添加EC2节点

1. 添加五个节点
    1. Ubuntu 20.04 LTS - Focal
    2. T2.medium
    3. 确保实例都在同一个子网下，安全组策略要允许节点间自由通信。
    4. 每个实例的弹性网卡要关闭Source/Destination检查。
2. 每个节点安装Docker
    1. `sudo apt update`
    2. `sudo apt install docker.io`
    3. `sudo systemctl enable docker`

## 安装Kubernetes

1. 参照[官方文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)安装kubeadm，kubelet，kubectl。
2. 选择一个节点作为master。在该节点上执行`kubeadm init --pod-network-cidr=192.168.0.0/16` <br/>这个Kubernetes的`pod-network-cidr`是集群中所有Pod的IP前缀。这个范围不能跟你VPC中的其它网络冲突。

3. 在所有其它节点上执行`sudo kubeadm join <output from kubeadm init>`
4. 复制管理员凭据
5. 测试访问
    1. 运行<br/>`kubectl get nodes`<br/>确认所有节点都加入进来了。此时节点应该都加入了但状态是`NotReady`，因为Kubernetes无法找到一个网络服务和配置。
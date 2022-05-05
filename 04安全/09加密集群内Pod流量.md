# 加密集群内Pod流量

## 大面儿

开启WireGuard，加固集群内的Pod流量。

## 价值

打开这个特性后，Calico自动在节点间创建并管理WireGuard隧道，为集群内Pod的流量在线路上提供传输级别的安全保障。WireGuard提供了[形式化验证](https://www.wireguard.com/formal-verification/)加固，以及[高性能隧道](https://www.wireguard.com/performance/)，而且无需任何特殊的硬件设备。关于WireGuard的具体实现，可以去看它的[白皮书](https://www.wireguard.com/papers/wireguard.pdf)。

## 特性

这里我们用到了Calico的以下特性：

- **Felix配置资源**及其WireGuard相关的配置参数

## 开始之前……

### 支持

下面的平台中只能用IPv4：

- Kubernetes，专有云部署
- 使用Calico CNI的EKS
- 使用AWS CNI的EKS
- 使用Azure CNI的AKS

在上面列出的平台上都可以实现Pod-to-Pod流量加密。此外，使用AKS或EKS的话，host-to-host流量也会被加密，包括使用主机网络的Pod。

- [安装并配置calicoctl](../05%E8%BF%90%E7%BB%B4/02calicoctl/01%E5%AE%89%E8%A3%85calicoctl.md)
- 确认操作系统是否[支持WireGuard](https://www.wireguard.com/install/)。
- Calico中的WireGuard需要拿到节点IP才能建立安全隧道。Calico可以使用[IP设置](../06%E5%8F%82%E8%80%83/06calico-node.md#IP设置)以及[calico/node](../06%E5%8F%82%E8%80%83/06calico-node.md)资源中的[IP自动探测方法](../06%E5%8F%82%E8%80%83/06calico-node.md#IP自动探测方法)来探测到节点的IP地址。
    - 将`IP`（或`IP6`）环境变量设置为`autodetect`。
    - 为`IP_AUTODETECTION_METHOD`（或`IP6_AUTODETECTION_METHOD`）设置适当的值。如果节点上有多个接口，就把这个值设置成探测主接口IP的模式。

## 怎么弄

- [安装WireGuard](#安装WireGuard)
- [启用WireGuard](#启用WireGuard)
- [关闭特定节点的WireGuard](#关闭特定节点的WireGuard)
- [验证配置](#验证配置)
- [关闭WireGuard](#关闭WireGuard)

### 安装WireGuard

WireGuard包含在了Linux 5.6+的内核中，并且在某些发行版中做了向后移植，支持更早的Linux内核。

根据[操作系统指南](https://www.wireguard.com/install/)为集群节点安装WireGuard。安装完后可能需要重启，这样内核模块才能变为可用。

对于下面的平台，按照对应的操作完成WireGuard的安装，WireGuard的安装说明中没有针对这些平台的说明。

#### EKS

在默认的Amazon Machine Image（AMI）中安装WireGuard：

```shell
$ sudo yum install kernel-devel-`uname -r` -y
$ sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y
$ sudo curl -o /etc/yum.repos.d/jdoss-wireguard-epel-7.repo https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
$ sudo yum install wireguard-dkms wireguard-tools -y
```

此外，可以使用下面的命令启用host-to-host加密。

```shell
$ calicoctl patch felixconfiguration default --type='merge' -p '{"spec": {"wireguardHostEncryptionEnabled": true}}'
```

### AKS

AKS集群节点用的是Ubuntu，已经装好WireGuard了，无需手动安装。但是可以用下面的命令启用host-to-host加密。

```shell
$ calicoctl patch felixconfiguration default --type='merge' -p '{"spec": {"wireguardHostEncryptionEnabled": true}}'
```

### OpenShift

在OpenShift v4.8中安装WireGuard：

1. 安装条件：
    - [CoreOS Butane](https://coreos.github.io/butane/getting-started/)
    - [Openshift CLI](https://docs.openshift.com/container-platform/4.2/cli_reference/openshift_cli/getting-started-cli.html)
2. 下载并配置kmod需要的工具。

```shell
$ FAKEROOT=$(mktemp -d)
$ git clone https://github.com/tigera/kmods-via-containers
$ cd kmods-via-containers
$ make install FAKEROOT=${FAKEROOT}
$ cd ..
$ git clone https://github.com/tigera/kvc-wireguard-kmod
$ cd kvc-wireguard-kmod
$ make install FAKEROOT=${FAKEROOT}
$ cd ..
```

3. 配置/编辑`${FAKEROOT}/root/etc/kvc/wireguard-kmod.conf`。

a. 在`$FAKEROOT/etc/kvc/wireguard-kmod.conf`中必须为`KERNEL_CORE_RPM`、`KERNEL_DEVEL_RPM`、`KERNEL_MODULES_RPM`包指定URL地址。从[RedHat Access]()中获取`kernel-core`、`kernel-devel`、`kernel-modules`rpm的副本，并保存到一个http文件服务中，提供给你的OCP工程师。

b. 关于`kvc-wireguard-kmod/wireguard-kmod.conf`的详细配置，见[kvc-wireguard-kmod的README](https://github.com/tigera/kvc-wireguard-kmod#quick-config-variables-guide)。注意里面同时给出了WireGuard版本和内核版本的兼容性。

4. 

### 启用WireGuard

### 关闭特定节点的WireGuard

### 验证配置

### 关闭WireGuard
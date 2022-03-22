# Istio集成

将Calico的策略跟[Istio](https://istio.io/)集成，可以编写施加于应用层属性的策略，比如HTTP方法或路径，或者时针对加密的安全身份。现在我们就要搞一下集成并进行测试。

## 安装FlexVolume驱动

Calico使用FlexVolume驱动来开启Felix和Pod中的Dikastes容器之间的安全连接。它会挂载一个共享卷，Felix会在里面写入一个Unix Domain Socket。

在每个节点上执行下面的命令安装FlexVolume驱动的二进制。

```shell
sudo mkdir -p /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
sudo docker run --rm \
  -v /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds:/host/driver \
  calico/pod2daemon-flexvol:v3.20.0
```

确认`uds`二进制有了

```shell
ls -lh /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
```

结果

```
total 5.2M
-r-xr-x--- 1 root root 5.2M Jul 25 22:31 uds
```

## 安装Istio

[参照这里的说明](../../../04%E5%AE%89%E5%85%A8/07Istio的策略/01为Istio施加网络策略.md)开启应用层策略，更新Istio的sidecar injector，将Calico授权服务添加到Istio网格中。

## 给default命名空间添加Istio的命名空间标签

应用层策略只会施加给带有Envoy和Dikastes边车（sidecar）的Pod。没有这些sidecar的Pod只能施加标准的Calico网络策略。

可以根据命名空间来控制这种策略。要在某个命名空间中开启Istio和应用层策略，需要添加标签`istio-injection=enabled`。

给default命名空间打标签，这里我们要用它。

```shell
kubectl label namespace default istio-injection=enabled
```

## 测试应用层策略

可以参照[应用层策略教程](../../../04%E5%AE%89%E5%85%A8/07Istio的策略/03根据Istio的教程来施加网络策略.md)进行测试。
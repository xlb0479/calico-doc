# 从etcd迁移到Kubernetes数据存储

## 大面儿

把你的Calico数据存储从etcdv3在线换成Kubernetes。

## 价值

使用Kubernetes作为数据存储要比直接使用etcdv3有更多优势，包括组件数量更少，以及更好的RBAC控制。对于大部分用户来说，使用Kubernets数据存储的体验也更好。我们提供了无缝地从etcdv3切换成Kubernetes数据存储的方法。使用Kubernetes数据存储的完整的优势列表，见[Calico数据存储](../02%E5%AE%89%E8%A3%85Calico/01Kubernetes/13%E8%87%AA%E5%B7%B1%E5%8A%A8%E6%89%8B%E5%AE%89%E8%A3%85Calico/03Calico%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8.md#使用Kubernetes作为数据存储)。

## 开始之前

- 确保你的Calico用的是etcdv3。本文不适用于使用Kubernetes API数据存储的安装环境。
- 使用**最新版本的calicoctl**，[安装并配置上etcd](../02%E5%AE%89%E8%A3%85Calico/00%E5%AE%89%E8%A3%85Calico.md)。

> 注意：因为下面的操作会修改calicoctl的配置，我们不建议将calicoctl安装成Kubernetes的Pod。应该直接在主机上安装二进制并能够连接etcd和Kubernetes API。

## 怎么弄

### 迁移数据存储

我们用`calicoctl datastore migrate`及其子命令来迁移数据存储。详见[calicoctl数据存储迁移](../06%E5%8F%82%E8%80%83/03calicoctl/12%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8/01%E7%AE%80%E4%BB%8B.md)文档。

1. 锁定要迁移的etcd数据存储。这样可以阻止数据变更导致集群异常。

```shell
calicoctl datastore migrate lock
```

> 注意：执行完上面的命令后就不可以对集群配置再做修改了，直到迁移完成。新的Pod不会启动，直到迁移完成。

2. 将数据存储中的数据导出到文件中。

```shell
calicoctl datastore migrate export > etcd-data
```

3. 将`calicoctl`配置成连接[Kubernetes数据存储](02calicoctl/02%E9%85%8D%E7%BD%AEcalicoctl/03%E9%85%8D%E7%BD%AEcalicoctl%E8%BF%9E%E6%8E%A5%E5%88%B0Kubernetes%20API%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8.md)。

4. 导入数据。

```shell
calicoctl datastore migrate import -f etcd-data
```

5. 确认数据存储导入正常。可以通过`calicoctl`查询原有etcd数据存储中的任意Calico资源进行确认（比如网络策略）。

```shell
calicoctl get networkpolicy
```

6. 将Calico配置成使用Kubernetes数据存储。参照基于Kubernetes数据存储安装Calico的文档。安装指引中包含了`calico.yaml`相关的版本。

```shell
kubectl apply -f calico.yaml
```

7. 等待Calico完成滚动更新，执行下面的命令跟踪更新过程

```shell
kubectl rollout status daemonset calico-node -n kube-system
```

8. 解锁数据存储。Calico资源可以恢复对集群的影响了。

```shell
calicoctl datastore migrate unlock
```

> 注意：当Kubernetes数据存储解锁后，数据存储迁移过程无法回滚。确保Kubernetes数据存储中已经得到了所有期望的Calico资源，然后再解锁。
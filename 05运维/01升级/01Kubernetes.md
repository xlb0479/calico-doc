# Calico版本升级

## 关于

这里我们讲一下如何从Calico v3.0及以后升级到v3.22。整个过程根据数据存储的类型和安装方式的不同而有所差异。

如果你的Calico使用的是etcd模式，我们建议按照[这里](../04%E4%BB%8Eetcd%E8%BF%81%E7%A7%BB%E5%88%B0Kubernetes%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8.md)说的升级到Kubernetes API数据存储。

如果你是用`calico.yaml`manifest安装的Calico，我们建议按照[这里](../05%E5%B0%86Calico%E8%BF%81%E7%A7%BB%E5%88%B0%E5%9F%BA%E4%BA%8Eoperator%E7%9A%84%E5%AE%89%E8%A3%85%E6%96%B9%E5%BC%8F.md)说的升级到operator模式。

- [Helm安装升级](#Helm安装升级)
- [operator安装升级](#operator安装升级)
- [基于manifest安装且使用Kubernetes API数据存储的升级](#基于manifest安装且使用Kubernetes%20API数据存储的升级)
- [直连etcd数据存储的升级](#直连etcd数据存储的升级)

> 重要：升级后不要使用旧版本的`calicoctl`。可能会导致行为和数据异常。

## 主机端点

> 重要：如果集群的主机端点配置了`interfaceName: *`那么升级前必须要让集群有所准备。否则可能会导致集群不可用。

在Calico v3.14之前，全接口主机端点（主机端点配置了`interfaceName: *`）只支持pre-DNAT策略。全接口主机端点在没有任何策略的情况下默认会放行所有流量。

从v3.14开始，全接口主机端点也可以支持普通的策略了。其中包括了一处对全接口主机端点默认行为的调整：如果没有策略，则默认**拒绝流量**。这种默认机制就跟“有命名”主机端点（指定了接口名，例如“eth0”）一致了；有命名的主机端点没有策略的话就是会拒绝流量。

升级到v3.22之前，你必须确保当前有已生效的全局网络策略，选中已有的全接口主机端点并且明确放行了已有流量。因此你可以创建一个放行所有流量的策略，选中已有的全接口主机端点。首先，我们要给已有的主机端点加个标签。查看所有全接口主机端点：

```shell
$ calicoctl get hep -owide | grep '*' | awk '{print $1}'
```

得到主机端点的名字后，我们就可以打标签了（例如**host-endpoint-upgrade: " "**）：

```shell
$ calicoctl get hep -owide | grep '*' | awk '{print $1}' \
  | xargs -I {} kubectl exec -i -n kube-system calicoctl -- /calicoctl label hostendpoint {} host-endpoint-upgrade=
```

既然现在所有的全接口主机端点都打上了**host-endpoint-upgrade**标签，我们可以创建一个临时策略，记录并放行这些主机端点上的所有出入流量。

```yaml
$ cat > allow-all-upgrade.yaml <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-all-upgrade
spec:
  selector: has(host-endpoint-upgrade)
  types:
  - Ingress
  - Egress
  ingress:
  - action: Log
  - action: Allow
  egress:
  - action: Log
  - action: Allow
EOF
```

执行策略：

```shell
$ calicoctl apply -f - < allow-all-upgrade.yaml
```

执行该策略后，全接口主机端点就会记录并放行它上面的所有流量。这个策略放行的是其它策略没有拦截的所有流量。升级完成后，看看syslog日志，看看主机端点上的出入流量，根据需要对策略进行调整，加固主机端点上的流量。

## Helm安装升级

1. 执行helm升级：

```shell
$ helm upgrade calico projectcalico/tigera-operator
```
## operator安装升级

1. 下载v3.22 operator的manifest。

```shell
$ curl https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml -O
```

2. 升级。

```shell
$ kubectl apply -f tigera-operator.yaml
```

## 基于manifest安装且使用Kubernetes API数据存储的升级

1. 下载v3.22的manifest。

### Calico策略及网络

```shell
$ curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
```

### Calico策略及flannel网络

```shell
$ curl https://projectcalico.docs.tigera.io/manifests/canal.yaml -O
```

### Calico策略（高级）

```shell
$ curl https://projectcalico.docs.tigera.io/manifests/calico-policy-only.yaml -O
```

> 注意：如果你以前手动改过manifest，那么这次也要按照原来的改一下。

2. 开始滚动更新，把`<manifest-file-name>`换成v3.22的manifest。

```shell
kubectl apply -f <manifest-file-name>
```

3. 监控升级过程。

```shell
watch kubectl get pods -n kube-system
```

确认Calico Pod的状态都是`Running`了。

```shel
calico-node-hvvg8     2/2   Running   0    3m
calico-node-vm8kh     2/2   Running   0    3m
calico-node-w92wk     2/2   Running   0    3m
```

4. 删除已有的`calicoctl`，[安装新版`calicoctl`](../02calicoctl/01%E5%AE%89%E8%A3%85calicoctl.md)并[连接到数据存储](../02calicoctl/02%E9%85%8D%E7%BD%AEcalicoctl/01%E7%AE%80%E4%BB%8B.md)。
5. 检查Calico版本号。

```shell
$ calicoctl version
```

返回的`Cluster Version`应该是`v3.22.x`。

6. 如果你[开启了应用层策略](../../04%E5%AE%89%E5%85%A8/07Istio%E7%9A%84%E7%AD%96%E7%95%A5/01%E4%B8%BAIstio%E6%96%BD%E5%8A%A0%E7%BD%91%E7%BB%9C%E7%AD%96%E7%95%A5.md)，请参考[下面的说明](#如果开了应用层策略)完成升级过程。如果没用Istio的话跳过即可。
7. 如果你是从v3.14之前的版本升级，并且按照上面说的执行了主机端点相关的前置操作，那么请检查一下临时策略中的流量日志，按需增加全局网络策略，然后删除临时网络策略**allow-all-upgrade**。
8. 恭喜！现在已经升级到了v3.22。

## 直连etcd数据存储的升级

1. 下载v3.22的manifest。

### Calico策略及网络

```shell
$ curl https://projectcalico.docs.tigera.io/manifests/calico-etcd.yaml -O
```

### Calico策略及flannel网络

```shell
$ curl https://projectcalico.docs.tigera.io/manifests/canal-etcd.yaml -O
```

> 注意：这里需要你完成对manifest的手动修改。至少要改改`etcd_endpoints`吧。

2. 开始滚动更新，将`<manifest-file-name>`换成v3.22的manifest文件名。

```shell
kubectl apply -f <manifest-file-name>
```

3. 监控升级过程。

```shell
watch kubectl get pods -n kube-system
```

确认Calico Pod的状态都是`Running`了。

```shel
calico-kube-controllers-6d4b9d6b5b-wlkfj   1/1       Running   0          3m
calico-node-hvvg8                          1/2       Running   0          3m
calico-node-vm8kh                          1/2       Running   0          3m
calico-node-w92wk                          1/2       Running   0          3m
```

> 提示：calico-node的`READY`列此时都应该是`1/2`。

4. 删除已有的`calicoctl`，[安装新版`calicoctl`](../02calicoctl/01%E5%AE%89%E8%A3%85calicoctl.md)并[连接到数据存储](../02calicoctl/02%E9%85%8D%E7%BD%AEcalicoctl/01%E7%AE%80%E4%BB%8B.md)。
5. 检查Calico版本号。

```shell
$ calicoctl version
```

返回的`Cluster Version`应该是`v3.22.x`。

6. 如果你[开启了应用层策略](../../04%E5%AE%89%E5%85%A8/07Istio%E7%9A%84%E7%AD%96%E7%95%A5/01%E4%B8%BAIstio%E6%96%BD%E5%8A%A0%E7%BD%91%E7%BB%9C%E7%AD%96%E7%95%A5.md)，请参考[下面的说明](#如果开了应用层策略)完成升级过程。如果没用Istio的话跳过即可。
7. 如果你是从v3.14之前的版本升级，并且按照上面说的执行了主机端点相关的前置操作，那么请检查一下临时策略中的流量日志，按需增加全局网络策略，然后删除临时网络策略**allow-all-upgrade**。
8. 恭喜！现在已经升级到了v3.22。

## 如果开了应用层策略

Dikastes跟Calico的版本保持同步，但是升级后的`calico-node`也可以跟旧版本的Dikastes一起工作，这样在升级过程中可以保证数据面的连通性。当`calico-node`升级完成后，你可以重新部署你的业务Pod并使用新版本的Dikastes。

如果你[开启了应用层策略](../../04%E5%AE%89%E5%85%A8/07Istio%E7%9A%84%E7%AD%96%E7%95%A5/01%E4%B8%BAIstio%E6%96%BD%E5%8A%A0%E7%BD%91%E7%BB%9C%E7%AD%96%E7%95%A5.md)，执行下面的操作对Diakstes进行升级。如果没用Istio则直接跳过即可。

1. 修改Istio的sidecar注入模板，使用新版本的Dikastes。将`<your Istio version>`替换成你的Istio版本号，例如`1.4.2`。

```shell
$ kubectl apply -f https://projectcalico.docs.tigera.io/manifests/alp/istio-inject-configmap-%3Cyour%20Istio%20version%3E.yaml
```

2. 新模板生效后，新创建的Pod就会使用升级后的Diakstes。给你所有的服务执行一次滚动更新，让它们都更新一下Dikastes的版本。

## 迁移到自动主机节点

> 重要：自动主机节点在没有网络策略的情况下会放行所有流量。可能会导致异常的行为和数据。

将已有的全接口主机端点迁移到Calico管理的自动主机端点：

1. 将全接口主机端点上的标签添加到对应的Kubernetes节点上。Calico会管理自动主机端点上的标签，从节点上同步。全接口主机端点上的所有标签都应该添加到对应的节点上。比如**node1**节点上的全接口主机端点带有标签**environment: dev**，那么应该给节点打上同样的标签：

```shell
$ kubectl label node node1 environment=dev
```

2. 根据[启用自动主机端点](../../04%E5%AE%89%E5%85%A8/05%E4%B8%BB%E6%9C%BA%E7%AD%96%E7%95%A5/02%E4%BF%9D%E6%8A%A4Kubernetes%E8%8A%82%E7%82%B9.md)的指引开启自动主机端点。注意自动主机端点没有网络策略的话默认会放行所有的流量。

```shell
$ calicoctl patch kubecontrollersconfiguration default --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```

3. 删除所有旧的全接口主机端点。有多种方式可以区分出Calico管理的主机端点。首先，自动主机端点带有**projectcalico.org/created-by: calico-kube-controllers**标签。其次，自动主机端点的名字都带有**-auto-hep**后缀。

```shell
$ calicoctl delete hostendpoint <old_hostendpoint_name>
```
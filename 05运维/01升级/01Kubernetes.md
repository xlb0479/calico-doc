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

## operator安装升级

## 基于manifest安装且使用Kubernetes API数据存储的升级

## 直连etcd数据存储的升级
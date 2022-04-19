# Calico策略教程

Calico的额网络策略**扩展**了Kubernetes网络策略的功能。为了显摆这个事儿，本文使用了一种跟[K8S策略，高级教程](../02K8S%E7%AD%96%E7%95%A5/04K8S%E7%AD%96%E7%95%A5%EF%BC%8C%E9%AB%98%E7%BA%A7%E6%95%99%E7%A8%8B.md)中类似的玩法，但用的是Calico的网络策略，而且把不同之处做了高亮显示，使用了Kubernetes网络策略中没有的特性。

## 要求

- 一个能用的Kubernetes集群，并且能用kubectl和calicoctl进行控制
- Kubernetes节点能上网
- 你得熟悉[Calico网络策略](01Calico网络策略入门.md)

## 教程大纲

1. 创建命名空间和NGINX服务
2. 配置默认拒绝
3. 允许busybox的出口流量
4. 允许NGINX的入口流量
5. 清理

## 1. 创建命名空间和NGINX服务

现在我们要用一个新的命名空间。执行下面的命令创建命名空间和一个简单的NGINX服务，监听80口。

```shell
$ kubectl create ns advanced-policy-demo
$ kubectl create deployment --namespace=advanced-policy-demo nginx --image=nginx
$ kubectl expose --namespace=advanced-policy-demo deployment nginx --port=80
```

### 网络验证——允许所有的ingress和egress

打开一个新的shell会话，使用`kubectl`连接Kubernetes集群创建一个busybox进行策略验证。这个工具Pod在本文中将会一直用于策略验证。

```shell
$ kubectl run --namespace=advanced-policy-demo access --rm -ti --image busybox /bin/sh
```

上面的命令在`busybox`中打开了一个新的shell会话，如下。

```shell
Waiting for pod advanced-policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.
/ #
```

然后在busybox中执行下面的命令测试一下nginx服务。

```shell
$ wget -q --timeout=5 nginx -O -
```

返回了nginx欢迎页的HTML。

继续，执行下面的命令访问一下google.com。

```shell
$ wget -q --timeout=5 google.com -O -
```

返回了google.com主页的HTML。

## 2. 干掉所有流量

我们起手先来一个默认拒绝的[全局Calico网络策略](../../../06%E5%8F%82%E8%80%83/04%E8%B5%84%E6%BA%90%E5%AE%9A%E4%B9%89/06%E5%85%A8%E5%B1%80%E7%BD%91%E7%BB%9C%E7%AD%96%E7%95%A5.md)（只能用Calico实现），这样有助于你践行[零信任网络模型](../../01%E9%9B%B6%E4%BF%A1%E4%BB%BB%E7%BD%91%E7%BB%9C%E6%A8%A1%E5%9E%8B.md)的最佳实践。注意，全局Calico网络策略不带命名空间，根据策略选择器会影响到所有的Pod。相反，Kubernetes的网络策略是带命名空间的，所以如果要达到相同的效果，那就得给每个命名空间都配一个默认拒绝的策略。还有就是为了让这份教程简单一些，我们忽略了kube-system命名空间中的Pod，这样我们在搞默认拒绝的时候就不用关心那些Kubernetes运行必备的策略调整。

```yaml
$ calicoctl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny
spec:
  selector: projectcalico.org/namespace != "kube-system"
  types:
  - Ingress
  - Egress
EOF
```

### 网络验证——拒绝所有ingress和egress

我们的Pod不在kube-system中，上面我们创建了针对性的策略，没有被明确放行的流量都会被这个策略拒掉。

我们可以回到我们的工具Pod中，验证一下对nginx服务的访问。 

```shell
$ wget -q --timeout=5 nginx -O -
```

返回：

```shell
wget: bad address 'nginx'
```

访问一下google.com。

```shell
$ wget -q --timeout=5 google.com -O -
```

返回：

```shell
wget: bad address 'google.com'
```

## 3. 允许busybox的出口流量


现在创建一个Calico网络策略，放行工具Pod的出口流量。如果是生产环境的工作负载，你可能需要把这个egress规则限制的再死一点，应该只允许它访问自身必需的服务。这里我们只是做下验证，所以就放行了所有的egress流量。

```yaml
$ calicoctl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-busybox-egress
  namespace: advanced-policy-demo
spec:
  selector: run == 'access'
  types:
  - Egress
  egress:
  - action: Allow
EOF
```

### 网络验证——工具Pod可以访问google但是不能访问nginx

既然我们放行了所有egress流量，现在应该就可以访问google了。

尝试获取google.com的主页。

```shell
$ wget -q --timeout=5 google.com -O -
```

返回了google主页的HTML。

如果我们访问nginx，从工具Pod中出来的egress流量是被放行了，但是连接依然会被拒绝，因为我们的默认拒绝策略阻止了nginx的ingress流量。

执行下面的命令确认一下。

```shell
$ wget -q --timeout=5 nginx -O -
```

返回：

```shell
wget: download timed out
```

可以访问google是因为我们放行了工具Pod的所有出口egress流量，无法访问nginx服务是因为对应的服务没有放行ingress流量。现在我们就搞一下。
## 4. 允许NGINX的入口流量

创建一个Calico网络策略，放行从工具Pod到nginx服务的ingress流量。

```yaml
$ calicoctl create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-nginx-ingress
  namespace: advanced-policy-demo
spec:
  selector: app == 'nginx'
  types:
  - Ingress
  ingress:
  - action: Allow
    source:
      selector: run == 'access'
EOF
```

### 网络验证——放行从工具Pod到nginx服务的ingress流量

执行下面的命令看看能不能访问nginx服务。

```shell
$ wget -q --timeout=5 nginx -O -
```

返回了nginx欢迎页的HTML。

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>...
```

再试试google.com。

```shell
$ wget -q --timeout=5 google.com -O -
```

返回了google主页的HTML。

现在我们通过Calico网络策略放行了工具Pod对互联网和nginx服务的访问！

## 5. 清理

执行下面的命令清理一下本文创建的各种东西。

```shell
$ calicoctl delete policy allow-busybox-egress -n advanced-policy-demo
$ calicoctl delete policy allow-nginx-ingress -n advanced-policy-demo
$ calicoctl delete gnp default-deny
$ kubectl delete ns advanced-policy-demo
```
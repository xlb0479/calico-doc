# 加固BGP会话

## 大面儿

使用BGP密码，防止攻击者注入错误的路由信息。

## 价值

给BGP对等设置一个密码，对于外部的BGP speaker来说，只有当它能提供同样的密码时才能建立对等。这样就针对伪装speaker的攻击者增加了一层防御，别想给我的集群搞事情。

## 特性

这里我们用到了Calico的以下特性：

- **BGPPeer**及其password字段

## 概念

### BGP会话的密码保护

密码保护属于是BGP会话的一个[标准化](https://datatracker.ietf.org/doc/html/rfc5925)的可选功能。效果就是只有当BGP会话的两端都配置了同样的密码后，才能进行通信并交换路由信息。

需要知道的是设置了密码并不会对数据交换进行*加密*。仍然能够非常容易地*窃听*到这些交换的数据，但是无法*注入*错误的信息。

### 使用Kubernetes Secret保存密码

在Kubernetes中，Secret资源是用来保存敏感信息的，包括密码。因此，对于这里用到的Calico特性，我们可以使用Secret来保存BGP的密码。

## 怎么弄

如果要在BGP对等中使用密码：

1. 在calico-node所在的命名空间中创建（或更新）一个Kubernetes Secret，这样就会得到一个key，它的值就是我们的密码。注意Secret的名字和key的名字。

> 注意：BGP密码最多80个字符。如果超长则无法建立BGP会话。

2. 确保calico-node的RBAC权限能够访问到这个Secret。
3. 在BGPPeer资源中指定Secret和key的名字。

### 创建或更新Kubernetes Secret

例如：

```yaml
kubectl create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: bgp-secrets
  namespace: kube-system
type: Opaque
stringData:
  rr-password: very-secret
EOF
```

如果你的calico-node并不在kube-system命名空间中，那么就应该在对应的命名空间中创建，而不是这里写的kube-system。

下面是一个使用该密码的BGPPeer资源，注意Secret的名字`bgp-secrets`和key的名字`rr-password`。

### 确认RBAC权限

calico-node必须有访问这个Secret的权限。如果要做授权，你需要：

```yaml
kubectl create -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-access
  namespace: <namespace>
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["bgp-secrets"]
  verbs: ["watch", "list", "get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-access
  namespace: <namespace>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-access
subjects:
- kind: ServiceAccount
  name: calico-node
  namespace: <namespace>
EOF
```

### 在BGPPeer资源中指定Secret和key的名字

然后，在[配置BGP对等](../../03%E7%BD%91%E7%BB%9C/02%E9%85%8D%E7%BD%AE%E7%BD%91%E7%BB%9C/01%E9%85%8D%E7%BD%AEBGP%E5%AF%B9%E7%AD%89.md)时，加上Secret和key的名字，例如：

```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-3040
spec:
  peerIP: 192.20.30.40
  asNumber: 64567
  password:
    secretKeyRef:
      name: bgp-secrets
      key: rr-password
```
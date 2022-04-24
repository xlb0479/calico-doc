# 在策略中使用ServiceAccount

## 大面儿

使用Calico网络策略为Kubernetes的ServiceAccount放行/拒绝流量。

## 价值

使用Calico网络策略，你可以利用Kubernetes的ServiceAccount和RBAC，对策略进行灵活管控。比如，安全团队可以通过RBAC：

- 控制开发团队在命名空间中可以使用哪些ServiceAccount
- 为ServiceAccount编写高优先级网络策略（开发团队无权覆盖）

网络安全团队可以维护所有的安全控制，也可以选择性的将操作下放给开发者。

如果应用**开启了Istio**和Calico网络策略，则会检查ServiceAccount对应的加密身份（以及网络身份），实现两步验证。

## 特性

本文使用Calico的以下特性：

**NetworkPolicy**或**GlobalNetworkPolicy**，及其ServiceAccount规则和匹配方式。

## 概念

### 使用权限的最小集

ServiceAccount的操作是由RBAC控制的，所以你可以只为授信实体（代码和/或人）授权创建、修改或删除ServiceAccount。在工作负载中执行的任何操作，客户端都需要Kubernetes API服务器的认证。

如果没有为Pod显式指定一个ServiceAccount，那么就使用命名空间中的默认ServiceAccount。

不要给命名空间中默认的ServiceAccount授予太大的权限。如果一个应用需要访问Kubernetes API，为它创建单独的ServiceAccount以及最小化的权限集合。

### ServiceAccount标签

类似于其它的Kubernetes对象，ServiceAccount也有标签。可以用标签为ServiceAccount创建“分组”。Calico网络策略可以用以下方式根据ServiceAccount匹配工作负载：

- 精确匹配ServiceAccount的名字
- ServiceAccount的标签选择器表达式

## 开始之前……

为应用配置一个唯一的Kubernetes ServiceAccount。

## 怎么弄

- [用ServiceAccount名限制工作负载的ingress流量](#用ServiceAccount名限制工作负载的ingress流量)
- [用ServiceAccount标签限制工作负载的ingress流量](#用ServiceAccount标签限制工作负载的ingress流量)
- [用Kubernetes RBAC控制ServiceAccount标签分配](#用Kubernetes%20RBAC控制ServiceAccount标签分配)

### 用ServiceAccount名限制工作负载的ingress流量

下面的例子中，如果ingress流量的来源ServiceAccount名为**api-service**或**user-auth-service**，那么就放行。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: demo-calico
  namespace: prod-engineering
spec:
  ingress:
    - action: Allow
      source:
        serviceAccounts:
          names:
            - api-service
            - user-auth-service
  selector: 'app == "db"'
```

### 用ServiceAccount标签限制工作负载的ingress流量

下面的例子中，如果ingress流量的来源ServiceAccount匹配标签选择器**app == web-frontend**，那么就放行。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-web-frontend
  namespace: prod-engineering
spec:
  ingress:
    - action: Allow
      source:
        serviceAccounts:
          selector: 'app == "web-frontend"'
  selector: 'app == "db"'
```

### 用Kubernetes RBAC控制ServiceAccount标签分配

网络策略可以根据端点、命名空间、ServiceAccount的标签进行选择。如果时根据ServiceAccount，你可以用Kubernetes RBAC控制谁可以给ServiceAccount打标签。这样就可以将部署Pod和给ServiceAccount打标签的人分开。

下面的例子中，具有内部ServiceAccount的Pod只可以跟带有`role: intern`标签的ServiceAccount进行通信。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: restrict-intern-access
  namespace: prod-engineering
spec:
  serviceAccountSelector: 'role == "intern"'
  ingress:
    - action: Allow
      source:
        serviceAccounts:
          selector: 'role == "intern"'
  egress:
    - action: Allow
      destination:
        serviceAccounts:
          selector: 'role == "intern"'
```
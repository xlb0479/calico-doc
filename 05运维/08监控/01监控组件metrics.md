# 监控Calico组件metrics

## 大面儿

给Calico组件配上Prometheus，抓取Calico的健康指标。

## 价值

使用开源的Prometheus做监控告警，可以通过Promehteus或Grafana查看Calico组件中的时序性metrics。

## 价值

这里用到了Calico的以下特性：

**Felix**，**Typha**以及**kube-controllers**组件，配上Prometheus以及相关参数（好让Prometheus进行处理）。

## 概念

### 关于Prometheus

Prometheus会根据任务指示去抓取metrics，然后通过可视化工具（比如Grafana）展示时序数据。对于Calico，这里的“任务”就是从Felix和Typha组件中获取metrics。

### 关于Calico Felix、Typha、kube-controllers组件

**Felix**是每个机器上的一个守护进程，它实现了网络策略。Felix是Calico的大脑。Typha则是一组可选的Pod，增强Felix在Calico节点和数据存储之间的流量扩展性。Kube-controllers则是一组控制器，负责各种控制面工作，比如通过Kubernetes API进行资源回收和同步。

你可以配置Felix、Typha、和/或kube-controllers来为Prometheus提供metrics。

## 开始之前……

我们假设你已经学习了所有其他的章节，并且已经有了一个装了Calico、calicoctl、kubectl的Kubernetes集群，

## 怎么弄

本文教你如何实现基本的基于Prometheus的Calico监控。

1. 配置Calico并开启metrics上报。
2. 创建Prometheus需要的命名空间和ServiceAccount。
3. 部署并配置Prometheus。
4. 在Prometheus界面上查看metrics并创建简易图表。

> 注意：这里我们假设Calico安装在了kube-system命名空间中。如果用了Operator，你需要把kube-system中的相关定义放到calico-system命名空间中。

### 1. 配置Calico并开启metrics上报

#### Felix配置

Felix的Prometheus metrics默认是**关闭**的。需要用calicoctl手动修改Felix配置（**prometheusMetricsEnabled**）。

> 注意：[这里](../../06%E5%8F%82%E8%80%83/07Felix/01%E9%85%8D%E7%BD%AE.md)有一份详细的配置参数。

```shell
$ calicoctl patch felixConfiguration default  --patch '{"spec":{"prometheusMetricsEnabled": true}}'
```

得到如下输出：

```text
Successfully patched 1 'FelixConfiguration' resource
```

#### 创建一个Service来暴露Felix的metrics

Prometheus用Kubernetes的Service来做动态发现。这里你需要创建一个名为`felix-metrics-svc`的Service，Prometheus会用它来发现所有的Felix metrics端点。

> 注意：Felix默认用TCP的9091端口来发布它的metrics。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: felix-metrics-svc
  namespace: kube-system
spec:
  selector:
    k8s-app: calico-node
  ports:
  - port: 9091
    targetPort: 9091
EOF
```

#### Typha配置

> 注意：Typha是可选的，如果你没装Typha，可以跳过这一节。

如果你不确定你装没装`Typha`，可以执行下面的操作：

```shell
$ kubectl get pods -A | grep typha
```

如果得到的结果跟下面的示例差不多那你就是装了。

> 注意：下面展示的Pod后缀是动态生成的。实际情况会有所差异。

```text
kube-system     calico-typha-56fccfcdc4-z27xj                         1/1     Running   0          28h
kube-system     calico-typha-horizontal-autoscaler-74f77cd87c-6hx27   1/1     Running   0          28h
```

有[两种方式](../../06%E5%8F%82%E8%80%83/08Typha/02%E9%85%8D%E7%BD%AE.md)让Prometheus抓Typha。

#### 创建一个Service来暴露Typha的metrics

> 注意：Typha模式使用TCP的**9091端口**来发布它的metrics。但如果是用[Amazon yaml](https://github.com/aws/amazon-vpc-cni-k8s/blob/b001dc6a8fff52926ed9a93ee6c4104f02d365ab/config/v1.5/calico.yaml#L535-L536)装的Calico，这个端口则是9093，因为它是通过**TYPHA_PROMETHEUSMETRICSPORT**这个环境变量来设定的。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: typha-metrics-svc
  namespace: kube-system
spec:
  selector:
    k8s-app: calico-typha
  ports:
  - port: 9093
    targetPort: 9093
EOF
```

#### kube-controllers配置

它的metrics默认是**启用**的，在TCP的9094端口。可以修改KubeControllersConfiguration资源来调整这个端口。

```shell
$ calicoctl patch kubecontrollersconfiguration default  --patch '{"spec":{"prometheusMetricsPort": 9095}}'
```

改成零的话就相当于禁用了。

#### 创建一个Service来暴露kube-controllers的metrics

Prometheus使用Kubernetes的Service来做动态发现。这里我们创建一个名为`kube-controllers-metrics-svc`的Service，Prometheus用它来发现所有的metrics端点。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: kube-controllers-metrics-svc
  namespace: kube-system
spec:
  selector:
    k8s-app: calico-kube-controllers
  ports:
  - port: 9094
    targetPort: 9094
EOF
```

### 2. 准备集群

#### 命名空间

`Namespace`是用来隔离资源的。这里你需要创建一个名为`calico-monitoring`的命名空间来放置监控相关的资源。

> 注意：Kubernetes的命名空间相关知识可以在[这里](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/)找到。

```yaml
$ kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: calico-monitoring
  labels:
    app:  ns-calico-monitoring
    role: monitoring
EOF
```

#### ServiceAccount

需要给Prometheus弄一个ServiceAccount，授予必备的权限来获取Calico的信息。

> 注意：关于RBAC的详细介绍可以在[这里](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)找到。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: calico-prometheus-user
rules:
- apiGroups: [""]
  resources:
  - endpoints
  - services
  - pods
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-prometheus-user
  namespace: calico-monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: calico-prometheus-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-prometheus-user
subjects:
- kind: ServiceAccount
  name: calico-prometheus-user
  namespace: calico-monitoring
EOF
```

### 3. 安装Prometheus

#### 创建配置文件

我们可以用ConfigMap来保存Prometheus的配置。

> 注意：关于配置文件的详细介绍可以在[这里](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)找到。

##### Operator

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: calico-monitoring
data:
  prometheus.yml: |-
    global:
      scrape_interval:   15s
      external_labels:
        monitor: 'tutorial-monitor'
    scrape_configs:
    - job_name: 'prometheus'
      scrape_interval: 5s
      static_configs:
      - targets: ['localhost:9090']
    - job_name: 'felix_metrics'
      scrape_interval: 5s
      scheme: http
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        regex: felix-metrics-svc
        replacement: $1
        action: keep
    - job_name: 'typha_metrics'
      scrape_interval: 5s
      scheme: http
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        regex: typha-metrics-svc
        replacement: $1
        action: keep
    - job_name: 'kube_controllers_metrics'
      scrape_interval: 5s
      scheme: http
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        regex: kube-controllers-metrics-svc
        replacement: $1
        action: keep
EOF
```

##### Manifest

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: calico-monitoring
data:
  prometheus.yml: |-
    global:
      scrape_interval:   15s
      external_labels:
        monitor: 'tutorial-monitor'
    scrape_configs:
    - job_name: 'prometheus'
      scrape_interval: 5s
      static_configs:
      - targets: ['localhost:9090']
    - job_name: 'felix_metrics'
      scrape_interval: 5s
      scheme: http
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        regex: felix-metrics-svc
        replacement: $1
        action: keep
    - job_name: 'typha_metrics'
      scrape_interval: 5s
      scheme: http
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        regex: typha-metrics-svc
        replacement: $1
        action: keep
    - job_name: 'kube_controllers_metrics'
      scrape_interval: 5s
      scheme: http
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        regex: kube-controllers-metrics-svc
        replacement: $1
        action: keep
EOF
```

#### 创建Prometheus

现在你已经有了一个具备足够权限的`serviceaccount`，以及一份有效的配置文件，可以开始创建Prometheus了。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: prometheus-pod
  namespace: calico-monitoring
  labels:
    app: prometheus-pod
    role: monitoring
spec:
  serviceAccountName: calico-prometheus-user
  containers:
  - name: prometheus-pod
    image: prom/prometheus
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    volumeMounts:
    - name: config-volume
      mountPath: /etc/prometheus/prometheus.yml
      subPath: prometheus.yml
    ports:
    - containerPort: 9090
  volumes:
  - name: config-volume
    configMap:
      name: prometheus-config
EOF
```

看看Pod建成功没，并且得是`Running`状态。

```shell
$ kubectl get pods prometheus-pod -n calico-monitoring
```

返回内容如下。

```text
NAME             READY   STATUS    RESTARTS   AGE
prometheus-pod   1/1     Running   0          16s
```

### 4. 查看metrics

现在可以通过端口转发来访问Prometheus的面板。

```shell
$ kubectl port-forward pod/prometheus-pod 9090:9090 -n calico-monitoring
```

浏览器中打开[http://localhost:9090](http://localhost:9090)，应该会看到Prometheus的界面。在Expression输入框中输入**felix_active_local_endpoints**然后带你及execute按钮。界面图表中应该会展示出你所有的节点以及其中的端点数量。

> 注意：关于metrics的详细介绍可以在[这里](../../06%E5%8F%82%E8%80%83/07Felix/02Prometheus%E6%8C%87%E6%A0%87.md)找到。

点击`Add Graph`，你会看到metrics被画成了一个图。

## 清理一下

执行下面的命令，删除本文中创建的所有资源和服务。

```shell
$ kubectl delete service felix-metrics-svc -n kube-system
$ kubectl delete service typha-metrics-svc -n kube-system
$ kubectl delete service kube-controllers-metrics-svc -n kube-system
$ kubectl delete namespace calico-monitoring
$ kubectl delete ClusterRole calico-prometheus-user
$ kubectl delete clusterrolebinding calico-prometheus-user
```

## 最佳实践

如果你开启了Calico的metrics，一种最佳实践是用网络策略限制一下对metrics端点的访问。详见[Calico Prometheus端点加固](../../04%E5%AE%89%E5%85%A8/10Calico%E7%BB%84%E4%BB%B6%E9%80%9A%E4%BF%A1%E5%8A%A0%E5%9B%BA/03%E5%8A%A0%E5%9B%BACalico%E7%9A%84Prometheus%E7%AB%AF%E7%82%B9.md)。

如果没用Prometheus，我们建议你禁用Prometheus相关的端口，更安全点。
# 可视化

## 大面儿

用Grafana查看Calico组件的metrics。

## 价值

Grafna可以帮助你进行metrics的可视化，让你快速发现异常点。下面的图片中展示了一些图标和metrics，帮你实现这些目标。

![img](https://projectcalico.docs.tigera.io/archive/v3.22/images/grafana-dashboard.png)

## 特性

这里用到了Calico的以下特性：

**Felix**和**Typha**组件，配置好Prometheus参数（让Prometheus可以过来抓取）和Grafana（做可视化）。

## 概念

### 关于Grafana

Grafana是一个开源的可视化及分析工具，允许你基于各种不同的数据源进行查询、可视化、告警以及探索metrics，包括保存在Prometheus中的Calico组件的metrics。

### 关于Prometheus

Prometheus是一个开源的监控工具，从配好的组件中抓取metrics并保存为时序数据，然后就可以通过Grafana进行可视化了。

## 开始之前……

这里我们假设你已经

- 有一个运行中的Kubernetes集群，装了Calico、calicoctl、kubectl
- 已经完成了所有[监控配置](01监控组件metrics.md)，装好了Prometheus并抓到了Calico组件的metrics。

## 怎么弄

本文给你讲一下如何在Grafana中创建Calico相关的看板。

### 准备Prometheus

你需要创建一个Service，让Prometheus对Grafana可见。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: prometheus-dashboard-svc
  namespace: calico-monitoring
spec:
  selector:
      app:  prometheus-pod
      role: monitoring
  ports:
  - port: 9090
    targetPort: 9090
EOF
```

### 准备Grafana

#### 1.提供数据源

Grafana的数据源就是指你的时序数据存在哪里。每种数据源都有一个特定的查询编辑器，对数据源自身的特性及功能进行了定制。

> 注意：关于Grafana数据源更详细的配置见[这里](https://grafana.com/docs/grafana/latest/datasources/)。

这里我们要用Grafana的供应机制来创建一个Prometheus数据源。

> 注意：关于供应机制的详细信息见[这里](https://grafana.com/docs/grafana/latest/administration/provisioning/)。

这里你要创建一个数据源并将它指向集群中Prometheus的Service。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-config
  namespace: calico-monitoring
data:
  prometheus.yaml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
               "access":"proxy",
                "editable": true,
                "name": "calico-demo-prometheus",
                "orgId": 1,
                "type": "prometheus",
                "url": "http://prometheus-dashboard-svc.calico-monitoring.svc:9090",
                "version": 1
            }
        ]
    }
EOF
```

#### 2.创建Calico看板

现在创建一个ConfigMap，包含了Felix和Typha的看板。

```shell
$ kubectl apply -f https://projectcalico.docs.tigera.io/archive/v3.22/manifests/grafana-dashboards.yaml
```

#### 3.创建Grafana

现在我们用之前创建好的配置文件来创建Grafana的Pod。

> 注意：Grafana默认使用3000端口。详见[这里](https://grafana.com/docs/grafana/latest/installation/configuration/#comments-in-ini-files)。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: grafana-pod
  namespace: calico-monitoring
  labels:
    app:  grafana-pod
    role: monitoring
spec:
  containers:
  - name: grafana-pod
    image: grafana/grafana:latest
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    volumeMounts:
    - name: grafana-config-volume
      mountPath: /etc/grafana/provisioning/datasources
    - name: grafana-dashboards-volume
      mountPath: /etc/grafana/provisioning/dashboards
    - name: grafana-storage-volume
      mountPath: /var/lib/grafana
    ports:
    - containerPort: 3000
  volumes:
  - name: grafana-storage-volume
    emptyDir: {}
  - name: grafana-config-volume
    configMap:
      name: grafana-config
  - name: grafana-dashboards-volume
    configMap:
      name: grafana-dashboards-config
EOF
```

#### 4.访问看板

现在你已经完成了所有必备操作。通过`port-forward`在你的机器上访问一下Grafana。

```shell
$ kubectl port-forward pod/grafana-pod 3000:3000 -n calico-monitoring
```

然后可以在浏览器中访问[http://localhost:3000](http://localhost:3000)打开Grafana，如果你想看Felix的看板可以直接点击[这里](http://localhost:3000/d/calico-felix-dashboard/felix-dashboard-calico?orgId=1)。

> 注意：用户名和密码都是`admin`。

登录后会提示你修改默认密码，你可以直接修改（`推荐`）并点击`Save`，或者点击`Skip`跳过一会儿再弄。

恭喜你，现在你看到了Felix的看板。

这里我们还给你准备了一个[Typha看板](http://localhost:3000/d/calico-typha-dashboard/typha-dashborad-calico?orgId=1)，如果你没用到Typha那么可以直接在页面中把它删掉。

> 注意：关于Typha的检测及安装详见[这里](01监控组件metrics.md#typha配置)。

## 清理

执行下面的命令，删除所有Calico的监控资源，包括本文创建的以及[监控组件metrics](01监控组件metrics.md)中创建的资源。

```shell
$ kubectl delete namespace calico-monitoring
```
# 自定义manifests

## 关于自定义manifests

我们提供了很多manifests，简化Calico的部署。在应用之前可以选择性的进行修改。或者你也可以在之后把想改的改一下重新应用也行。

各种修改对应的详细介绍。

- [定制Calico manifests](#定制Calico%20manifests)
- [定制应用层策略manifests](#定制应用层策略manifests)

## 定制Calico manifests

### 关于定制Calico manifests

每个manifest中都包含了所有安装Calico需要的资源。

它在Kubernetes中安装以下资源：

- 用DaemonSet在每个节点安装`calico/node`容器。
- 用DaemonSet在每个节点安装Calico CNI二进制和网络配置。
- 用Deployment安装`calico/kube-controllers`。
- `calico-etcd-secrets`Secret，可以提供etcd的TLS制品。
- `calico-config`ConfigMap，包含了用于配置安装过程的参数。

下面我们进一步讲一下这里面可配置的参数。

### 配置Pod的IP范围

Calico的IPAM从[IP池](../../../06%E5%8F%82%E8%80%83/04资源定义/09IP池.md)中分配IP地址。

要修改Pod的默认IP地址范围，就要改`calico.yaml`中的`CALICO_IPV4POOL_CIDR`。详见[配置calico/node](../../../06%E5%8F%82%E8%80%83/06calico-node.md)。

### 配置IP-in-IP

默认情况下，manifests中开启了跨子网时的IP-in-IP封装。许多用户可能想把这个IP-in-IP封装关了，例如下列场景。

- 集群[运行在一个配置好的AWS VPC中]()
- 所有Kubernetes节点都连到了同一个2层网络中。
- 打算用BGP对等，让底层设施了解Pod的IP地址。

要关闭IP-in-IP封装，改一下manifest中的`CALICO_IPV4POOL_IPIP`。详见[配置calico/node](../../../06%E5%8F%82%E8%80%83/06calico-node.md)。

### 把IP-in-IP改成VXLAN

默认Calico开启了IP-in-IP封装。如果你的网络中阻止了IP-in-IP，比如Azure，那么你可能就会想改成[Calico的VXLAN封装模式](../../../03%E7%BD%91%E7%BB%9C/02配置网络/02配置overlay网络.md)。要想在安装时做这种改动（这样Calico就会用VXLAN创建默认的IP池，不需要撤销已有的IP-in-IP配置）：

- 从[Calico策略和网络](#自定义manifests)的一个manifests开始搞。
- 把环境变量名`CALICO_IPV4POOL_IPIP`改为`CALICO_IPV4POOL_VXLAN`。改完之后它的值得是“Always”。
- 可选的，（如果是在一个只有VXLAN的集群中，为了节省一些资源）完全关闭Calico的基于BGP的网络：
    - 将`calico_backend: "bird"`改为`calico_backend: "vxlan"`。关闭BIRD。
    - 在calico/node的readiness/liveness检查中注释掉`- -bird-ready`和`- -bird-live`（否则关闭了BIRD会导致每个节点的readiness/liveness检查都会失败）：
    ```yaml
          livenessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-live
             # - -bird-live
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              # - -bird-ready
              - -felix-ready
    ```

关于calico/node的其它配置，包括其他的VXLAN配置，见[配置calico/node](../../../06%E5%8F%82%E8%80%83/06calico-node.md)。

> 注意：环境变量`CALICO_IPV4POOL_VXLAN`只有在第一个calico/node开始创建默认的IP池时才会起作用。如果池子已经建完了那这个环境变量也就不再起作用了。如果要在安装之后改成VXLAN模式，需要用calicoctl修改[IPPool](../../../06%E5%8F%82%E8%80%83/04资源定义/09IP池.md)资源。

### 配置etcd

默认情况下这些manifests不会配置到etcd的安全访问，并且假设每个节点上都有一个etcd代理。下面的选项允许你自定义etcd集群的endpoint，以及TLS。

下表中给出了用于支持etcd的`ConfigMap`选项：

|**选项**|**描述**|**默认值**
|-|-|-
|etcd_endpoints|逗号间隔的etcd endpoint列表|http://127.0.0.1:2379
|etcd_ca|这个文件中包含了签发etcd服务证书的CA的根证书。配置`calico/node`，CNI插件，以及Kubernetes控制器，让它们信任etcd服务提供的证书中的签名。|无
|etcd_key|这个文件中包含了`calico/node`的私钥，CNI插件和Kubernetes控制器的客户端证书。让这些组件加入到双向TLS认证，并且对etcd服务表明自己的身份。|无
|etcd_cert|这个文件中包含了签发给`calico/node`，CNI插件，以及Kubernetes控制器的客户端证书。让这些组件加入到双向TLS认证，并且对etcd服务表明自己的身份。|无

使用这些manifests的时候，如果etcd集群那边启用了TLS，那么必须按照下面的操作进行：

1. 根据你的安装方式下载对应的v3.22 manifest。

**Calico策略和网络**

```shell
curl https://projectcalico.docs.tigera.io/manifests/calico-etcd.yaml -O
```

**Calico策略和flannel网络**

```shell
curl https://projectcalico.docs.tigera.io/manifests/canal.yaml -O
```

2. 在`ConfigMap`中打开`etcd_ca`，`etcd_key`，`etcd_cert`的注释，然后如下。

```yaml
etcd_ca: "/calico-secrets/etcd-ca"
etcd_cert: "/calico-secrets/etcd-cert"
etcd_key: "/calico-secrets/etcd-key"
```

3. 确保上面要的三个文件你已经有了。

4. 用下面的命令转base64并且去掉换行。

```shell
cat <file> | base64 -w 0
```

5. 在名为`calico-etcd-secrets`的`Secret`中，打开`etcd_ca`，`etcd_key`，`etcd_cert`的注释，然后把base64值复制粘贴到对应的值中。

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # Populate the following files with etcd TLS configuration if desired, but leave blank if
  # not using TLS for etcd.
  # This self-hosted install expects three files with the following names.  The values
  # should be base64 encoded strings of the entire contents of each file.
  etcd-key: LS0tLS1CRUdJTiB...VZBVEUgS0VZLS0tLS0=
  etcd-cert: LS0tLS1...ElGSUNBVEUtLS0tLQ==
  etcd-ca: LS0tLS1CRUdJTiBD...JRklDQVRFLS0tLS0=
```

6. 应用manifest。

**Calico策略和网络**

```shell
kubectl apply -f calico.yaml
```

**Calico策略和flannel网络**

```shell
kubectl apply -f canal.yaml
```

### 授权选项

Calico的manifests会授予它的组件两个ServiceAccount中的一个。根据集群的授权模式，可能需要为这些ServiceAccount提供必要的权限。

### 其他配置

下表给出其他在`ConfigMap`中支持的选项。

|**选项**|**描述**|**默认值**
|-|-|-
|calico_backend|使用的backend|`bird`
|cni_network_config|每个节点上安装的CNI网络配置。支持下面讲的模板化功能。|

### CNI网络配置模板

`cni_network_config`配置选项支持下面的模板属性，它们会被`calico/cni`容器自动填入：

|**属性**|**替换为**
|-|-
|`__KUBERNETES_SERVICE_HOST__`|Kubernetes Service Cluster IP，例如`10.0.0.1`
|`__KUBERNETES_SERVICE_PORT__`|Kubernetes Service端口，例如`443`
|`__SERVICEACCOUNT_TOKEN__`|该命名空间的ServiceAccount token，如果有的话。
|`__ETCD_ENDPOINTS__`|`etcd_endpoints`中指定的etcd接入点。
|`__KUBECONFIG_FILEPATH__`|跟CNI网络配置文件在同一目录下自动生成的kubeconfig文件。
|`__ETCD_KEY_FILE__`|在节点上安装的etcd密钥文件。没有的话就留空。
|`__ETCD_CERT_FILE__`|在节点上安装的etcd证书文件，没有的话就留空。
|`__ETCD_CA_CERT_FILE__`|在节点上安装的etcd证书授权文件。没有的话就留空。

## 定制应用层策略manifests

### 关于应用层策略manifests

不用我们已经提前弄好的Istio manifests，定制自己的Istio安装或者装一个不同版本的Istio。这里我们要教你如何在一个普通的Istio manifests上通过必要的修改来实现应用层策略。

### Sidecar注入器

在标准的Istio manifests中，为sidecar注入器提供了一个ConfigMap，包含了添加Pod时要使用的模板。这个模板增加了一个初始化容器和一个Envoy sidecar。应用层策略需要一个额外的轻量级sidecar，名为Dikastes，它接收从Felix发过来的Calico策略，将它应用到进来的连接和请求上。

如果你啥还没开始弄，先下载[Isitio](https://github.com/istio/istio/releases)并解压。

在编辑器中打开`install/kubernetes/istio-demo-auth.yaml`文件，找到`istio-sidecar-injector`ConfigMap。在`istio-proxy`容器中，添加一个新的`volumeMount`。

```yaml
        - mountPath: /var/run/dikastes
          name: dikastes-sock
```

在模板中添加一个新的容器。

```yaml
      - name: dikastes
        image: calico/dikastes:v3.22.1
        args: ["server", "-l", "/var/run/dikastes/dikastes.sock", "-d", "/var/run/felix/nodeagent/socket"]
        securityContext:
          allowPrivilegeEscalation: false
        livenessProbe:
          exec:
            command:
            - /healthz
            - liveness
          initialDelaySeconds: 3
          periodSeconds: 3
        readinessProbe:
          exec:
            command:
            - /healthz
            - readiness
          initialDelaySeconds: 3
          periodSeconds: 3
        volumeMounts:
        - mountPath: /var/run/dikastes
          name: dikastes-sock
        - mountPath: /var/run/felix
          name: felix-sync
```

添加两个新的数据卷。

```yaml
      - name: dikastes-sock
        emptyDir:
          medium: Memory
      - name: felix-sync
        flexVolume:
          driver: nodeagent/uds
```

你建的这些数据卷是用来创建Unix domain socket的，允许Dikastes和Envoy、Dikastes和Felix之间进行通信。创建之后，Unix domain socket就是一个位于内存中的通信渠道。这些数据卷不负责任何基于磁盘的有状态存储。

参考[Calico ConfigMap manifest](https://projectcalico.docs.tigera.io/manifests/alp/istio-inject-configmap-1.4.2.yaml)，包含了以上修改的例子。
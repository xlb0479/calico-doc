## 配置calicoctl连接到Kubernetes API数据存储

## 默认配置

默认情况下，calicoctl会使用位于`$(HOME)/.kube/config`的kubeconfig来调用Kubernetes API。

如果默认的kubeconfig不存在，或者你想使用其它的API访问配置，可以使用以下配置选项。

## Kubernetes API连接配置
|**配置文件选项**|**环境变量**|**描述**|**值**
|-|-|-|-
|`datastoreType`|`DATASTORE_TYPE`|使用的数据存储类型。如果为指定，默认是`kubernetes`。（可选）|`kubernetes`，`etcdv3`
|`kubeconfig`|`KUBECONFIG`|使用Kubernetes数据存储时，指定kubeconfig文件的位置，例如/path/to/kube/config。|字符串
|`k8sAPIEndpoint`|`K8S_API_ENDPOINT`|Kubernetes API地址。如果用了kubeconfig可以不指定这个值。\[默认：`https://kubernetes-api:443`\]|字符串
|`k8sCertFile`|`K8S_CERT_FILE`|访问Kubernetes API的客户端证书位置。例如`/path/to/cert`。|字符串
|`k8sKeyFile`|`K8S_KEY_FILE`|访问Kubernetes API的客户端密钥位置。例如`/path/to/key`。|字符串
|`k8sCAFile`|`K8S_CA_FILE`|访问Kubernetes API的CA位置。例如`/path/to/ca`。|字符串
|`k8sToken`||访问Kubernetes API使用的token。|字符串

> 注意：所有环境变量都可以加`CALICO_`前缀，比如`CALICO_DATASTORE_TYPE`和`CALICO_KUBECONFIG`等都可以使用。如果你的环境变量名出现了冲突，有前缀的帮忙的话就会很方便了。

## 例子

### Kubernetes命令行

```shell
$ DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get nodes
```

### 配置文件

```yaml
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "/path/to/.kube/config"
```

### 使用环境变量

```shell
$ export DATASTORE_TYPE=kubernetes
$ export KUBECONFIG=~/.kube/config
$ calicoctl get workloadendpoints
```

使用`CALICO_`前缀：

```shell
$ export CALICO_DATASTORE_TYPE=kubernetes
$ export CALICO_KUBECONFIG=~/.kube/config
$ calicoctl get workloadendpoints
```

多个`kubeconfig`文件：

```shell
$ export DATASTORE_TYPE=kubernetes
$ export KUBECONFIG=~/.kube/main:~/.kube/auxy
$ calicoctl get --context main workloadendpoints
$ calicoctl get --context auxy workloadendpoints
```

### 检查配置

检查安装和配置是否正确。

```shell
$ calicoctl get nodes
```

如果安装正确，会得到一个已注册的节点列表。如果返回的列表为空，那么你要么是连错数据存储了，要么是节点都没有注册上。如果返回报错，那么就针对问题进行修改然后再试一试。
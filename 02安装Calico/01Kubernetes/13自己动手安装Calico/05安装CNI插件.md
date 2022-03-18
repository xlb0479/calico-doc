# 安装CNI插件

Kubernetes用容器网络接口（CNI）跟网络服务进行交互，比如Calico。Calico二进制中给出的这套Kubernetes API就叫**CNI插件**，必须安装在Kubernetes集群中的所有节点上。

要想了解容器网络接口（CNI）在Kubernetes中的工作方式，以及它是如何赋能Kubernetes网络的，请看我们的[Kubernetes CNI指引](https://www.tigera.io/learn/guides/kubernetes-networking/kubernetes-cni/)。

## 为插件创建Kubernetes user account

创建Pod，CNI插件跟Kubernetes API服务交互的时候，既要读取额外的信息，也要更新自己的数据存储，更新Pod的信息。

在Kubernetes的master节点上，为CNI插件的认证创建一个key，和CSR。

```
openssl req -newkey rsa:4096 \
           -keyout cni.key \
           -nodes \
           -out cni.csr \
           -subj "/CN=calico-cni"
```

用Kubernetes的主CA对这个证书进行签名。

```
sudo openssl x509 -req -in cni.csr \
                  -CA /etc/kubernetes/pki/ca.crt \
                  -CAkey /etc/kubernetes/pki/ca.key \
                  -CAcreateserial \
                  -out cni.crt \
                  -days 365
sudo chown $(id -u):$(id -g) cni.crt
```

然后我们为CNI插件创建一个kubeconfig文件，用来访问Kubernetes。

```shell
APISERVER=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')
kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=$APISERVER \
    --kubeconfig=cni.kubeconfig

kubectl config set-credentials calico-cni \
    --client-certificate=cni.crt \
    --client-key=cni.key \
    --embed-certs=true \
    --kubeconfig=cni.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=calico-cni \
    --kubeconfig=cni.kubeconfig

kubectl config use-context default --kubeconfig=cni.kubeconfig
```

将`cni.kubeconfig`文件复制到集群的所有节点上。

## 创建RBAC

为CNI插件定义一个集群角色，用来访问Kubernetes。

```shell
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-cni
rules:
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  # The CNI plugin patches pods/status.
  - apiGroups: [""]
    resources:
      - pods/status
    verbs:
      - patch
 # These permissions are required for Calico CNI to perform IPAM allocations.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - blockaffinities
      - ipamblocks
      - ipamhandles
    verbs:
      - get
      - list
      - create
      - update
      - delete
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - ipamconfigs
      - clusterinformations
      - ippools
    verbs:
      - get
      - list
EOF
```

将集群角色绑定到`calico-cni`账号上。

```
kubectl create clusterrolebinding calico-cni --clusterrole=calico-cni --user=calico-cni
```

## 安装插件

在每个节点上执行以下操作。

用root执行这些命令。

```shell
sudo su
```

安装CNI插件的二进制

```shell
curl -L -o /opt/cni/bin/calico https://github.com/projectcalico/cni-plugin/releases/download/v3.14.0/calico-amd64
chmod 755 /opt/cni/bin/calico
curl -L -o /opt/cni/bin/calico-ipam https://github.com/projectcalico/cni-plugin/releases/download/v3.14.0/calico-ipam-amd64
chmod 755 /opt/cni/bin/calico-ipam
```

创建配置目录

```shell
mkdir -p /etc/cni/net.d/
```

将之前弄好的kubeconfig复制过来

```shell
cp cni.kubeconfig /etc/cni/net.d/calico-kubeconfig
chmod 600 /etc/cni/net.d/calico-kubeconfig
```

搞一下CNI的配置

```shell
cat > /etc/cni/net.d/10-calico.conflist <<EOF
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
```

退出su回到登录的用户上。

```shell
exit
```

此时Kubernetes的节点都变成了`Ready`，因为Kubernetes有了一个网络服务，配置也弄好了。

```shell
kubectl get nodes
```
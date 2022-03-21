# 用户RBAC

本堂课我们要创建适用于生产环境急群众的RBAC。我们讲的是用Calico时的策略。生产环境Kubernetes集群的一般RBAC超出了本堂课的范围。

## 使用`calicoctl`

为了让`calicoctl`能够做版本匹配校验（确保集群和`calicoctl`的版本一致），用它的人需要在集群级别对`clusterinformation`有`get`权限，不是命名空间范围。下面给出的网络管理员角色就有这个权限，一会儿我们会把它加给服务拥有者。

```yaml
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calicoctl-user
rules:
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - clusterinformations
    verbs:
      - get
EOF
```

## 网络管理员

一个网络管理员是整体负责Calico网络配置和运维的人。所以他们需要访问所有的Calico自定义资源，以及一些相关的Kubernetes资源。

创建角色

```yaml
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: network-admin
rules:
  - apiGroups: [""]
    resources:
      - pods
      - nodes
    verbs:
      - get
      - watch
      - list
      - update
  - apiGroups: [""]
    resources:
      - namespaces
      - serviceaccounts
    verbs:
      - get
      - watch
      - list
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs: ["*"]
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - felixconfigurations
      - ipamblocks
      - blockaffinities
      - ipamhandles
      - ipamconfigs
      - bgppeers
      - bgpconfigurations
      - ippools
      - hostendpoints
      - clusterinformations
      - globalnetworkpolicies
      - globalnetworksets
      - networkpolicies
      - networksets
    verbs: ["*"]
EOF
```

为了测试这个网络管理员的角色，我们创建一个名为Nik的用户，并把这个角色授权给它。

在Kubernetes的master节点上，创建key和CSR。注意我们在subject中放的是`/O=network-admins`。这就把Nik放到了`network-admins`组中。

```shell
openssl req -newkey rsa:4096 \
           -keyout nik.key \
           -nodes \
           -out nik.csr \
           -subj "/O=network-admins/CN=nik"
```

用主Kubernetes CA对证书进行签名。

```shell
sudo openssl x509 -req -in nik.csr \
                  -CA /etc/kubernetes/pki/ca.crt \
                  -CAkey /etc/kubernetes/pki/ca.key \
                  -CAcreateserial \
                  -out nik.crt \
                  -days 365
sudo chown $(id -u):$(id -g) nik.crt
```

然后给Nik创建一个kubeconfig文件。

```shell
APISERVER=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')
kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=$APISERVER \
    --kubeconfig=nik.kubeconfig

kubectl config set-credentials nik \
    --client-certificate=nik.crt \
    --client-key=nik.key \
    --embed-certs=true \
    --kubeconfig=nik.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=nik \
    --kubeconfig=nik.kubeconfig

kubectl config use-context default --kubeconfig=nik.kubeconfig
```

将角色绑定到`network-admins`组中。

```shell
kubectl create clusterrolebinding network-admins --clusterrole=network-admin --group=network-admins
```

测试Nik的访问权限，创建一个全局网络集

```yaml
KUBECONFIG=./nik.kubeconfig calicoctl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: niks-set
spec:
  nets:
  - 110.120.130.0/24
  - 210.220.230.0/24
EOF
```

看看是不是有了

```shell
KUBECONFIG=./nik.kubeconfig calicoctl get globalnetworkset -o wide
```

结果

```
NAME       NETS
niks-set   110.120.130.0/24,210.220.230.0/24
```

删了它

```shell
KUBECONFIG=./nik.kubeconfig calicoctl delete globalnetworkset niks-set
```

## Service的拥有者

一个Service的拥有者负责Kubernetes中一个或多个Service的运维。他们应该能给他们的Service定义网络策略，但是不需要查看或修改任何Calico相关的全局配置。

定义角色

```yaml
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: network-service-owner
rules:
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs: ["*"]
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - networkpolicies
      - networksets
    verbs: ["*"]
EOF
```

测试这个Service拥有者的角色，我们创建一个名为Sam的用户，把这个角色授予他。

在Kubernetes的master节点，创建key和CSR。

```shell
openssl req -newkey rsa:4096 \
           -keyout sam.key \
           -nodes \
           -out sam.csr \
           -subj "/CN=sam"
```

用主Kubernetes的CA对这个证书签名。

```shell
sudo openssl x509 -req -in sam.csr \
                  -CA /etc/kubernetes/pki/ca.crt \
                  -CAkey /etc/kubernetes/pki/ca.key \
                  -CAcreateserial \
                  -out sam.crt \
                  -days 365
sudo chown $(id -u):$(id -g) sam.crt
```

然后，给Sam创建一个kubeconfig文件。

```shell
APISERVER=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')
kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=$APISERVER \
    --kubeconfig=sam.kubeconfig

kubectl config set-credentials sam \
    --client-certificate=sam.crt \
    --client-key=sam.key \
    --embed-certs=true \
    --kubeconfig=sam.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=sam \
    --kubeconfig=sam.kubeconfig

kubectl config use-context default --kubeconfig=sam.kubeconfig
```

我们限制Sam只能访问一个命名空间。创建命名空间

```shell
kubectl create namespace sam
```

在这个命名空间中给Sam绑定角色

```shell
kubectl create rolebinding -n sam network-service-owner-sam --clusterrole=network-service-owner --user=sam
```

同时还要把`calicoctl-user`角色在集群级别绑定给sam，这样可以正常使用`calicoctl`

```shell
kubectl create clusterrolebinding calicoctl-user-sam --clusterrole=calicoctl-user --user=sam
```

Sam无法创建全局网络集资源（像Nik似的是个网络管理员）

```shell
KUBECONFIG=./sam.kubeconfig calicoctl get globalnetworkset -o wide
```

结果

```
connection is unauthorized: globalnetworksets.crd.projectcalico.org is forbidden: User "sam" cannot list resource "globalnetworksets" in API group "crd.projectcalico.org" at the cluster scope
```

但是Sam可以在他自己的命名空间中创建资源

```yaml
KUBECONFIG=./sam.kubeconfig calicoctl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: NetworkSet
metadata:
  name: sams-set
  namespace: sam
spec:
  nets:
  - 110.120.130.0/24
  - 210.220.230.0/24
EOF
```

看看有了没

```shell
KUBECONFIG=./sam.kubeconfig calicoctl get networksets -n sam
```

结果

```
NAMESPACE   NAME
sam         sams-set
```

删掉NetworkSet

```shell
KUBECONFIG=./sam.kubeconfig calicoctl delete networkset sams-set -n sam
```
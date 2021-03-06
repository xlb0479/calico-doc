# 安装Typha

**Typha**位于Kubernetes API服务，跟各个节点上类似**Felix**和**confd**（运行在`calico/node`里）这种守护进程之间。它监视着这些守护进程使用的Kubernetes资源以及Calico自定义资源，当某个资源发生变更时，会将更新扇出（fanout）到这些守护进程上。这样减少了Kubernetes API服务需要支撑的监视数量，提高了集群的伸缩性。

## 创建证书

我们要创建双向TLS认证，确保`calico/node`和Typha的通信安全。本节，我们生成一个CA，用它为Typha签名证书。

创建CA证书和key

```shell
openssl req -x509 -newkey rsa:4096 \
                  -keyout typhaca.key \
                  -nodes \
                  -out typhaca.crt \
                  -subj "/CN=Calico Typha CA" \
                  -days 365
```

将CA证书保存到ConfigMap中，让Typha和`calico/node`能访问。

```shell
kubectl create configmap -n kube-system calico-typha-ca --from-file=typhaca.crt
```

创建Typha key和CSR

```shell
openssl req -newkey rsa:4096 \
           -keyout typha.key \
           -nodes \
           -out typha.csr \
           -subj "/CN=calico-typha"
```

证书中的Common Name（CN）是`calico-typha`。`calico/node`要配置并校验这个名字。

用CA对这个Typha证书签名

```shell
openssl x509 -req -in typha.csr \
                  -CA typhaca.crt \
                  -CAkey typhaca.key \
                  -CAcreateserial \
                  -out typha.crt \
                  -days 365
```

将Typha key和证书保存到secret中，让Typha能访问到。

```shell
kubectl create secret generic -n kube-system calico-typha-certs --from-file=typha.key --from-file=typha.crt
```

## 创建RBAC

创建一个用来运行Typha的ServiceAccount。

```shell
kubectl create serviceaccount -n kube-system calico-typha
```

为Typha定义一个集群角色，带有对Calico数据存储对象的监视权限。

```yaml
kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-typha
rules:
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
      - serviceaccounts
      - endpoints
      - services
      - nodes
    verbs:
      # Used to discover service IPs for advertisement.
      - watch
      - list
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - globalfelixconfigs
      - felixconfigurations
      - bgppeers
      - globalbgpconfigs
      - bgpconfigurations
      - ippools
      - ipamblocks
      - globalnetworkpolicies
      - globalnetworksets
      - networkpolicies
      - clusterinformations
      - hostendpoints
      - blockaffinities
      - networksets
    verbs:
      - get
      - list
      - watch
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      #- ippools
      #- felixconfigurations
      - clusterinformations
    verbs:
      - get
      - create
      - update
EOF
```

将集群角色绑定到`calico-typha`这个ServiceAccount上。

```shell
kubectl create clusterrolebinding calico-typha --clusterrole=calico-typha --serviceaccount=kube-system:calico-typha
```

## 安装Deployment

因为`calico/node`需要Typha，而且`calico/node`建立了Pod网络，我们要将Typha作为一个主机网络Pod来运行，避免鸡生蛋蛋生鸡的问题。我们跑了3个Typha副本，这样即便在滚动更新的时候，一个单点故障不会导致Typha不可用。

```yaml
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calico-typha
  namespace: kube-system
  labels:
    k8s-app: calico-typha
spec:
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      k8s-app: calico-typha
  template:
    metadata:
      labels:
        k8s-app: calico-typha
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: 'true'
    spec:
      hostNetwork: true
      tolerations:
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
      serviceAccountName: calico-typha
      priorityClassName: system-cluster-critical
      containers:
      - image: calico/typha:v3.8.0
        name: calico-typha
        ports:
        - containerPort: 5473
          name: calico-typha
          protocol: TCP
        env:
          # Disable logging to file and syslog since those don't make sense in Kubernetes.
          - name: TYPHA_LOGFILEPATH
            value: "none"
          - name: TYPHA_LOGSEVERITYSYS
            value: "none"
          # Monitor the Kubernetes API to find the number of running instances and rebalance
          # connections.
          - name: TYPHA_CONNECTIONREBALANCINGMODE
            value: "kubernetes"
          - name: TYPHA_DATASTORETYPE
            value: "kubernetes"
          - name: TYPHA_HEALTHENABLED
            value: "true"
          # Location of the CA bundle Typha uses to authenticate calico/node; volume mount
          - name: TYPHA_CAFILE
            value: /calico-typha-ca/typhaca.crt
          # Common name on the calico/node certificate
          - name: TYPHA_CLIENTCN
            value: calico-node
          # Location of the server certificate for Typha; volume mount
          - name: TYPHA_SERVERCERTFILE
            value: /calico-typha-certs/typha.crt
          # Location of the server certificate key for Typha; volume mount
          - name: TYPHA_SERVERKEYFILE
            value: /calico-typha-certs/typha.key
        livenessProbe:
          httpGet:
            path: /liveness
            port: 9098
            host: localhost
          periodSeconds: 30
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /readiness
            port: 9098
            host: localhost
          periodSeconds: 10
        volumeMounts:
        - name: calico-typha-ca
          mountPath: "/calico-typha-ca"
          readOnly: true
        - name: calico-typha-certs
          mountPath: "/calico-typha-certs"
          readOnly: true
      volumes:
      - name: calico-typha-ca
        configMap:
          name: calico-typha-ca
      - name: calico-typha-certs
        secret:
          secretName: calico-typha-certs
EOF
```

我们把`TYPHA_CLIENTCN`设置成`calico-node`，跟下一个实验中`calico/node`要用的证书中的common name一致。

检查Typha的三个实例都起来了没

```shell
kubectl get pods -l k8s-app=calico-typha -n kube-system
```

结果：

```shell
NAME                            READY   STATUS    RESTARTS   AGE
calico-typha-66498ddfbd-2pzsr   1/1     Running   0          69s
calico-typha-66498ddfbd-lrtzw   1/1     Running   0          50s
calico-typha-66498ddfbd-scckd   1/1     Running   0          62s
```

## 安装Service

`calico/node`使用一个Kubernetes Service来对Typha进行负载均衡。

```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: calico-typha
  namespace: kube-system
  labels:
    k8s-app: calico-typha
spec:
  ports:
    - port: 5473
      protocol: TCP
      targetPort: calico-typha
      name: calico-typha
  selector:
    k8s-app: calico-typha
EOF
```

确认Typha用了TLS。

```shell
TYPHA_CLUSTERIP=$(kubectl get svc -n kube-system calico-typha -o jsonpath='{.spec.clusterIP}')
curl https://$TYPHA_CLUSTERIP:5473 -v --cacert typhaca.crt
```

结果
```
* Rebuilt URL to: https://10.103.120.116:5473/
*   Trying 10.103.120.116...
* TCP_NODELAY set
* Connected to 10.103.120.116 (10.103.120.116) port 5473 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: typhaca.crt
  CApath: /etc/ssl/certs
* (304) (OUT), TLS handshake, Client hello (1):
* (304) (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS alert, Server hello (2):
* error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate
* stopped the pause stream!
* Closing connection 0
curl: (35) error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate
```

这样就说明Typha给出了它的TLS证书并拒绝了我们的连接，因为我们没有出示证书。下一个实验中我们会部署`calico/node`并带有Typha可接受的证书。
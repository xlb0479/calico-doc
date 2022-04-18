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

### 

## 2. 配置默认拒绝

## 3. 允许busybox的出口流量

## 4. 允许NGINX的入口流量

## 5. 清理
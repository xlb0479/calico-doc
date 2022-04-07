# 配置Kubernetes的控制面使用IPv6

## 大面儿

如果你的节点之间和工作负载之间打通了IPv6，那么Kubernetes控制面的通信你可能也想搞成IPv6的，不用IPv4了。

## 怎么弄

要让Kubernetes组件只使用IPv6，设置以下选项。

|**组件**|**选项**|**值/内容**
|-|-|-
|**kube-apiserver**
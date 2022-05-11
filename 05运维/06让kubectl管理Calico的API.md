# 让kubectl管理Calico的API

## 大面儿

**版本**：从Calico v3.20+开始进入GA

在集群中安装Calico API服务，让kubectl能够管理Calico的API。

## 价值

API服务可以提供Calico的REST API，允许通过kubectl管理`projectcalico.org/v3`API，无需使用calicoctl。

> 注意：从Calico v3.20.0开始，基于operator的安装默认就带API服务，不再需要本文讲的这些操作了。

以下略。
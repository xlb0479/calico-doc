# 配置calicoctl连接到etcd数据存储

## etcd的完整配置选项

|**配置文件选项**|**环境变量**|**描述**|**值**
|-|-|-|-
|`datastoreType`|`DATASTORE_TYPE`|使用的数据存储类型。如果为指定，默认是`kubernetes`。（可选）|`kubernetes`，`etcdv3`
|`etcdEndpoints`|`ETCD_ENDPOINTS`|逗号间隔的etcd地址列表，例如`http://127.0.0.1:2379,http://127.0.0.2:2379` （必填）|字符串
|`etcdDiscoverySrv`|`ETCD_DISCOVERY_SRV`|使用SRV记录做etcd地址发现时使用的域名。跟`etcdEndpoints`互斥。例如：`example.com`（可选）|字符串
|`etcdUsername`|`ETCD_USERNAME`|用户名。例如：`user`（可选）|字符串
|`etcdPassword`|`ETCD_PASSWORD`|密码。例如：`password`（可选）|字符串
|`etcdKeyFile`|`ETCD_KEY_FILE`|保存`calicoctl`客户端证书私钥文件的路径。能够让`calicoctl`开启双向TLS认证，向etcd服务器表明自己的身份。例如：`/etc/calicoctl/key.pem`（可选）|字符串
|`etcdCertFile`|`ETCD_CERT_FILE`|保存`calicoctl`的客户端证书文件的路径。能够让`calicoctl`开启双向TLS认证，向etcd服务器表明自己的身份。例如：`/etc/calicoctl/cert.pem`（可选）|字符串
|`etcdCACertFile`|`ETCD_CA_CERT_FILE`|签发etcd服务端证书的CA根证书文件路径。使`calicoctl`信任这个CA。文件中可以包含多个根证书，让`calicoctl`信任其中的每一个CA。例如：`/etc/calicoctl/ca.pem`（可选）|字符串
|`etcdKey`| |`calicoctl`客户端证书的私钥。能够让`calicoctl`开启双向TLS认证，向etcd服务器表明自己的身份。示例见下面。（可选）|字符串
|`etcdCert`| |签发给`calicoctl`的客户端证书。能够让`calicoctl`开启双向TLS认证，向etcd服务器表明自己的身份。示例见下面。（可选）|字符串
|`etcdCACert`| |签发etcd服务端证书的CA根证书。使`calicoctl`信任这个CA。文件中可以包含多个根证书，让`calicoctl`信任其中的每一个CA。（可选）|

> 注意：
> - 如果启用了TLS，确保接口地址使用了HTTPS。
> - 如果是通过环境变量指定，则对于etcdv3来说`DATASTORE_TYPE`是必填的。
> - 所有环境变量都可以加`CALICO_`前缀，比如`CALICO_DATASTORE_TYPE`和`CALICO_ETCD_ENDPOINTS`等都可以使用。如果你的环境变量名出现了冲突，有前缀的帮忙的话就会很方便了。
> - `etcdCACert`、`etcdCert`、`etcdKey`没有对应的环境变量。
> - 老版本的`calicoctl`支持`ETCD_SCHEME`和`ETCD_AUTHORITY`环境变量，作为指定etcd地址的一种机制。这些环境变量现在都不能用了。都用`ETCD_ENDPOINTS`。

## 例子

### 配置文件示例

```yaml
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  etcdEndpoints: https://etcd1:2379,https://etcd2:2379,https://etcd3:2379
  etcdKeyFile: /etc/calico/key.pem
  etcdCertFile: /etc/calico/cert.pem
  etcdCACertFile: /etc/calico/ca.pem
```

### 内联CA证书、客户端证书及密钥

```yaml
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: etcdv3
  etcdEndpoints: "https://127.0.0.1:2379"
  etcdCACert: |
      -----BEGIN CERTIFICATE-----
      MIICKzCCAZSgAwIBAgIBAzANBgkqhkiG9w0BAQQFADA3MQswCQYDVQQGEwJVUzER
      MA8GA1UEChMITmV0c2NhcGUxFTATBgNVBAsTDFN1cHJpeWEncyBDQTAeFw05NzEw
      MTgwMTM2MjVaFw05OTEwMTgwMTM2MjVaMEgxCzAJBgNVBAYTAlVTMREwDwYDVQQK
      EwhOZXRzY2FwZTENMAsGA1UECxMEUHViczEXMBUGA==
      -----END CERTIFICATE-----
  etcdCert: |
      -----BEGIN CERTIFICATE-----
      gI6iLXgMsp2EOlD56I6FA1jrCtNb01XQvX3eyFuA6g5T1jWGYBDtvQb0WRVkdUy9
      L/uK+sHQwtloCSuakcQAsWV9bajCQtHX8XGu25Yz56kpJ/OJjcishxT6pc/sthum
      A5PX739JsNUi/p5aG+H/6eNx+ukJP7QaM646YCfS5i8S9DJUvim+/BSlKi2ZiOCd
      0MYH4Xb7lmAOTNmTvSYpKo9J2fZ9erw0MYSBTyjh6F7PRbHBiivgUnJfGQ==
      -----END CERTIFICATE-----
  etcdKey: |
      -----BEGIN RSA PRIVATE KEY-----
      k0dWj16h9P6TvfcNl2iwT4VIwx0uy2faWBED1DrCJcuQCy5nPrts2ZIaAWPi1t3t
      VbDKQvs+KXBEeqh0qYcYkejUXqIF0uKUFLjiQmZssjpL5RHqqWuYKbO87n+Jod1L
      TjGRHdbP0zF2U0LdjM17rc2hpJ3qrmgJ7pOLzbXMcOr+NP1ojRCArXhQ4iLs7D8T
      eHw9QH4luJYtnmk7x03izLMQdLWcKnUbqh/xOVPyazgJHXwRxwNXpMsBVGY=
      -----END RSA PRIVATE KEY-----
```

### 使用环境变量

```shell
$ ETCD_ENDPOINTS=http://myhost1:2379 calicoctl get bgppeers
```

### 使用etcd的DNS发现

```shell
$ ETCD_DISCOVERY_SRV=example.com calicoctl get nodes
```

### IPv6

创建一个单节点etcd集群，监听在IPv6的localhost`[::1]`。

```shell
$ etcd --listen-client-urls=http://[::1]:2379 --advertise-client-urls=http://[::1]:2379
```

连接：

```shell
$ ETCD_ENDPOINTS=http://[::1]:2379 calicoctl get bgppeers
```

### IPv4/IPv6混用

创建一个单节点etcd集群，监听在IPv4和IPv6的localhost`[::1]`。

```shell
$ etcd --listen-client-urls=http://[::1]:2379,http://127.0.0.1:2379 --advertise-client-urls=http://[::1]:2379
```

连接到IPv6：

```shell
$ ETCD_ENDPOINTS=http://[::1]:2379 calicoctl get bgppeers
```

连接到IPv4：

```shell
$ ETCD_ENDPOINTS=http://127.0.0.1:2379 calicoctl get bgppeers
```

## calico/node

需要注意的是，calicoctl不光是直接在主机上使用这些key来访问etcd，**它还会传递环境变量，挂载key到`calico-node`容器中。**

因此，执行`calicoctl node run`时只要参数正确，配置`calico/node`非常简单。

### 检查配置

检查安装和配置是否正确。

```shell
$ calicoctl get nodes
```

如果安装正确，会得到一个已注册的节点列表。如果返回的列表为空，那么你要么是连错数据存储了，要么是节点都没有注册上。如果返回报错，那么就针对问题进行修改然后再试一试。
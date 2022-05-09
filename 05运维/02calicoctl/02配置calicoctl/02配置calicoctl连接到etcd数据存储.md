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
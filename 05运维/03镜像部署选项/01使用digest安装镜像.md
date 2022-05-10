# 使用digest安装镜像

## 大面儿

使用operator通过容器仓库digest部署镜像。

## 价值

有的环境对安全要求比较严格，要求部署镜像的时候必须基于不可变的digest，而不是tag。一经发布，官方的Calico镜像和tag都不会再修改。但使用不可变的digest，安全团队可以对特定镜像进行审查和校验。

## 特性

这里我们用到了Calico的以下特性：

- **ImageSet**

## 概念

### 容器仓库

通过容器仓库可以用tag或digest访问容器镜像。

### 镜像tag

版本化容器镜像通常都是通过标签来引用，加在镜像名字后面。例如：`<repo>/<image>:<tag>`。容器镜像的tag通常不会改也不更新，但是大部分镜像仓库并不强制要求这一点，也就是说允许往同一个tag上继续推送新的代码。

### 镜像digest

容器镜像加入到仓库后，都会被赋予一个唯一的哈希值，可以用来拉取镜像的特定版本，它无法修改或更新。

## 开始之前

### 条件

- 基于operator的Calico
- 配好Docker客户端，可以从仓库拉取镜像
- 具备Kubernetes权限，可以在集群中部署ImageSet

## 怎么弄

1. [使用digest更新operator部署](#使用digest更新operator部署)
2. [创建ImageSet](#创建ImageSet)
3. [查看是否使用了正确的ImageSet](#查看是否使用了正确的ImageSet)

### 其它任务

- [升降级时创建新的ImageSet](#升降级时创建新的ImageSet)

### 排错

- [为什么Installation资源的状态中没有包含我的ImageSet？](#为什么Installation资源的状态中没有包含我的ImageSet？)
- [怎么能看出我的ImageSet有没有问题？](#怎么能看出我的ImageSet有没有问题？)

### 使用digest更新operator部署

创建`tigera-operator.yaml`之前，修改一下，使用镜像digest。

使用如下命令获取镜像的digest（使用不同的operator镜像时做针对性的修改）：

```shell
$ docker pull quay.io/tigera/operator:v1.25.7
$ docker inspect quay.io/tigera/operator:v1.25.7 -f '{{range .RepoDigests}}{{printf "%s\n" .}}{{end}}'
```

如果返回了多个digest，选择一个跟你使用的仓库匹配的。

更新tigera-operator：

```shell
$ sed -ie "s|\(image: .*/operator\):.*|?\1@<put-digest-here>|" tigera-operator.yaml
```

### 创建ImageSet

创建一个[ImageSet](../../06%E5%8F%82%E8%80%83/02%E5%AE%89%E8%A3%85.md#ImageSet)，文件名为`imageset.yaml`：

```yaml
apiVersion: operator.tigera.io/v1
kind: ImageSet
metadata:
  name: calico-v3.22.2
spec:
  images:
  - image: "calico/apiserver"
    digest: "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  - image: "calico/cni"
    digest: "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  - image: "calico/kube-controllers"
    digest: "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  - image: "calico/node"
    digest: "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  - image: "calico/typha"
    digest: "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  - image: "calico/pod2daemon-flexvol"
    digest: "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  - image: "calico/windows-upgrade"
    digest: "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  - image: "tigera/operator"
    digest: "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
  - image: "tigera/key-cert-provisioner"
    digest: "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
```

然后可以手动或基于脚本进行。

#### 手动

1.将上面的内容复制到`imageset.yaml`中，然后再下面的操作中进行编辑。
2.将ImageSet的名字改为`calico-<version>`（例如：`calico-v3.22.2`）。版本可以通过以下命令获取：

```shell
docker run quay.io/tigera/operator:v1.25.7 --version
```

3.给每个镜像添加正确的digest。如果你用的是私有仓库，确保你是从私有仓库拉的，而且使用的是自由仓库的digest。

- 如果使用的默认镜像，获取镜像列表：

```shell
docker run quay.io/tigera/operator:v1.25.7 --print-images=list
```

> 注意：如果用的不是默认镜像仓库或路径，必须创建你自己的镜像列表（上面的命令就不用了）。

> 注意：列表中包含了企业版的镜像，不需要添加到ImageSet中。

- 将上面得到的镜像放到下面的命令中，获取它们的digest：

```shell
docker pull <repo/image:tag> && docker inspect <repo/image:tag> -f '{{range .RepoDigests}}{{printf "%s\n" .}}{{end}}'
```

- 使用repo/image对应的digest。如果你用的是私仓，或者指定了一个[imagePath](../../06%E5%8F%82%E8%80%83/02%E5%AE%89%E8%A3%85.md#Installation)，那么在`image`字段中你就还用“默认的”`<owner>/<image>`，比如你的节点镜像是`example.com/registry/imagepath/node`，那么在ImageSet的镜像字段中就还是使用`calico/node`。

> 示例：对于镜像`quay.io/tigera/operator@sha256:d111db2f94546415a30eff868cb946d47e183faa804bd2e9a758fd9a8a4eaff1`，复制`@`后面的内容，作为`tigera/operator`镜像的digest。

最后，在集群中创建`imageset.yaml`。

#### 脚本

把下面的脚本复制到一个文件中，增加可执行权限并运行脚本。这个脚本会在当前目录下创建一个`imageset.yaml`文件。

> 注意：该脚本只能用于默认仓库和imagePath。

```bash
#!/bin/bash -e

images=(calico/apiserver calico/cni calico/kube-controllers calico/node calico/typha calico/pod2daemon-flexvol calico/windows-upgrade tigera/key-cert-provisioner tigera/operator)

OPERATOR_IMAGE=quay.io/tigera/operator:v1.25.7
echo "Pulling $OPERATOR_IMAGE"
echo
docker pull $OPERATOR_IMAGE -q >/dev/null
versions=$(docker run $OPERATOR_IMAGE --version)
ver=$(echo -e "$versions" | grep 'Calico:')

imagelist=($(docker run $OPERATOR_IMAGE --print-images=list))

cat > ./imageset.yaml <<EOF
apiVersion: operator.tigera.io/v1
kind: ImageSet
metadata:
  name: calico-$(echo $ver | sed -e 's|^.*: *||')
spec:
  images:
EOF

for x in "${imagelist[@]}"; do
  for y in ${images[*]}; do
    if [[ $x =~ $y: ]]; then
      digest=$(docker run --rm gcr.io/go-containerregistry/crane:v0.7.0 digest ${x})
      echo "Adding digest for $x"
      echo 
      echo "  - image: \"$(echo $x | sed -e 's|^.*/\([^/]*/[^/]*\):.*$|\1|')\"" >> ./imageset.yaml
      echo "    digest: \"$digest\"" >> ./imageset.yaml
    fi
  done
done
```

最后，在集群中创建`imageset.yaml`。

### 查看是否使用了正确的ImageSet

1. 执行`kubectl get tigerastatus`查看tigerastatus，看是否有Degraded。
    - 如果有组件显示Degraded，[继续检查](#怎么能看出我的ImageSet有没有问题？)。
2. 当所有组件的tigerastatus都显示Available True，则ImageSet已生效。

```text
NAME     AVAILABLE   PROGRESSING   DEGRADED   SINCE
calico   True        False         False      54s
```

3. 查看是否使用了正确的ImageSet。在Installation的状态中，查看`imageset`字段，是否是你创建的ImageSet。执行下面的命令：

```shell
kubectl get installation default -o yaml | grep imageSet
```

应该会看到：

```yaml
    imageSet: calico-v3.22.2
```

## 其它任务

### 升降级时创建新的ImageSet

升降级之前，必须用新的镜像引用和名字创建一个新的[ImageSet](../../06%E5%8F%82%E8%80%83/02%E5%AE%89%E8%A3%85.md#ImageSet)。这个必须得提前弄，这样在使用新的manifest的时候，就可以使用正确的ImageSet了。

### 为什么Installation资源的状态中没有包含我的ImageSet？

Installation资源的[status.imageset](../../06%E5%8F%82%E8%80%83/02%E5%AE%89%E8%A3%85.md#InstallationStatus)字段直到`calico`组件完全部署完成后才会更新。只有当`kubectl get tigerastatus calico`返回的Available是True并且Progressing和Degraded都是False的情况下`calico`才是完全部署完成。

### 怎么能看出我的ImageSet有没有问题？

如果你怀疑你的ImageSet有问题，执行`kubectl get tigerastatus`检查tigerastatus。如果有组件处于降级状态，可以通过`kubectl get tigerastatus <component-name> -o yaml`查看详细信息。如果某个镜像的digest不对，或者拉不下来，tigerastatus中不会直接体现这一点，但是你会看到Deployment、Daemonset、或Job发布有问题。如果你怀疑这是因为镜像导致的，那你需要对指定的Pod进行`get`或`describe`，查看问题详情。
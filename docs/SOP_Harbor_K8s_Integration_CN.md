# Harbor 与 Kubernetes 对接标准操作流程 (SOP)

本文档适用于在 VMware 虚拟机上搭建的 2 节点 Kubernetes 集群。当前底层容器运行时为 `containerd`。

## 阶段一：前期准备工作

### 1. 安装 Helm
如果您的主节点还未安装 Helm，请执行以下命令安装：
```bash
wget https://get.helm.sh/helm-v4.2.1-linux-amd64.tar.gz
tar -zxvf helm-v4.2.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
```

### 2. 配置 containerd 信任不安全的 Registry (HTTP)
由于 Harbor 采用 HTTP 提供服务（若使用自签名证书也会报不信任），我们需要配置 2 台虚拟机的 containerd，让其信任 Harbor 仓库，否则在 `pull` 或 `push` 镜像时会报 `http: server gave HTTP response to HTTPS client` 的错误。

> [!IMPORTANT]
> 以下步骤需要在 **所有 K8s 节点（Master 和 Node）** 上执行。

假设您的 Harbor 将运行在虚拟机的 IP 上，设该 IP 为 `192.168.9.41`（例如 `192.168.1.100`），端口映射为 `30002`。

#### 第一步：开启 containerd 的 certs.d 配置目录支持
默认情况下，新建的 K8s 集群可能没有开启外部 Registry 配置目录。我们需要修改 containerd 的主配置文件：

1. 编辑配置文件 `/etc/containerd/config.toml`（如果没有该文件，可通过 `containerd config default > /etc/containerd/config.toml` 生成一份默认配置）。
2. 在文件中找到 `[plugins.'io.containerd.cri.v1.images'.registry]` 这一段。
3. 确保包含 `config_path = "/etc/containerd/certs.d"`，修改后应该是这样的：
   ```toml
   [plugins.'io.containerd.cri.v1.images'.registry]
     config_path = '/etc/containerd/certs.d'
   ```

#### 第二步：创建 Harbor 的专属镜像仓库配置 (hosts.toml)
containerd 采用基于目录的配置方式，我们需要为我们的 Harbor 地址创建一个同名文件夹。

```bash
# 1. 创建存放证书/配置的目录 (目录名必须与镜像仓库地址完全一致)
mkdir -p /etc/containerd/certs.d/192.168.9.41:30002

# 2. 写入 hosts.toml 配置，明确告知使用 http 协议并跳过 TLS 验证
cat <<EOF > /etc/containerd/certs.d/192.168.9.41:30002/hosts.toml
server = "http://192.168.9.41:30002"

[host."http://192.168.9.41:30002"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
EOF
```

#### 第三步：重启服务使配置生效
修改完成后，重启 containerd 进程：
```bash
systemctl daemon-reload
systemctl restart containerd
systemctl status containerd
```

*(备注：如果您计划配置域名访问，如 `hub.local.com:30002`，那么上述所有的 `192.168.9.41:30002` 都需要替换为 `hub.local.com:30002`，并确保集群中所有节点的 `/etc/hosts` 中添加了该域名的解析：`192.168.9.41 hub.local.com`。)*

---

## 阶段二：部署 Harbor

我们将使用 Helm 将 Harbor 的 Chart 拉取到本地后再进行部署。这有助于您查看原生配置模板，也非常适合在内网或离线环境中操作。

### 1. 添加仓库并拉取 Harbor Chart 到本地
首先，添加官方仓库并更新：
```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
```

然后，将 Harbor 的 Helm Chart 下载到本地并解压：
```bash
helm pull harbor/harbor --untar
```
执行完毕后，您所在的当前目录下会多出一个名为 `harbor` 的文件夹，里面包含了该版本所有的模板文件和默认 `values.yaml`。

### 2. 使用本地 Chart 和自定义 `harbor-values.yaml` 安装
将本项目生成的自定义配置文件 `harbor-values.yaml` 上传到您的主节点，并确保它放在与刚解压出来的 `harbor` 目录**同级的路径下**。

创建命名空间，并指定**本地解压的目录**执行安装：
```bash
kubectl create namespace harbor
# 安装 harbor，注意这里使用的是本地的 ./harbor 目录
helm install my-harbor ./harbor -n harbor -f harbor-values.yaml
```

等待 Pod 启动完毕：
```bash
kubectl get pods -n harbor -w
```

---

## 阶段三：Harbor 基本使用与镜像推送

### 1. 在 Harbor 中创建项目
1. 登录 Harbor UI。
2. 点击“新建项目”。
3. 项目名称填写 `my-project`。
4. 访问级别保持为默认的**私有**。

### 2. 安装与配置 NERDCTL 或 Docker
由于您使用的是 containerd，建议安装 `nerdctl` 来进行镜像构建和推送（操作习惯与 docker 一致），或者直接使用 `ctr` / `crictl`。如果节点上已有 Docker，也可以直接用 Docker push。

登录 Harbor：
```bash
# 使用 nerdctl 或 docker
nerdctl login -u admin -p Harbor12345 192.168.9.41:30002
```

获取一个测试镜像并推送到 Harbor：
```bash
# 拉取公共镜像
nerdctl pull nginx:alpine

# 重新打标签，格式：<Harbor地址>/<项目名>/<镜像名>:<标签>
nerdctl tag nginx:alpine 192.168.9.41:30002/my-project/nginx:alpine

# 推送到私有仓库
nerdctl push 192.168.9.41:30002/my-project/nginx:alpine
```

---

## 阶段四：Kubernetes 部署与对接

K8s 拉取私有镜像需要凭证（Secret）。

### 1. 创建拉取凭证 (ImagePullSecret)
您可以使用命令行快速创建，也可以使用本项目中的 `k8s/harbor-registry-secret.yaml`。

命令行方式：
```bash
kubectl create secret docker-registry harbor-login-secret \
  --docker-server=192.168.9.41:30002 \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  -n default
```

### 2. 部署应用测试拉取
执行本项目的部署清单：

```bash
kubectl apply -f k8s/test-deployment.yaml
```

验证 Pod 是否成功拉取镜像并运行：
```bash
kubectl get pods
kubectl describe pod <pod-name>
```
如果输出最后显示 `Successfully pulled image...` 且状态为 `Running`，则恭喜您，对接成功！

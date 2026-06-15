# Harbor 与 Kubernetes 对接标准操作流程 (SOP)

本文档适用于在 VMware 虚拟机上搭建的 2 节点 Kubernetes 集群。当前底层容器运行时为 `containerd`。

## 阶段一：前期准备工作

### 1. 安装 Helm
如果您的主节点还未安装 Helm，请执行以下命令安装：
```bash
wget https://get.helm.sh/helm-v3.12.0-linux-amd64.tar.gz
tar -zxvf helm-v3.12.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
```

### 2. 配置 containerd 信任不安全的 Registry (HTTP)
由于我们仅作学习，Harbor 将通过 HTTP 提供服务（若使用自签名证书也会报不信任）。我们需要配置 2 台虚拟机的 containerd，让其信任 Harbor 仓库。

> [!IMPORTANT]
> 以下步骤需要在 **所有 K8s 节点（Master 和 Node）** 上执行。

假设您的 Harbor 将运行在虚拟机的 IP 上，我们设定该 IP 为 `<VM_IP>`（例如 `192.168.1.100`），端口我们假设映射为 `30002`。

修改 containerd 的配置文件 `/etc/containerd/config.toml`：

```bash
mkdir -p /etc/containerd/certs.d/<VM_IP>:30002

cat <<EOF > /etc/containerd/certs.d/<VM_IP>:30002/hosts.toml
server = "http://<VM_IP>:30002"

[host."http://<VM_IP>:30002"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
EOF

# 重启 containerd 使配置生效
systemctl restart containerd
```

*(备注：如果您配置了域名访问，如 `hub.local.com`，需要将 `<VM_IP>:30002` 替换为域名，并确保集群中所有节点的 `/etc/hosts` 中添加了该域名的解析 `<VM_IP> hub.local.com`。)*

---

## 阶段二：部署 Harbor

我们将使用 Helm 部署 Harbor。

### 1. 添加 Harbor Helm 仓库
```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
```

### 2. 根据 `harbor-values.yaml` 安装
将本项目中的 `harbor-values.yaml` 文件上传到您的主节点。
执行部署命令：
```bash
kubectl create namespace harbor
# 安装 harbor (发布名称为 my-harbor)
helm install my-harbor harbor/harbor -n harbor -f harbor-values.yaml
```

等待 Pod 启动完毕：
```bash
kubectl get pods -n harbor -w
```
部署成功后，您可以在浏览器中通过 `http://<VM_IP>:30002` 访问 Harbor。
默认管理员账号：`admin`
默认密码：`Harbor12345`

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
nerdctl login -u admin -p Harbor12345 <VM_IP>:30002
```

获取一个测试镜像并推送到 Harbor：
```bash
# 拉取公共镜像
nerdctl pull nginx:alpine

# 重新打标签，格式：<Harbor地址>/<项目名>/<镜像名>:<标签>
nerdctl tag nginx:alpine <VM_IP>:30002/my-project/nginx:alpine

# 推送到私有仓库
nerdctl push <VM_IP>:30002/my-project/nginx:alpine
```

---

## 阶段四：Kubernetes 部署与对接

K8s 拉取私有镜像需要凭证（Secret）。

### 1. 创建拉取凭证 (ImagePullSecret)
您可以使用命令行快速创建，也可以使用本项目中的 `k8s/harbor-registry-secret.yaml`。

命令行方式：
```bash
kubectl create secret docker-registry harbor-login-secret \
  --docker-server=<VM_IP>:30002 \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  -n default
```

### 2. 部署应用测试拉取
修改本项目中 `k8s/test-deployment.yaml` 里的 `<VM_IP>` 为您的实际 IP，然后执行：

```bash
kubectl apply -f k8s/test-deployment.yaml
```

验证 Pod 是否成功拉取镜像并运行：
```bash
kubectl get pods
kubectl describe pod <pod-name>
```
如果输出最后显示 `Successfully pulled image...` 且状态为 `Running`，则恭喜您，对接成功！

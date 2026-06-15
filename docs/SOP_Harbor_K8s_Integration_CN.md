# Harbor 与 Kubernetes 对接标准操作流程 (SOP) - 完整实战版

本手册记录了在 VMware 虚拟机上搭建的 Kubernetes 集群中，从零开始部署 Harbor 并与集群打通完整镜像拉取链路的全过程。

**环境基准信息：**
* **底层环境**：VMware 虚拟机搭建的 K8s 集群
* **容器运行时**：containerd
* **访问方式**：NodePort + HTTP 协议（直接使用 Master 节点 IP `192.168.9.41`，端口 `30002`）

---

## 阶段一：前期准备工作

### 1. 安装 Helm 部署工具
在主节点执行以下命令安装 Helm，用于后续部署 Harbor：
```bash
wget https://get.helm.sh/helm-v4.2.1-linux-amd64.tar.gz
tar -zxvf helm-v4.2.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
```

### 2. 安装 NERDCTL 客户端
因为 K8s 底层是 containerd，强烈建议安装轻量级的 `nerdctl` 客户端（它完美兼容 docker 命令，用来操作镜像非常方便）：
```bash
wget https://github.com/containerd/nerdctl/releases/download/v1.7.6/nerdctl-1.7.6-linux-amd64.tar.gz
tar -zxvf nerdctl-1.7.6-linux-amd64.tar.gz -C /usr/local/bin/ nerdctl
nerdctl version
```

### 3. 配置 containerd 信任不安全的 Registry (HTTP)
> [!IMPORTANT]
> 此步骤需要在**所有 K8s 节点（Master 和 Node）**上执行。否则 Kubernetes 节点无法拉取私有库镜像，会报错 `http: server gave HTTP response to HTTPS client`。

**第一步：开启 containerd 的外部配置支持**
确保 `/etc/containerd/config.toml` 中配置了 `certs.d` 目录支持：
```toml
   [plugins.'io.containerd.cri.v1.images'.registry]
     config_path = '/etc/containerd/certs.d'
```

**第二步：创建 Harbor 专属信任配置**
目录名必须和待访问的 Harbor 地址（含端口）一模一样。
```bash
mkdir -p /etc/containerd/certs.d/192.168.9.41:30002

cat <<EOF > /etc/containerd/certs.d/192.168.9.41:30002/hosts.toml
server = "http://192.168.9.41:30002"

[host."http://192.168.9.41:30002"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
EOF
```

**第三步：重启生效**
```bash
systemctl daemon-reload
systemctl restart containerd
```

---

## 阶段二：部署 Harbor

我们采用将 Helm Chart 下载到本地后修改源文件的方式进行安装，方便管控配置。

### 1. 拉取 Harbor Chart 到本地
```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
helm pull harbor/harbor --untar
```
执行完毕后，当前目录下会生成一个 `harbor` 文件夹。

### 2. 修改 `harbor/values.yaml` (核心配置)
进入 `harbor` 目录，编辑 `values.yaml`。为了适配我们的虚拟机测试环境，请精准修改以下五处：

1. **暴露方式设为 NodePort 并关闭 HTTPS**
   ```yaml
   expose:
     type: nodePort      # 默认为 ingress，改为 nodePort
     tls:
       enabled: false    # 默认为 true，改为 false
   ```
2. **指定 HTTP 暴露的端口**
   找到 `nodePort.ports.http.nodePort`，将其设为固定的 `30002`（如果没找到或者被注释了，可以手动加上）。
3. **修改外部访问 URL**
   这决定了 Harbor 内部系统互相通信及提供给客户端的 URL。
   ```yaml
   externalURL: http://192.168.9.41:30002
   ```
4. **关闭持久化存储 (防卡壳)**
   虚拟机 K8s 默认没有提供 StorageClass，如果开着持久化，Harbor 会卡在 Pending 状态。
   ```yaml
   persistence:
     enabled: false
   ```
5. **精简资源，关闭扫描组件 (可选)**
   为了节约虚拟机内存，关闭漏洞扫描工具。
   ```yaml
   trivy:
     enabled: false
   ```

### 3. 执行本地安装
退回到 `harbor` 目录的上一级，执行安装：
```bash
kubectl create namespace harbor
helm install my-harbor ./harbor -n harbor
```
等待大约 2 分钟，使用 `kubectl get pods -n harbor` 观察，全部变为 Running 且 1/1 或 2/2 即为成功（起初 jobservice 可能会重启几次，属于正常依赖等待现象）。

---

## 阶段三：Harbor 仓库操作与镜像推送

### 1. UI 界面手动创建私有项目
> [!CAUTION]
> Harbor **不会** 在 push 时自动创建项目！如果您 push 到一个不存在的项目，会直接报 `401 Unauthorized`。

1. 浏览器访问 `http://192.168.9.41:30002`
2. 账号 `admin`，密码 `Harbor12345`（默认密码）。
3. 手动点击新建项目，名称为 `my-project`，务必保持级别为 **公开关闭（即私有 Private）**。

### 2. 使用 nerdctl 登录并推送
由于我们使用纯 HTTP 协议，命令行非常严格，必须在登录和推送时加上 `--insecure-registry`：

```bash
# 登录仓库 (必须带 --insecure-registry)
nerdctl login --insecure-registry -u admin -p Harbor12345 192.168.9.41:30002

# 拉取测试镜像并打标签
nerdctl pull nginx:alpine
nerdctl tag nginx:alpine 192.168.9.41:30002/my-project/nginx:alpine

# 推送到私有库
nerdctl push --insecure-registry 192.168.9.41:30002/my-project/nginx:alpine
```
如果看到进度条并显示 SHA 值，即代表推送成功。

---

## 阶段四：Kubernetes 对接验证测试

要让 Kubernetes 从我们刚刚搭建的私有项目 `my-project` 中拉取镜像，必须提供认证密钥（Secret）。

### 1. 创建镜像拉取凭证 (ImagePullSecret)
您可以使用一行命令快捷生成，K8s 会自动在后台将账号密码转成 Base64 存储：
```bash
kubectl create secret docker-registry harbor-login-secret \
  --docker-server=192.168.9.41:30002 \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  -n default
```
*(注：如果目标命名空间不是 default，记得修改 `-n` 参数)*

### 2. 编写与部署 Deployment
为了确保测试效果，我们要保证它去网络拉取，而不是使用本地缓存。创建一个 `test.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: harbor-private-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: harbor-test
  template:
    metadata:
      labels:
        app: harbor-test
    spec:
      containers:
      - name: nginx
        image: 192.168.9.41:30002/my-project/nginx:alpine
        # 必须设置为 Always，强制 K8s 每次都去 Harbor 重新拉取
        imagePullPolicy: Always
        ports:
        - containerPort: 80
      # 挂载我们在第一步创建的拉取密钥
      imagePullSecrets:
      - name: harbor-login-secret
```

部署并验证：
```bash
kubectl apply -f test.yaml
kubectl get pods -w
```
当 Pod 状态变更为 `Running` 时，可以通过 `kubectl describe pod <pod名字>`，查看底部的 Events。
如果看到类似 `Successfully pulled image "192.168.9.41:30002/my-project/nginx:alpine"` 的提示，说明 K8s 成功使用了 `harbor-login-secret` 密钥，完成了私有镜像库的完美对接！

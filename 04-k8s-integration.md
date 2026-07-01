# Kubernetes 与 Harbor 对接

Kubernetes 与 Harbor 的对接通常涉及两部分：配置节点信任 HTTP 仓库（解决 `connection refused: 443` 报错），以及配置私有镜像拉取凭证（解决 `Unauthorized` 报错）。

## 0. 配置节点信任不安全仓库 (HTTP)

如果你的 Harbor 使用的是 HTTP 协议，Kubernetes 节点在拉取镜像时会因为默认强制使用 HTTPS 而报错。你需要在**所有 K8s Node 节点**上修改容器运行时配置。

以较新版本 Kubernetes 默认的 **Containerd** 为例：

1. 编辑 `/etc/containerd/config.toml`，找到 `[plugins.'io.containerd.cri.v1.images'.registry]`，将 `config_path` 修改为：
   ```toml
   [plugins.'io.containerd.cri.v1.images'.registry]
     config_path = '/etc/containerd/certs.d'
   ```
2. 创建配置目录：`mkdir -p /etc/containerd/certs.d/192.168.9.5`
3. 创建 `/etc/containerd/certs.d/192.168.9.5/hosts.toml` 并写入：
   ```toml
   server = "http://192.168.9.5"

   [host."http://192.168.9.5"]
     capabilities = ["pull", "resolve", "push"]
     skip_verify = true
   ```
4. 重启 containerd：`systemctl restart containerd`

## 1. 创建 Docker Registry Secret (拉取凭证)

当网络连通后，如果镜像存放在 Harbor 的私有项目中，我们需要配置**镜像拉取凭证 (imagePullSecrets)**。Kubernetes 提供了一种特殊类型的 Secret 用于存储 Docker Registry 的登录凭证。

### 方法一：使用 kubectl 命令创建

如果你能直接在命令行提供账号密码：

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=192.168.9.5 \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --docker-email=admin@example.com \
  -n your-namespace
```
*(注意：替换 server, username, password 和 namespace 为你的实际值)*

### 方法二：使用现有 `config.json` 创建

如果你已经在机器上执行了 `docker login`，可以使用生成的 `~/.docker/config.json` 来创建 Secret：

```bash
kubectl create secret generic harbor-secret \
  --from-file=.dockerconfigjson=/root/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson \
  -n your-namespace
```

## 2. 在 Pod/Deployment 中使用该 Secret

在你的 Kubernetes yaml 清单（如 Deployment）中，通过 `imagePullSecrets` 字段引用刚才创建的 secret。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: your-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        # 填写 Harbor 中的镜像地址
        image: 192.168.9.5/library/my-nginx:v1.0 
        ports:
        - containerPort: 80
      # 引用上面创建的 secret
      imagePullSecrets:
      - name: harbor-secret
```

部署此应用：

```bash
kubectl apply -f my-app.yaml
```

## 3. (高级进阶) 配置 ServiceAccount

如果你不想每次在 Deployment 中都手写 `imagePullSecrets`，你可以将这个 Secret 绑定到命名空间的默认 `ServiceAccount` 上。

```bash
# 假设你在 default 命名空间
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "harbor-secret"}]}'
```

这样，该命名空间下新创建的 Pod，如果没有显式指定 `imagePullSecrets`，也会自动使用 `harbor-secret` 去拉取私有镜像。

## 总结

Kubernetes 与 Harbor 对接的核心逻辑就是：
1. Harbor 配置好项目访问权限（公开或私有）。如果是私有，则需要凭证。
2. K8s 将拉取凭证存储为 `kubernetes.io/dockerconfigjson` 类型的 Secret。
3. K8s 资源清单在调度 Pod 时声明使用该 Secret 去拉取镜像。

# Harbor 在 Kubernetes 中的部署

在生产环境中，我们通常会将 Harbor 部署在 Kubernetes 集群内部，利用 K8s 的高可用特性。官方推荐使用 [Helm Chart](https://github.com/goharbor/harbor-helm) 进行部署。

## 前置要求

- 一个运行中的 Kubernetes 集群
- 已安装并配置好 Helm 3+
- 集群中配置了 StorageClass (Harbor 需要持久化存储来保存镜像、数据库、Redis 数据等)
- Ingress Controller (如 Nginx Ingress) - 可选但推荐

## 部署步骤

### 1. 添加 Harbor Helm 仓库

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
```

### 2. 配置 values.yaml

下载官方的默认 `values.yaml` 来进行定制：

```bash
helm show values harbor/harbor > my-values.yaml
```

修改 `my-values.yaml` 中的关键配置：

- **expose.type**: 暴露服务的方式，通常使用 `ingress` 或 `clusterIP` (如果你使用 NodePort 等其他方式)。
- **expose.tls**: 如果配置了证书，启用。如果只是内网 HTTP 测试，可以将 `expose.tls.enabled` 设置为 `false`。
- **externalURL**: Harbor 外部访问的 URL，例如 `http://core.harbor.domain`。
- **persistence**:
  - 确保 `persistentVolumeClaim.registry.storageClass` 指定了你集群中可用的 StorageClass（如 `standard`、`nfs-client` 等）。如果集群有默认 SC，可以留空。
- **harborAdminPassword**: 设置管理员初始密码。

### 3. 执行安装

创建一个命名空间并安装 Harbor：

```bash
kubectl create namespace harbor
helm install my-harbor harbor/harbor -n harbor -f my-values.yaml
```

### 4. 验证部署

查看 Pod 状态：

```bash
kubectl get pods -n harbor
```

当所有的 pod（core, portal, registry, jobservice, database, redis 等）都变成 `Running` 状态时，说明部署成功。
此时你可以通过配置的 Ingress 域名或暴露的端口在浏览器中访问 Harbor Portal。

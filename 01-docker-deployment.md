# Harbor 在 Docker 中的部署

使用 Docker Compose 是测试和体验 Harbor 最简单的方式。

## 前置要求

- 安装了 Docker (推荐版本 17.06.0-ce+)
- 安装了 Docker Compose (推荐版本 1.18.0+)
- 机器至少有 4GB 内存（建议配置）

## 部署步骤

### 1. 下载 Harbor 安装包

访问 Harbor 的 [GitHub Releases](https://github.com/goharbor/harbor/releases) 页面，下载最新的 `offline-installer` (离线安装包) 或 `online-installer` (在线安装包)。

```bash
# 示例：下载在线安装包
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
tar xzvf harbor-online-installer-v2.10.0.tgz
cd harbor
```

### 2. 配置文件 `harbor.yml`

在解压后的 `harbor` 目录中，有一个 `harbor.yml.tmpl` 模板文件，将其复制为 `harbor.yml`：

```bash
cp harbor.yml.tmpl harbor.yml
```

编辑 `harbor.yml`，修改关键配置：
- **hostname**: 修改为你的机器 IP 或域名 (例如：`reg.yourdomain.com` 或 `192.168.9.5`)。**注意：不能使用 `localhost` 或 `127.0.0.1`**。
- **http**: 如果不配置 HTTPS，确保 http 端口为 80。
- **https**: 如果只是本地测试，可以注释掉 `https` 及其相关配置（port, certificate, private_key），或者自己生成自签名证书并配置路径。
- **harbor_admin_password**: 默认管理员密码 (默认是 `Harbor12345`)。

### 3. 执行安装脚本

在 `harbor` 目录下执行 `install.sh` 脚本：

```bash
# 如果不需要附加组件 (如 Trivy 扫描仪, Notary)
./install.sh

# 如果需要安装 Trivy 扫描组件
./install.sh --with-trivy
```

脚本会自动加载镜像并使用 `docker-compose` 启动所有相关的 Harbor 容器（如 nginx, harbor-core, harbor-db, redis, registry 等）。

### 4. 验证部署

部署成功后，你可以通过浏览器访问你在 `harbor.yml` 中配置的 `hostname`，使用默认账号 `admin` 和你设置的密码（默认 `Harbor12345`）登录 Web 控制台。

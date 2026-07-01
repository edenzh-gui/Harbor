# Docker 与 Harbor 对接

当你拥有了一个 Harbor 仓库后，你通常需要在开发机上通过 Docker 客户端来推送（Push）和拉取（Pull）镜像。

## 1. 配置 Docker 信任不安全的 Registry (HTTP 模式下)

如果你在部署 Harbor 时**没有配置 HTTPS（SSL 证书）**，Docker 客户端默认会拒绝连接不安全的 HTTP registry。你需要修改 Docker 客户端的配置来将其加入白名单。

编辑 `/etc/docker/daemon.json` (Mac 上在 Docker Desktop -> Settings -> Docker Engine 中修改)：

```json
{
  "insecure-registries" : ["192.168.9.5", "reg.yourdomain.com"]
}
```
*将上述地址替换为你实际的 Harbor hostname。*

保存后，**重启 Docker 服务**：
```bash
# Linux
sudo systemctl restart docker
```

## 2. 登录 Harbor

在终端中输入：

```bash
docker login 192.168.9.5
# 或者域名
docker login reg.yourdomain.com
```

根据提示输入用户名（如 `admin` 或你在 Harbor 创建的账号）和密码。登录成功的凭证会保存在 `~/.docker/config.json` 中。

## 3. 为镜像打标签 (Tag)

要将本地镜像推送到 Harbor，镜像的名称必须包含 Harbor 的地址以及具体的项目(Project)名称。
假设 Harbor 中有一个公开或私有的项目叫 `library`。

```bash
# 语法
docker tag SOURCE_IMAGE[:TAG] HARBOR_HOST/PROJECT_NAME/IMAGE_NAME[:TAG]

# 示例：将本地的 nginx:latest 推送到 harbor 的 library 项目下
docker tag nginx:latest 192.168.9.5/library/my-nginx:v1.0
```

## 4. 推送镜像 (Push)

```bash
docker push 192.168.9.5/library/my-nginx:v1.0
```

推送完成后，你可以在 Harbor 的 Web 界面的 `library` 项目下看到这个镜像。

## 5. 拉取镜像 (Pull)

其他机器想要使用这个镜像，只需（可能需要先 login）：

```bash
docker pull 192.168.9.5/library/my-nginx:v1.0
```

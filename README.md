# Harbor & Kubernetes 镜像仓库集成实战项目

本项目旨在记录和演示在自建 Kubernetes 集群（基于 VMware 虚拟机）中，从零开始部署 Harbor 私有镜像仓库，并彻底打通 Kubernetes 节点到 Harbor 镜像拉取全链路的完整实战过程。

## 🎯 项目目标
* 在纯内网/虚拟机环境中，使用 Helm 快速落地 Harbor 服务。
* 掌握 `containerd` 运行时关于非安全 Registry（HTTP 协议）的认证配置方法。
* 熟练使用 `nerdctl` 客户端进行登录和镜像推流。
* 掌握 K8s 中 `ImagePullSecrets`（拉取凭证）的创建原理及声明式应用。

## 💻 基础环境说明
本项目的配置与说明基准建立在以下环境之上，供后续参考或复现使用：
* **基础设施**：3 节点 Kubernetes 集群（VMware 虚拟机）
* **Master 节点 IP**：`192.168.9.41`
* **底层容器运行时**：`containerd`
* **暴露方式**：基于 NodePort 暴露 HTTP 服务（禁用 TLS 证书验证，端口 `30002`）

## 📁 目录结构指南

```text
Harbor/
├── README.md                             <-- 本项目说明文档
├── docs/
│   └── SOP_Harbor_K8s_Integration_CN.md  <-- ⭐ 【核心】保姆级标准操作流程 (SOP) 手册
├── harbor/                               <-- (需手动拉取) 下载到本地的 Harbor 原生 Helm Chart
├── values.yaml                           <-- 修改定制后的 Harbor 安装配置 (覆盖原生配置)
└── k8s/
    ├── harbor-registry-secret.yaml       <-- Kubernetes 拉取密钥声明式 YAML
    └── test-deployment.yaml              <-- 包含 imagePullPolicy 和拉取测试的 Deployment
```

## 🚀 快速开始

所有的实战经验、踩坑记录以及一步步的具体执行命令，都已经详细整理在了 SOP 文档中。

强烈建议您直接打开阅读：
👉 **[SOP_Harbor_K8s_Integration_CN.md](docs/SOP_Harbor_K8s_Integration_CN.md)**

### 核心步骤预览：
1. 为所有节点的 `containerd` 配置信任 `192.168.9.41:30002` 的 HTTP 请求。
2. 安装轻量级客户端 `nerdctl`。
3. 把官方 Harbor Chart `pull` 到本地。
4. 替换/修改 `values.yaml` 中的核心设置（开启 NodePort、关闭 TLS、关闭持久化以防卡壳）。
5. 执行 `helm install` 进行本地部署。
6. 在 Harbor UI 新建私有项目。
7. 使用带 `--insecure-registry` 标识的 nerdctl 完成镜像的 tag 和 push。
8. 在 K8s 侧应用拉取凭证（Secret），并部署业务容器进行拉取测试。

## 💡 踩坑备忘录 (Troubleshooting)
* **拉取/推送时提示 HTTP/HTTPS 协议冲突**：请务必检查 `/etc/containerd/certs.d/` 目录结构及命名是否和请求地址绝对一致，并确保重启了 containerd。使用 nerdctl 操作时记得携带 `--insecure-registry` 参数。
* **推送时提示 `401 Unauthorized`**：不要怀疑密码错误，**极大概率是因为您想 push 的目标项目尚未在 Harbor 网页端手动创建**。
* **Pod 部署不需要 Secret 也能拉取成功**：因为您推到了自带的 `library` 项目，该项目默认是 Public（公开）的，必须建立私有项目才能验证拉取凭证的作用。

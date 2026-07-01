# Harbor 学习指南

本项目旨在帮助你学习和掌握 [Harbor](https://goharbor.io/)，包括其核心概念、如何通过 Docker 和 Kubernetes 进行部署，以及如何在这两个平台中与 Harbor 进行对接。

## 什么是 Harbor？

Harbor 是一个开源的、受信任的云原生镜像注册表（Registry），用于存储、签名和扫描容器镜像。它在开源 Docker Distribution 的基础上增加了企业级功能，如：
- **基于角色的访问控制 (RBAC)**：控制用户对镜像仓库的访问权限。
- **镜像漏洞扫描**：集成 Trivy 或 Clair 对镜像进行安全扫描。
- **镜像签名**：集成 Notary 确保镜像来源的可靠性。
- **跨数据中心复制**：支持多 Harbor 实例间的镜像同步。
- **图形化界面管理**：提供易用的 Web UI。

## 目录索引

你可以按照以下顺序阅读本项目的详细文档：

1. [Harbor 在 Docker 中的部署](./01-docker-deployment.md) - 学习如何使用 docker-compose 快速拉起一个 Harbor 实例。
2. [Docker 与 Harbor 对接](./02-docker-integration.md) - 学习 Docker 客户端如何登录、推送和拉取镜像。
3. [Harbor 在 Kubernetes 中的部署](./03-k8s-deployment.md) - 学习如何使用 Helm 将 Harbor 部署到 K8s 集群中。
4. [Kubernetes 与 Harbor 对接](./04-k8s-integration.md) - 学习 K8s 怎么配置凭证（imagePullSecrets）来拉取 Harbor 中的私有镜像。

---
开始你的云原生镜像管理之旅吧！

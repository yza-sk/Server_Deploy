# Deploy 所需变量清单

编写 deploy 脚本与配置时，需严格按照以下变量名命名。分为 **GitHub Actions**、**Jenkins** 两部分。

## GitHub Actions 环境变量/Secrets

| 变量名 | 类型 | 说明 | 用途 | 示例 |
|--------|------|------|------|------|
| `ACR_REGISTRY_PUBLIC` | Secret | ACR 仓库公网地址 | GitHub Actions 推送镜像到公网 ACR | `registry.cn-hangzhou.aliyuncs.com` |
| `ACR_USERNAME` | Secret | ACR 登录用户名 | 登录 ACR 仓库 | `your-acr-username` |
| `ACR_PASSWORD` | Secret | ACR 登录密码 | 登录 ACR 仓库 | `your-acr-password` |
| `ACR_NAMESPACE` | Secret | ACR 命名空间 | 构造 ACR 镜像完整名 | `your-namespace` |
| `SRC_REGISTRY_USERNAME` | Secret | 源镜像仓库用户名（可选） | 若源镜像需认证，使用此凭据登录 | `docker-hub-user`（如需） |
| `SRC_REGISTRY_PASSWORD` | Secret | 源镜像仓库密码（可选） | 若源镜像需认证，使用此凭据登录 | 同上 |
| `JENKINS_WEBHOOK_URL` | Secret | Jenkins Webhook 地址 | GitHub Actions 完成后通知 Jenkins | `http://jenkins-server:8080/generic-webhook-trigger/invoke?token=your-token` |
| `JENKINS_TOKEN` | Secret | Jenkins Hook Token | Webhook 认证 | `your-jenkins-token` |

## Jenkins 环境/凭据

| 变量/凭据名 | 类型 | 说明 | 用途 | 示例 |
|------------|------|------|------|------|
| `ACR_REGISTRY_INTRANET` | 环境变量 | ACR 仓库内网地址（可选） | Jenkins 从内网拉取 ACR 镜像（加速） | `registry-internal.cn-hangzhou.aliyuncs.com`（若有） |
| `ACR_REGISTRY_PUBLIC` | 环境变量 | ACR 仓库公网地址 | Jenkins 从公网拉取 ACR 镜像（备选） | `registry.cn-hangzhou.aliyuncs.com` |
| `ACR_USERNAME` | 凭据（Credential） | ACR 登录用户名 | `withCredentials` 中使用 | `your-acr-username` |
| `ACR_PASSWORD` | 凭据（Credential） | ACR 登录密码 | `withCredentials` 中使用 | `your-acr-password` |
| `ACR_NAMESPACE` | 环境变量 | ACR 命名空间 | 构造 ACR 镜像完整名 | `your-namespace` |
| `DOCKER_CONFIG_PATH` | 环境变量 | Docker 配置文件路径（可选） | 若使用 `.docker/config.json` 认证 | `/root/.docker/config.json` |

## 变量关键说明

### 1. ACR_REGISTRY_PUBLIC vs ACR_REGISTRY_INTRANET
- **Public**：公网地址，GitHub Actions 必须使用；Jenkins 在无内网的情况下也用这个。
- **Intranet**（可选）：内网地址，若 Jenkins 服务器与 ACR 同局域网，使用内网加速拉取。

### 2. ACR_USERNAME / ACR_PASSWORD
- GitHub Actions & Jenkins 都需要相同凭据登录 ACR。
- 可用 Access Key ID / Access Key Secret（推荐），或固定用户密码。

### 3. ACR_NAMESPACE
- 完整镜像名格式：`<ACR_REGISTRY>/<ACR_NAMESPACE>/<image-name>:<tag>`
- 示例：`registry.cn-hangzhou.aliyuncs.com/your-namespace/nginx:1.25`

### 4. SRC_REGISTRY_USERNAME / SRC_REGISTRY_PASSWORD
- 仅当源镜像（如 Docker Hub 私有仓库）需认证时才需要配置。
- 否则置空或不使用。

### 5. JENKINS_WEBHOOK_URL / JENKINS_TOKEN
- GitHub Actions 在推送 ACR 后通过 Webhook 通知 Jenkins 触发拉取。
- Token 需与 Jenkins Job 的 `Generic Webhook Trigger` 插件配置一致。

## 配置检查清单

- [ ] GitHub Secrets 中配置了 `ACR_REGISTRY_PUBLIC`, `ACR_USERNAME`, `ACR_PASSWORD`, `ACR_NAMESPACE`。
- [ ] GitHub Secrets 中配置了 `JENKINS_WEBHOOK_URL`, `JENKINS_TOKEN`（若需要完整链路）。
- [ ] GitHub Secrets 中配置了 `SRC_REGISTRY_USERNAME`, `SRC_REGISTRY_PASSWORD`（如源镜像需认证）。
- [ ] Jenkins 服务器环境变量或 Job 凭据中配置了 `ACR_REGISTRY_INTRANET`（可选）、`ACR_REGISTRY_PUBLIC`、`ACR_USERNAME`、`ACR_PASSWORD`、`ACR_NAMESPACE`。
- [ ] Jenkins Job 的 `Generic Webhook Trigger` 插件配置的 Token 与 `JENKINS_TOKEN` Secret 一致。

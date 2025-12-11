# Server Deploy (GitHub Actions + Jenkins + ACR)

一个最小化的流程设计，解决：GitHub 仓库推送后自动从源仓库拉取镜像、改名并推送到阿里云 ACR；Jenkins 通过 Hook 拉取 ACR 镜像、恢复原名以兼容 `docker-compose`，最后由人工部署。

## 目标
- 统一镜像上传/拉取入口，减少服务器拉取失败。 
- Git push 触发 GitHub Actions，完成镜像重命名与推送到个人 ACR。 
- Jenkins Webhook 触发，从 ACR 拉取镜像到服务器并改回原名，保证现有 `docker-compose` 直接可用。

## 期望目录结构（示例）
```
Server_Deploy/
├─ README.md               # 本文档
├─ deploy/                 # 预期未来放置脚本/配置
│  └─ docker-compose.yml   # 服务器部署用 compose，直接作为镜像清单来源
└─ .github/workflows/ci.yml# GitHub Actions 工作流
```

## 核心流程（端到端）
1) 开发者 `git push` 到 GitHub。 
2) GitHub Actions 读取 `deploy/docker-compose.yml`：
  - 解析 `services[*].image` 列表（借助 `yq`/`docker compose config`）。
  - 登录源镜像 Registry（如 Docker Hub/私服）。
  - 对每个镜像：`docker pull` 源镜像，改名为 ACR 镜像（追加 ACR 前缀），`docker push` 到 ACR。 
3) GitHub Actions 完成后，向 Jenkins 发送 Webhook（含镜像列表或 Git commit）。
4) Jenkins Job 被触发：
   - 登录 ACR；
   - 拉取 ACR 镜像并改回原名，供 `docker-compose` 使用；
   - （可选）写入本地镜像缓存或私有 Registry。
5) 你在服务器上手动运行/更新 `docker-compose up -d`。

## GitHub Actions 思路（`/.github/workflows/ci.yml`）
- 触发：
  - 现阶段：`push` 到 `main` 触发“完整链路”（推 ACR + 通知 Jenkins）。
  - 计划中：`push` 到 `onlypush` 仅执行“推送到 ACR”，不触发 Jenkins 拉取/回标。
- 步骤要点：
  1. 检出代码。
  2. 设置 Docker Buildx（可选）。
  3. 登录源 Registry（如需）。
  4. 登录阿里云 ACR（`ACR_USERNAME`/`ACR_PASSWORD`/`ACR_REGISTRY`）。
  5. 解析 `deploy/docker-compose.yml` 获取镜像列表：
    - `docker compose config | yq '.services.*.image' | sort -u`，或使用 `yq '.services[].image' deploy/docker-compose.yml`。
  6. 对每个镜像：`docker pull` 源镜像 → `docker tag` 为 ACR 镜像（`$ACR_REGISTRY/<namespace>/<name>:tag`）→ `docker push` 到 ACR。
  7. 针对 `main`：发送 Webhook 给 Jenkins（使用 `curl`，带 token 与镜像列表或 commit SHA）；针对 `onlypush`：跳过通知。
- 建议 Secrets：`ACR_USERNAME`, `ACR_PASSWORD`, `ACR_REGISTRY`, `SRC_REGISTRY_USERNAME`/`SRC_REGISTRY_PASSWORD`（如有私有源），`JENKINS_WEBHOOK_URL`, `JENKINS_TOKEN`。

## Jenkins 侧建议
- 类型：Pipeline (Declarative)。
- 触发：GitHub Actions 发送的 Webhook，附带 commit 或镜像列表。
- 步骤示例：
  1. 解析 Webhook 载荷，得到要处理的镜像列表（来自 compose 解析结果或 GitHub Actions 传递的 ACR 名）。
  2. `docker login` 到 ACR（使用 Jenkins Credential）。
  3. 对每个镜像：`docker pull <acr-image>`，`docker tag <acr-image> <source-name>`（source-name 对应 compose 中的原始 `image`），可选 `docker rmi <acr-image>` 释放空间。
  4. （可选）触发/提示 `docker-compose up -d`。
- Jenkins 凭据：`ACR_CREDENTIAL`（用户名/密码或 AK/SK），可用凭据绑定到 `withCredentials`。

## 命名与 Tag 约定
- 尽量保持与 `docker-compose.yml` 中一致的 tag，减少额外映射。
- 若需要加速，可统一使用 `:latest` 或日期 tag，但要保证 compose 文件与实际镜像同步。

## 变量配置
详见 `VARIABLES.md`。包含 GitHub Actions Secrets、Jenkins 凭据与环境变量的完整列表、说明与示例。

## TODO / 下一步
- [ ] 添加 `deploy/docker-compose.yml` 占位文件（仅作镜像清单来源）。
- [ ] 编写 `.github/workflows/ci.yml`，完成拉取/改名/推送/通知逻辑。
- [ ] 编写 Jenkins Pipeline 示例（接受 Webhook，拉取 ACR，回标 tag）。
- [ ] 严格按 `VARIABLES.md` 配置所有变量，确保脚本正确识别。

## 使用步骤（最小可行版）
1. 创建 GitHub 仓库并推送本目录。 
2. 在 GitHub Secrets 配置 ACR/Jenkins/源 Registry 凭据。 
3. 填写 `deploy/docker-compose.yml`（至少包含 services.image）。 
4. 推送代码：
   - 到 `main`：执行完整链路（推到 ACR + 通知 Jenkins）。
   - 到 `onlypush`（计划中）：仅推到 ACR，不通知 Jenkins。
5. 若走完整链路：Jenkins Webhook 被触发，自动拉取并回标镜像。 
6. 服务器执行/更新 `docker-compose up -d` 完成部署。

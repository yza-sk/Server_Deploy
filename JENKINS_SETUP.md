# Jenkins 配置指南

## 前提条件

- Jenkins 已安装且正在运行
- 服务器上安装了 Docker 和 docker-compose
- 已获取 ACR 登录凭据（用户名、密码或 Access Key）

---

## 第一步：安装必需的 Jenkins 插件

### 1.1 打开 Jenkins 管理页面
1. 访问 Jenkins 控制台：`http://jenkins-server:8080`（根据实际地址修改）
2. 点击左侧 **Manage Jenkins** → **Manage Plugins**（或 **Plugins**）

### 1.2 需要安装的插件
在 **Available plugins** 标签页搜索并安装以下插件：
- **Generic Webhook Trigger** - 接收 GitHub Actions 的 Webhook 回调（**必需**）
- **Docker Pipeline** - Docker 支持（可选，如果需要在 Jenkins 中构建镜像）
- **Pipeline** - Pipeline 支持（应该已默认安装）
- **Credentials Binding Plugin** - 凭据绑定（应该已默认安装）

**安装步骤**：
1. 在搜索框中输入插件名：`Generic Webhook Trigger`
2. 勾选插件
3. 点击 **Install without restart** 或 **Install**
4. 等待安装完成

**验证安装**：
安装完成后，创建或编辑 Pipeline Job，在 **Build Triggers** 标签下应该能看到 **Generic Webhook Trigger** 选项。如果没有，尝试重启 Jenkins。

### 1.3 如果找不到 Generic Webhook Trigger 插件

**检查步骤**：
1. 点击 **Manage Jenkins** → **Plugins** → **Installed plugins**
2. 搜索 `Generic Webhook Trigger`
3. 如果已安装，确认状态是否为"已启用"

**手动安装（如果插件市场中没有）**：
1. 访问插件页面：https://plugins.jenkins.io/generic-webhook-trigger/
2. 下载 `.hpi` 文件
3. 在 Jenkins 中：**Manage Jenkins** → **Plugins** → **Advanced settings**
4. 在 **Deploy Plugin** 部分上传 `.hpi` 文件
5. 重启 Jenkins

---

## 第二步：配置 ACR 凭据

### 2.1 进入凭据管理
1. 点击左侧 **Manage Jenkins** → **Manage Credentials**
2. 点击 **System** → **Global credentials (unrestricted)**
3. 点击左侧 **Add Credentials**

### 2.2 创建 ACR 凭据
**填写以下信息**：
- **Kind**: 选择 `Username with password`
- **Scope**: 选择 `Global`
- **Username**: 填入 `ACR_USERNAME`（即你的 ACR 登录用户名）
- **Password**: 填入 `ACR_PASSWORD`（即你的 ACR 登录密码）
- **ID**: 输入 `acr-credential`（**重要**：与 Jenkinsfile 中的 `credentials('acr-credential')` 对应）
- **Description**: 输入 `ACR Registry Credential`

3. 点击 **Create** 保存

---

## 第三步：在 Jenkins 中创建新 Job（Pipeline）

### 3.1 创建 Pipeline Job
1. 点击 Jenkins 首页左侧 **New Item** 或 **创建任务**
2. 输入 Job 名称，例如：`acr-image-pull`
3. 选择 **Pipeline**（下拉选项）
4. 点击 **OK**

### 3.2 配置 General 标签
1. 勾选 **This project is parameterized**
2. 点击 **Add Parameter** 添加三个参数（与 Jenkinsfile 中的 `parameters` 块对应）：

**参数 1 - COMMIT_SHA**:
- Name: `COMMIT_SHA`
- Default Value: `` （留空）
- Description: `Git commit SHA from webhook`

**参数 2 - ACR_NAMESPACE**:
- Name: `ACR_NAMESPACE`
- Default Value: `` （留空）
- Description: `ACR namespace`

**参数 3 - IMAGE_LIST_JSON**:
- Name: `IMAGE_LIST_JSON`
- Default Value: `[]`
- Description: `JSON array of source images`

### 3.3 配置 Build Triggers 标签（启用 Webhook）

**重要提示**：如果你没有看到 **Generic Webhook Trigger** 选项，说明插件未正确安装，请先返回第一步确认插件安装。

#### 方法一：使用 Generic Webhook Trigger（推荐）

1. 点击 **Build Triggers** 标签
2. 勾选 **Generic Webhook Trigger**
3. 配置以下内容：

**Token 配置**：
- 在 **Token** 字段输入一个 Token（例如：`your-jenkins-webhook-token`）
- **重要**：这个 Token 必须与 GitHub Secrets 中的 `JENKINS_TOKEN` 一致

**Post content parameters（参数映射）**：
点击 **Add** 添加以下参数，从 Webhook JSON 中提取数据：

| Variable | Expression | JSONPath | Default Value |
|----------|------------|----------|---------------|
| `COMMIT_SHA` | `$.commit_sha` | JSONPath | (空) |
| `ACR_NAMESPACE` | `$.acr_namespace` | JSONPath | (空) |
| `IMAGE_LIST_JSON` | `$.images` | JSONPath | `[]` |

**配置步骤**：
- 点击 **Post content parameters** 下的 **Add**
- **Variable**: 输入 `COMMIT_SHA`
- **Expression**: 输入 `$.commit_sha`
- **JSONPath**: 勾选
- 重复以上步骤添加其他两个参数

**Optional filter（可选）**：
如果需要只响应特定条件的 Webhook，可以配置过滤器，否则留空。

### 3.4 配置 Pipeline 标签
1. 点击 **Pipeline** 标签
2. 在 **Definition** 下拉中选择 **Pipeline script from SCM**
   
   **或者** 选择 **Pipeline script** 并直接粘贴 Jenkinsfile 内容

**如果选择 "Pipeline script from SCM"**：
- **SCM**: 选择 `Git`
- **Repository URL**: 输入你的 GitHub 仓库 URL（`https://github.com/your-username/Server_Deploy.git`）
- **Credentials**: 如果是私有仓库，需要添加 GitHub 凭据
- **Branch Specifier**: 输入 `*/main` 或 `*/onlypush`（根据需要）
- **Script Path**: 输入 `Jenkinsfile`（默认）
- 点击 **Save**

**或者如果选择 "Pipeline script"**：
- 复制 Jenkinsfile 的全部内容到文本框
- 点击 **Save**

---

## 第四步：配置环境变量

### 4.1 在 Jenkins 全局配置中设置环境变量
1. 点击 **Manage Jenkins** → **Configure System**
2. 滚动到 **Global properties** 部分
3. 勾选 **Environment variables**
4. 点击 **Add** 按钮，添加以下环境变量：

| 变量名 | 值 | 说明 |
|--------|-----|------|
| `ACR_REGISTRY_PUBLIC` | `registry.cn-hangzhou.aliyuncs.com` | ACR 公网地址 |
| `ACR_REGISTRY_INTRANET` | `registry-internal.cn-hangzhou.aliyuncs.com` | ACR 内网地址（如有，否则留空） |
| `ACR_NAMESPACE` | `your-namespace` | ACR 命名空间 |

5. 点击 **Save** 保存

**备注**：也可以在 Job 级别配置这些变量，步骤如下：
1. 打开 Job 配置页面
2. 点击 **Build Environment** 标签
3. 勾选 **Add timestamps to the Console Output**（可选）
4. 或者在 Pipeline 脚本中直接定义 environment 块（已在 Jenkinsfile 中定义）

---

## 第五步：测试 Pipeline

### 5.1 手动触发 Pipeline（测试）
1. 打开 Job 页面
2. 点击左侧 **Build with Parameters**
3. 填入测试参数：
   - `COMMIT_SHA`: `abc123def456` （任意值）
   - `ACR_NAMESPACE`: `your-namespace`
   - `IMAGE_LIST_JSON`: `["nginx:1.25-alpine", "postgres:15-alpine"]`
4. 点击 **Build**
5. 在 **Console Output** 中观察执行过程

### 5.2 查看日志
1. 点击 Build 旁边的 **Console Output**
2. 检查是否有错误信息

---

## 第六步：完整工作流测试

### 测试流程：
1. **修改代码** - 在本地修改 `deploy/docker-compose.yml`（例如改个镜像 tag）
2. **推送到 main** - `git push origin main`
3. **GitHub Actions 触发** - 在 GitHub 仓库的 **Actions** 标签页观察 Workflow 执行
4. **Jenkins 被触发** - GitHub Actions 完成后，应该自动触发 Jenkins Job
5. **检查结果** - 在 Jenkins 控制台查看镜像是否成功拉取并改名

---

## 重要说明

**GitHub 仓库的 Settings → Webhooks 中不需要配置任何 Webhook！**

原因是：
- GitHub Actions 通过 `on: push` 自动触发 ci.yml
- ci.yml 中的 `curl` 命令**手动调用** Jenkins 的 Generic Webhook Trigger
- 不是 GitHub 主动推送事件到 Jenkins

所以 Webhook 完全由 GitHub Actions 中的 ci.yml 脚本控制。

---

## 常见问题排查

### Q1: Jenkins 未收到 GitHub Actions 的回调
**原因**：
- Token 不匹配（Jenkins 配置的 Token ≠ GitHub Secrets 的 `JENKINS_TOKEN`）
- `JENKINS_WEBHOOK_URL` 在 Secrets 中配置错误
- Jenkins 防火墙/安全组未开放，GitHub Actions 无法访问

**解决**：
- 确认 Token 与 Generic Webhook Trigger 配置一致
- 检查 `JENKINS_WEBHOOK_URL` 是否为 `http://jenkins-server:8080/generic-webhook-trigger/invoke?token=your-token`
- 在 GitHub Actions 日志中查看 `curl` 请求的响应
- 在 Jenkins 日志中搜索 `WebhookTrigger` 相关信息

### Q2: ACR 登录失败
**原因**：
- 凭据配置错误
- ACR 用户名/密码不正确

**解决**：
- 在本地手动测试：`docker login -u username -p password registry.cn-hangzhou.aliyuncs.com`
- 重新检查 Jenkins 凭据配置

### Q3: 镜像拉取失败
**原因**：
- ACR_NAMESPACE 未正确配置
- GitHub Actions 未成功推送镜像到 ACR

**解决**：
- 检查 GitHub Actions 执行日志
- 确认镜像是否真的在 ACR 中：`docker search <acr-registry>/<namespace>/image-name`

### Q4: 镜像改名后仍然是 ACR 名称
**原因**：
- Jenkinsfile 中的镜像名提取逻辑有问题
- IMAGE_LIST_JSON 格式不正确

**解决**：
- 在 Jenkins Console 中打印 IMAGE_LIST_JSON 和提取的镜像名
- 确保格式是标准 JSON 数组，如 `["nginx:1.25-alpine"]`

---

## 总结

配置顺序总结：
1. ✅ 安装 Generic Webhook Trigger 等必需插件
2. ✅ 创建 ACR 凭据（ID: `acr-credential`）
3. ✅ 创建 Pipeline Job 并配置参数
4. ✅ 启用 Generic Webhook Trigger（设置 Token）
5. ✅ 配置 Pipeline 脚本（Jenkinsfile）
6. ✅ 设置全局环境变量（ACR_REGISTRY_PUBLIC 等）
7. ✅ 在 GitHub Secrets 中配置 `JENKINS_WEBHOOK_URL` 和 `JENKINS_TOKEN`
8. ✅ 测试整个工作流

完成上述步骤后，整个 CI/CD 链路就应该完全工作了！

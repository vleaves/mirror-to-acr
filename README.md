# Docker Hub 镜像自动同步到阿里云 ACR

通过 GitHub Actions 定时检查 Docker Hub 镜像更新，并将更新后的镜像自动同步到阿里云容器镜像服务（ACR）。

本项目适用于运行环境无法直接访问 Docker Hub，但能够访问阿里云 ACR 的场景。

## 功能特性

- **定时检查**：每天北京时间 10:00 自动检查镜像更新。
- **增量同步**：通过 Digest 判断镜像是否变化，未更新则跳过复制。
- **多架构支持**：使用 Skopeo 的 `--all` 参数复制多架构镜像。
- **状态持久化**：将已成功同步的源镜像 Digest 保存到 Git 仓库。
- **企业微信通知**：同步完成后发送已更新、无需更新和失败的镜像列表。
- **失败重试**：同步失败时不更新对应镜像的历史 Digest，后续运行可再次尝试。
- **手动触发**：支持在 GitHub Actions 页面手动执行同步任务。
- **自动记录状态**：同步成功后自动提交更新后的 `state/digests.json`。

## 工作原理

```text
Docker Hub
    │
    ▼
GitHub Actions 定时运行
    │
    ▼
读取 images.txt
    │
    ▼
检查源镜像 Digest
    │
    ├── Digest 未变化 ──► 跳过同步
    │
    └── Digest 已变化或首次同步
              │
              ▼
        复制镜像到阿里云 ACR
              │
              ▼
        更新 state/digests.json
              │
              ▼
        提交状态文件到 GitHub
              │
              ▼
        发送企业微信通知
```

## 项目结构

```text
.
├── images.txt
├── state/
│   └── digests.json
├── .github/
│   └── workflows/
│       └── sync-images.yml
└── README.md
```

| 文件 | 说明 |
|---|---|
| `images.txt` | 需要同步的 Docker Hub 镜像列表 |
| `state/digests.json` | 保存源镜像最近一次成功同步的 Digest |
| `.github/workflows/sync-images.yml` | GitHub Actions 自动同步工作流 |
| `README.md` | 项目说明和使用文档 |

## 一、准备阿里云 ACR

在阿里云容器镜像服务控制台：

1. 创建或选择可用的 ACR 实例。
2. 创建目标命名空间。
3. 获取实例对应的 Registry 地址。
4. 准备具有目标命名空间推送权限的账号和密码。

例如：

```text
Registry：registry.cn-hangzhou.aliyuncs.com
命名空间：my-namespace
```

本项目采用扁平化目标路径，只保留 Docker Hub 源镜像路径的最后一段作为 ACR 镜像名。

例如：

| Docker Hub 源镜像 | ACR 目标镜像 |
|---|---|
| `nginx:latest` | `my-namespace/nginx:latest` |
| `redis:latest` | `my-namespace/redis:latest` |
| `grafana/grafana:latest` | `my-namespace/grafana:latest` |
| `prom/prometheus:v3.0.0` | `my-namespace/prometheus:v3.0.0` |

完整地址示例：

```text
registry.cn-hangzhou.aliyuncs.com/my-namespace/grafana:latest
```

请确认 ACR 账号具有推送权限。能够登录 ACR 并不代表一定可以上传镜像。

## 二、配置 GitHub Secrets

进入 GitHub 仓库：

`Settings → Secrets and variables → Actions → New repository secret`

添加以下 Repository secrets：

| Secret 名称 | 说明 |
|---|---|
| `ACR_REGISTRY` | ACR Registry 地址，不带 `https://` |
| `ACR_NAMESPACE` | ACR 命名空间 |
| `ACR_USERNAME` | ACR 登录用户名 |
| `ACR_PASSWORD` | ACR 登录密码或相应访问凭证 |
| `WECHAT_WEBHOOK_URL` | 企业微信群机器人 Webhook 地址 |

请勿将真实账号密码、访问令牌或 Webhook 地址直接提交到仓库。

### 配置 GitHub Actions 写入权限

进入：

`Settings → Actions → General → Workflow permissions`

确保 GitHub Actions 具有写入仓库内容的权限，以便自动提交 `state/digests.json`。

## 三、配置镜像列表

编辑仓库根目录的 `images.txt`，每行填写一个 Docker Hub 镜像。

示例：

```text
# Docker Hub 官方镜像
nginx:latest
redis:latest
mysql:8.4

# 第三方镜像
grafana/grafana:latest
prom/prometheus:latest
```

配置规则：

- 每行填写一个镜像。
- 支持使用 `#` 添加注释。
- 空行会被忽略。
- 不填写 Tag 时，默认使用 `latest`。
- 当前工作流不支持通过 Digest 固定版本的镜像引用。
- 目标镜像名采用扁平化规则，需避免不同源镜像映射到同一个目标名称。

**注意：** 例如 `user1/app:latest` 和 `user2/app:latest` 都会映射到 `app:latest`。如果存在这种情况，工作流会检测到目标名称冲突并记录失败。

## 四、运行方式

### 1. 定时运行

工作流默认每天北京时间 10:00 执行一次，对应 UTC 时间 02:00。

```yaml
schedule:
  - cron: "0 2 * * *"
```

GitHub Actions 的定时任务可能因平台负载而延迟执行。

### 2. 修改配置后自动运行

当 `main` 分支中的以下文件发生变化并被推送时，会触发工作流：

- `images.txt`
- `.github/workflows/sync-images.yml`

新增或删除镜像后，可以通过提交变更立即触发下一次检查。

### 3. 手动运行

1. 打开 GitHub 仓库的 `Actions` 页面。
2. 选择 `Sync Docker Hub Images to Alibaba ACR`。
3. 点击 `Run workflow`。
4. 选择 `main` 分支。
5. 点击运行按钮。

运行日志和任务摘要可以在对应的 workflow run 中查看。

## 五、Digest 状态管理

状态文件：

```text
state/digests.json
```

首次运行时可以初始化为空对象：

```json
{}
```

同步后，状态文件会记录源镜像及其最近一次成功同步的 Digest，例如：

```json
{
  "grafana/grafana:latest": "sha256:example",
  "nginx:latest": "sha256:example"
}
```

以上 Digest 仅用于说明文件格式，并非真实镜像摘要。

工作流的判断规则：

- **首次同步**：没有历史记录，执行同步。
- **镜像未变化**：跳过复制。
- **镜像已更新**：复制到 ACR，成功后保存新 Digest。
- **复制失败**：保留旧 Digest，后续运行再次尝试。

状态文件由 GitHub Actions 自动提交到仓库。

本方案以 Git 仓库中的 Digest 为同步判断依据，不会每天检查 ACR 中的镜像是否被外部删除或修改。因此，请避免手动删除或修改已同步的 ACR 镜像及状态文件。

## 六、企业微信通知

每次工作流执行后，都会尝试通过企业微信群机器人发送文本通知。

### 同步成功示例

```text
已更新镜像：
nginx:latest
grafana/grafana:latest

无需更新镜像：
redis:latest
mysql:8.4
```

### 同步失败示例

```text
镜像同步任务失败，请检查 GitHub Actions 日志。

已更新镜像：
nginx:latest

同步失败：
grafana/grafana:latest - ACR copy failed
```

如果任务在检查镜像之前就失败，通知会提示查看 GitHub Actions 日志。

企业微信通知依赖 `WECHAT_WEBHOOK_URL` 配置正确且机器人接口可用。若通知发送失败，请检查工作流日志和机器人配置。

## 七、常见问题

### 1. `ACR_USERNAME: unbound variable`

**原因：** 当前工作流步骤未获取到 `ACR_USERNAME` 环境变量，或相应 Secret 未配置。

**排查方法：**

- 检查 Secret 名称是否完全一致。
- 检查 Secret 是否配置在正确的仓库或 Environment 中。
- 确认使用变量的步骤在 `env` 中声明了相应映射。

### 2. `requested access to the resource is denied`

**原因：** 可能是 ACR 登录凭证、目标 Registry、命名空间或推送权限不正确。

**排查方法：**

- 确认 `ACR_REGISTRY` 与实际实例地址一致。
- 确认目标命名空间存在。
- 确认账号具有镜像推送权限。
- 确认目标地址符合 `命名空间/镜像名:tag` 的格式。

### 3. `JSONDecodeError`

**原因：** 程序尝试解析空内容或无效 JSON。

**排查方法：**

- 检查 `state/digests.json` 是否为有效 JSON。
- 检查 Skopeo 命令是否成功执行。
- 检查源镜像是否可访问。
- 确保命令失败时不会继续解析无效输出。

### 4. 镜像一直被判定为更新

**可能原因：**

- 状态文件没有成功提交到 GitHub。
- 源镜像的 Manifest 内容发生变化。
- 当前 Digest 计算方式与 Registry 返回的规范 Digest 不一致。

**排查方法：**

查看 GitHub Actions 日志，确认状态文件已成功提交。严格的 Digest 比较建议使用 Registry 返回的规范 Digest。

### 5. 镜像复制失败，但 ACR 登录成功

**原因：** 登录成功不代表具有目标仓库的写入权限。

**解决方法：**

检查 ACR 账号授权、命名空间权限、实例访问控制，以及目标镜像地址是否正确。

### 6. 为什么未更新的镜像不会重新推送？

工作流使用上一次成功同步的源镜像 Digest 判断镜像是否变化。如果 Digest 相同，就跳过复制。

这可以减少不必要的传输，但前提是 ACR 中已有镜像仍然存在且未被外部修改。

## 八、安全建议

- 将凭证保存在 GitHub Secrets 中。
- 不要将企业微信群机器人 Webhook 提交到公开仓库。
- 使用权限范围尽可能小的 ACR 账号或凭证。
- 避免在日志中打印密码或完整 Webhook 地址。
- 定期检查 GitHub Actions 运行记录和失败通知。

## 九、AI 辅助开发

本项目在开发过程中使用了 **ChatGPT（OpenAI）** 作为 AI 辅助工具，协助完成方案设计、工作流编写、错误排查及文档整理。

主要包括：

- **方案设计**：设计基于 GitHub Actions 的 Docker Hub 镜像定时同步方案。
- **工作流编写**：协助编写和调整 GitHub Actions、Skopeo 镜像复制及自动化执行逻辑。
- **ACR 适配**：根据阿里云 ACR 的镜像命名规则调整目标路径。
- **状态管理**：设计基于 `state/digests.json` 的源镜像 Digest 记录机制。
- **消息通知**：设计企业微信机器人文本通知，展示更新、未更新和失败的镜像。
- **问题排查**：协助分析 GitHub Actions 日志，定位环境变量、镜像推送权限和 JSON 解析等问题。
- **文档编写**：协助整理项目说明、配置教程及常见问题。

ChatGPT 提供开发建议和代码参考，最终代码审查、配置验证、运行测试及项目维护由项目维护者负责。AI 生成的代码可能存在错误，使用前应进行检查和测试。



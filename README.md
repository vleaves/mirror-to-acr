# Docker Hub 镜像自动同步到阿里云 ACR

通过 GitHub Actions 定时检查 Docker Hub 镜像更新，并将镜像同步到阿里云容器镜像服务（ACR）。

本项目采用 **Skopeo 优先、Docker pull/push 兜底** 的同步策略，通过 Git 仓库持久化源镜像 Digest，并使用企业微信群机器人发送同步结果。

适用于需要将 Docker Hub 镜像同步到阿里云 ACR，以便在无法直接访问 Docker Hub 的环境中拉取镜像的场景。

## 功能特性

- **定时检查**：每天北京时间 10:00 自动运行。
- **增量同步**：计算源镜像 Manifest 的 SHA-256 Digest，与历史状态比较，未变化则跳过同步。
- **优先使用 Skopeo**：通过 Skopeo 检查和复制镜像，减少对本地 Docker 镜像存储的依赖。
- **Docker 兜底**：Skopeo 复制失败时，尝试通过 Docker pull、tag 和 push 完成同步。
- **架构控制**：Skopeo 复制时指定 `linux/amd64` 覆盖参数；Docker 兜底时使用 `--platform linux/amd64`。
- **状态持久化**：将成功同步的源镜像 Digest 保存到 `state/digests.json`。
- **自动提交状态**：同步完成后，由 GitHub Actions 将更新后的状态文件提交到 `main` 分支。
- **企业微信通知**：任务结束后发送已更新、无需更新和失败的镜像列表。
- **失败重试**：同步失败时不更新对应镜像的历史 Digest，后续运行时可再次尝试。
- **手动触发**：支持在 GitHub Actions 页面手动运行。
- **目标冲突检测**：检查扁平化镜像名称是否发生冲突，避免同一轮任务中不同源镜像覆盖同一个目标。

## 一、工作原理

```text
Docker Hub
    │
    ▼
GitHub Actions 定时触发或手动触发
    │
    ▼
读取 images.txt
    │
    ▼
校验镜像名称并生成 ACR 目标地址
    │
    ▼
Skopeo inspect --raw 获取源 Manifest
    │
    ▼
计算源 Manifest 的 SHA-256 Digest
    │
    ├── Digest 未变化 ──► 跳过同步
    │
    └── Digest 已变化或首次同步
                  │
                  ▼
            尝试 Skopeo copy
                  │
           ┌──────┴──────┐
           │             │
           ▼             ▼
         成功           失败
           │             │
           │             ▼
           │       Docker 登录 ACR
           │             │
           │             ▼
           │       Docker pull
           │             │
           │             ▼
           │       Docker tag
           │             │
           │             ▼
           │       Docker push
           │             │
           └──────┬──────┘
                  │
                  ▼
          保存源镜像 Digest
                  │
                  ▼
          记录同步结果与摘要
                  │
                  ▼
          提交状态文件到 GitHub
                  │
                  ▼
          发送企业微信通知
```

**注意：** 上图描述的是预期的正常同步路径。当前工作流在 `skopeo inspect --raw` 失败时会直接将镜像记为失败并跳过，不会进入 Docker 兜底流程。Docker 兜底只处理后续 `skopeo copy` 失败的情况。

## 二、项目结构

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
| `.github/workflows/sync-images.yml` | GitHub Actions 工作流 |
| `README.md` | 项目说明和使用文档 |

## 三、准备阿里云 ACR

在阿里云容器镜像服务控制台完成以下操作：

1. 创建或选择可用的 ACR 实例。
2. 创建目标命名空间。
3. 获取实例对应的 Registry 地址。
4. 创建具有目标命名空间推送权限的账号或访问凭证。

例如：

```text
Registry：registry.cn-hangzhou.aliyuncs.com
命名空间：my-namespace
```

实际配置应使用你的 ACR 实例地址和命名空间。

### 目标镜像命名规则

当前工作流采用扁平化命名：**只保留源镜像路径最后一段作为目标镜像名称**。

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

如果不同源镜像具有相同的最后一级名称，例如：

```text
user1/app:latest
user2/app:latest
```

它们都会映射到：

```text
registry.cn-hangzhou.aliyuncs.com/my-namespace/app:latest
```

当前工作流会检测同一轮运行中的此类冲突，并将冲突镜像记为失败。

## 四、配置 GitHub Secrets

进入 GitHub 仓库：

`Settings → Secrets and variables → Actions → New repository secret`

添加以下 Repository secrets：

| Secret 名称 | 说明 |
|---|---|
| `ACR_REGISTRY` | ACR Registry 地址，不带 `https://` 或 `http://` |
| `ACR_NAMESPACE` | ACR 目标命名空间 |
| `ACR_USERNAME` | ACR 登录用户名 |
| `ACR_PASSWORD` | ACR 登录密码或相应访问凭证 |
| `WECHAT_WEBHOOK_URL` | 企业微信群机器人 Webhook 地址 |

请勿将真实账号密码、访问令牌或 Webhook 地址直接提交到仓库。

### GitHub Actions 写入权限

进入：

`Settings → Actions → General → Workflow permissions`

确保工作流具有向仓库写入内容的权限，以便提交更新后的 `state/digests.json`。

如果仓库启用了分支保护规则，还需要确认 `main` 分支允许该工作流使用 `GITHUB_TOKEN` 推送，或另行配置具有相应权限的凭证。

### Docker 登录说明

当前工作流在开始同步前使用 Skopeo 登录 ACR，并将认证信息保存到临时认证文件。

Skopeo 使用的认证文件不会自动成为 Docker CLI 的登录凭据。因此，Skopeo 复制失败时，工作流会再次执行 `docker login` 登录 ACR。

当前 Docker 兜底逻辑没有单独执行 Docker Hub 登录。对于需要认证的 Docker Hub 私有镜像，需要额外配置源仓库凭证，并在 Docker pull 前执行相应登录。

## 五、配置镜像列表

编辑仓库根目录的 `images.txt`，每行填写一个 Docker Hub 镜像引用。

例如：

```text
# Docker Hub 官方镜像
nginx:latest
redis:latest
mysql:8.4

# 第三方镜像
grafana/grafana:latest
prom/prometheus:latest
```

### 支持的格式

当前工作流接受以下形式：

```text
image
image:tag
namespace/image
namespace/image:tag
```

规则如下：

- 每行填写一个镜像。
- 支持使用 `#` 添加注释。
- 空行会被忽略。
- 不填写 Tag 时，默认使用 `latest`。
- 不支持通过 `@sha256:...` 固定 Digest 的引用。
- 当前校验逻辑只接受单段或双段的 Docker Hub 镜像路径。
- 不支持在源镜像引用中直接指定其他 Registry。
- 目标镜像采用扁平化命名，需避免不同源镜像映射到同一个目标。

## 六、运行方式

### 1. 定时运行

工作流配置为每天 UTC 时间 02:00 执行，对应北京时间 10:00。

```yaml
schedule:
  - cron: "0 2 * * *"
```

GitHub Actions 定时任务可能因平台负载而延迟执行，实际开始时间不保证精确到分钟。

### 2. 修改镜像列表后自动运行

当 `main` 分支中的以下文件发生变化并被推送时，会触发工作流：

```text
images.txt
.github/workflows/sync-images.yml
```

修改其他文件不会触发当前配置中的 `push` 事件。

### 3. 手动运行

1. 打开 GitHub 仓库的 `Actions` 页面。
2. 选择 `Sync Docker Hub Images to Alibaba ACR`。
3. 点击 `Run workflow`。
4. 选择 `main` 分支。
5. 点击运行按钮。

可以在对应的 workflow run 中查看运行日志和任务摘要。

## 七、Digest 状态管理

状态文件：

```text
state/digests.json
```

首次运行前，可以将文件初始化为：

```json
{}
```

同步后，文件会保存源镜像名称与对应的 Digest：

```json
{
  "grafana/grafana:latest": "sha256:example",
  "nginx:latest": "sha256:example"
}
```

以上 Digest 仅用于说明文件格式，不是真实镜像摘要。

### 判断规则

| 场景 | 工作流行为 |
|---|---|
| 首次同步，没有历史记录 | 尝试复制镜像 |
| 源 Manifest Digest 未变化 | 跳过复制 |
| 源 Manifest Digest 已变化 | 尝试复制镜像 |
| Skopeo 复制失败，Docker 兜底成功 | 保存新的源 Digest |
| Skopeo 和 Docker 复制均失败 | 不更新该镜像的历史 Digest |
| Digest 保存失败 | 记录失败，不将镜像计入成功更新列表 |

工作流通过 `skopeo inspect --raw` 获取源镜像的原始顶层 Manifest 内容，再计算其 SHA-256。

该值是对获取到的 Manifest 字节计算出的摘要。对于多架构镜像，顶层 Manifest 可能是 Manifest List 或 OCI Index；当前代码没有单独从 Registry 的响应头读取规范 Digest。

状态文件由 GitHub Actions 自动提交到 `main` 分支。若提交失败，工作流会记录失败并以非零状态结束。

### 状态管理的限制

当前工作流只通过源 Digest 判断是否需要同步，不会每天检查 ACR 中的镜像是否被删除、覆盖或修改。

如果目标镜像被外部删除，但源 Digest 没有变化，工作流仍可能跳过复制。因此，应避免手动修改目标镜像或状态文件；如果需要强制重新同步，可以在确认后删除对应的历史 Digest 记录，再运行工作流。

## 八、镜像架构说明

当前代码中的 Skopeo 复制参数为：

```bash
skopeo copy \
  --override-os linux \
  --override-arch amd64 \
  --authfile "$AUTH_FILE" \
  "docker://$SOURCE" \
  "docker://$TARGET"
```

Docker 兜底方式为：

```bash
docker pull --platform linux/amd64 "$SOURCE"
docker tag "$SOURCE" "$TARGET"
docker push "$TARGET"
```

需要注意：

- `--override-os` 和 `--override-arch` 用于指定平台覆盖参数，不等同于复制全部架构。
- 当前 Skopeo 命令没有使用 `--all`。
- Docker 兜底明确拉取 `linux/amd64`，不保留完整的多架构 Manifest List。
- 因此，当前实现不能保证将源镜像的所有架构完整同步到 ACR。

如果项目要求完整保留多架构镜像，应专门调整 Skopeo 的复制策略，并验证复制后的 Manifest List 或 OCI Index，而不能仅依赖 Docker pull/tag/push 兜底。

## 九、企业微信通知

工作流通过企业微信群机器人发送同步结果。

通知步骤配置为 `if: always()`，因此即使同步步骤失败，也会尝试发送通知。

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

同步失败：
grafana/grafana:latest - Docker push failed
```

通知脚本会读取以下文件：

```text
$RUNNER_TEMP/updated.txt
$RUNNER_TEMP/unchanged.txt
$RUNNER_TEMP/failed.txt
```

如果任务在这些文件创建之前就失败，通知脚本会将不存在的文件视为空列表，并提示查看 GitHub Actions 日志。

### 通知注意事项

- 必须配置 `WECHAT_WEBHOOK_URL`。
- Webhook 必须有效，机器人接口必须可访问。
- 当前通知步骤在 Webhook 未配置、请求失败或企业微信返回错误时会退出失败。
- 因此，即使镜像同步成功，通知失败仍可能导致整个 Job 最终显示失败。
- 如果希望通知故障不影响同步结果，需要单独调整通知步骤的错误处理策略。

## 十、常见问题

### 1. `ACR_USERNAME: unbound variable`

可能原因：

- Secret 名称不一致。
- 没有在对应步骤的 `env` 中映射 Secret。
- 环境变量未定义，而脚本启用了 `set -u`。

检查工作流中是否正确配置：

```yaml
env:
  ACR_REGISTRY: ${{ secrets.ACR_REGISTRY }}
  ACR_NAMESPACE: ${{ secrets.ACR_NAMESPACE }}
  ACR_USERNAME: ${{ secrets.ACR_USERNAME }}
  ACR_PASSWORD: ${{ secrets.ACR_PASSWORD }}
```

### 2. `requested access to the resource is denied`

检查以下内容：

- Registry 地址是否正确。
- ACR 命名空间是否存在。
- 账号是否拥有推送权限。
- 目标镜像名称是否符合预期。
- Docker 兜底时是否已经登录 ACR。
- 如果源镜像为私有镜像，是否配置了 Docker Hub 登录。

### 3. `JSONDecodeError`

可能原因：

- `state/digests.json` 不是有效 JSON。
- 源 Manifest 获取失败或内容无效。
- 程序试图解析空文件。

当前工作流会检查状态 JSON 的有效性，并在解析源 Manifest 前检查命令是否成功、文件是否为空以及内容是否为有效 JSON。

### 4. 镜像一直被判定为更新

可能原因：

- `state/digests.json` 没有成功提交到 GitHub。
- 源镜像的顶层 Manifest 内容发生变化。
- 获取到的 Manifest 字节与历史记录不同。

建议检查：

- 历史 Digest 是否存在。
- 源 Manifest 是否发生变化。
- 状态文件是否成功提交到 `main` 分支。

### 5. Skopeo 失败后没有执行 Docker 兜底

当前 Docker 兜底只处理 `skopeo copy` 失败。

如果 `skopeo inspect --raw` 失败，或者源 Manifest 为空、格式无效，工作流会直接将该镜像记录为失败并继续处理下一个镜像。

### 6. Docker pull 失败

检查：

- GitHub Actions Runner 是否能访问 Docker Hub。
- 源镜像是否公开。
- 镜像名称和 Tag 是否正确。
- 是否需要 Docker Hub 登录。
- 源镜像是否支持 `linux/amd64` 平台。

### 7. ACR 中的镜像与源镜像架构不一致

Docker 兜底只拉取 `linux/amd64`。如果需要多架构镜像，应调整复制策略，避免仅同步单个平台的镜像。

### 8. 镜像已同步，但工作流仍然显示失败

检查：

- Digest 状态文件是否成功提交。
- `git push` 是否被分支保护规则拒绝。
- 企业微信 Webhook 是否配置正确。
- 通知接口是否返回错误。

任务最终状态不仅取决于镜像复制，还受到状态提交和通知步骤的影响。

## 十一、安全建议

- 使用 GitHub Secrets 保存账号密码和 Webhook。
- 避免在日志中打印密码、访问令牌和完整 Webhook 地址。
- 使用权限范围尽可能小的 ACR 账号。
- 定期检查 GitHub Actions 运行记录。
- 为 `main` 分支配置合理的保护规则。
- 在正式运行前，先手动执行工作流验证镜像同步、状态提交和通知流程。

## 十二、AI 辅助开发

本项目在开发过程中使用了 ChatGPT（OpenAI）、DeepSeek 作为 AI 辅助工具，协助进行方案设计、工作流编写、问题排查及文档整理。

主要包括：

- 设计 GitHub Actions 定时镜像同步方案。
- 编写和调整 Skopeo 复制及 Docker 兜底逻辑。
- 设计基于源镜像 Digest 的增量同步机制。
- 适配阿里云 ACR 的目标镜像命名规则。
- 整理企业微信机器人通知流程。
- 协助排查 GitHub Actions 环境变量、权限、JSON 解析及 Bash 错误处理问题。
- 编写项目配置说明和常见问题文档。

# EdgeOne Pages 自动重新部署方案

> 基于 GitHub Actions 定时任务，监听目标仓库更新并自动触发 EdgeOne Pages 部署钩子

## 一、方案概述

### 1.1 背景

当前平台的定时任务（cron-tick）在免费计划中最小间隔为 1 天（86400s），无法支持 2 小时一次的调度需求。本方案利用 **GitHub Actions** 的 `schedule` 触发器（最小精度 5 分钟）作为外部调度器，每 2 小时检查目标 GitHub 仓库是否有更新，若有更新则调用 **EdgeOne Pages 部署钩子（Deploy Hook）** 触发重新部署。

### 1.2 目标

- 监听 `https://github.com/1c7/chinese-independent-developer` 仓库的更新
- 每 2 小时检查一次
- 仅在检测到新提交时触发 EdgeOne Pages 重新部署
- 避免无更新时的无效部署

### 1.3 整体架构

```
┌─────────────────────┐   每2小时（cron）   ┌──────────────────────────┐
│  GitHub Actions     │ ──────────────────→ │  GitHub Actions Runner   │
│  (Schedule Trigger) │                      │  (ubuntu-latest)         │
└─────────────────────┘                      └───────────┬──────────────┘
                                                         │
                              ┌──────────────────────────┼──────────────────────────┐
                              │                          │                          │
                              ▼                          ▼                          ▼
                ┌─────────────────────┐  有更新  ┌──────────────────┐  无更新  ┌─────────────┐
                │  GitHub API         │ ───────→ │  EdgeOne Pages  │ ───────→ │  结束      │
                │  获取最新 commit SHA │          │  Deploy Hook    │          │  (不部署)   │
                └─────────────────────┘          └──────────────────┘          └─────────────┘
                                                         │
                                                         ▼
                                                ┌──────────────────┐
                                                │  重新构建并部署   │
                                                │  EdgeOne Pages   │
                                                └──────────────────┘
```

### 1.4 技术要点

| 组件 | 技术/服务 | 说明 |
|------|-----------|------|
| 调度器 | GitHub Actions `schedule` | 每 2 小时触发一次 |
| 变更检测 | GitHub REST API | 获取仓库最新 commit SHA |
| 状态存储 | GitHub Actions Cache | 缓存上次检测到的 SHA，用于对比 |
| 部署触发 | EdgeOne Pages Deploy Hook | POST 请求触发重新部署 |

---

## 二、前置准备

### 2.1 获取 EdgeOne Pages 部署钩子 URL

1. 登录 [EdgeOne Pages 控制台](https://edgeone.cloud.tencent.com/)
2. 进入你的项目 → **项目设置** 页面
3. 找到 **部署钩子（Webhook）** 模块
4. 点击 **新建部署钩子**
5. 输入名称（如 `github-watch-deploy`），选择要触发的分支（如 `master` 或 `main`）
6. 点击 **确定**
7. 复制生成的唯一 URL，格式类似：
   ```
   https://api.edgeone.com/pages/deploy/webhook/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   ```

> ⚠️ **安全提醒**：部署钩子 URL 与项目唯一关联，无需额外认证。请务必妥善保护，避免泄露。如怀疑泄露，立即删除并重新生成。

### 2.2 创建 GitHub 仓库

如果没有现成的仓库，创建一个新的 GitHub 仓库（Public 或 Private 均可），用于存放 Workflow 文件。

---

## 三、实现步骤

### 3.1 创建 Workflow 文件

在你的 GitHub 仓库中创建 `.github/workflows/edgeone-redeploy.yml`：

```yaml
name: EdgeOne Auto Redeploy

on:
  schedule:
    # 每 2 小时触发一次（UTC 时间）
    - cron: '0 */2 * * *'
  workflow_dispatch:  # 允许手动触发，方便测试

env:
  TARGET_REPO: "1c7/chinese-independent-developer"
  TARGET_BRANCH: "master"

jobs:
  check-and-deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # ─── 步骤1: 获取目标仓库最新 commit SHA ───
      - name: Get latest commit SHA from target repo
        id: get_sha
        run: |
          LATEST_SHA=$(curl -s -H "Accept: application/vnd.github.sha" \
            "https://api.github.com/repos/${{ env.TARGET_REPO }}/commits/${{ env.TARGET_BRANCH }}")
          echo "latest_sha=${LATEST_SHA}" >> $GITHUB_OUTPUT
          echo "Latest SHA: ${LATEST_SHA}"

      # ─── 步骤2: 从 cache 读取上次记录的 SHA ───
      - name: Restore last recorded SHA from cache
        id: cache_sha
        uses: actions/cache@v4
        with:
          path: .last_sha
          key: repo-sha-${{ env.TARGET_REPO }}-${{ env.TARGET_BRANCH }}

      # ─── 步骤3: 对比 SHA，判断是否有更新 ───
      - name: Compare SHA and decide whether to deploy
        id: check_update
        run: |
          CURRENT_SHA="${{ steps.get_sha.outputs.latest_sha }}"
          
          if [ -f .last_sha ]; then
            LAST_SHA=$(cat .last_sha)
            echo "Last recorded SHA: ${LAST_SHA}"
          else
            echo "No previous SHA found (first run)"
            LAST_SHA=""
          fi
          
          if [ "${CURRENT_SHA}" != "${LAST_SHA}" ]; then
            echo "🔄 Repository has been updated!"
            echo "  Old SHA: ${LAST_SHA}"
            echo "  New SHA: ${CURRENT_SHA}"
            echo "should_deploy=true" >> $GITHUB_OUTPUT
          else
            echo "✅ No changes detected, skip deployment"
            echo "should_deploy=false" >> $GITHUB_OUTPUT
          fi

      # ─── 步骤4: 触发 EdgeOne Pages 部署钩子 ───
      - name: Trigger EdgeOne Pages Deploy Hook
        if: steps.check_update.outputs.should_deploy == 'true'
        run: |
          echo "🚀 Triggering EdgeOne Pages deployment..."
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
            -X POST "${{ secrets.EDGEONE_DEPLOY_HOOK_URL }}" \
            -H "Content-Type: application/json")
          echo "Deploy Hook Response Code: ${HTTP_CODE}"
          
          if [ "${HTTP_CODE}" -ge 200 ] && [ "${HTTP_CODE}" -lt 300 ]; then
            echo "✅ Deployment triggered successfully"
          else
            echo "❌ Failed to trigger deployment (HTTP ${HTTP_CODE})"
            exit 1
          fi

      # ─── 步骤5: 更新记录的 SHA ───
      - name: Update recorded SHA
        if: steps.check_update.outputs.should_deploy == 'true'
        run: |
          echo "${{ steps.get_sha.outputs.latest_sha }}" > .last_sha
          echo "Updated .last_sha to ${{ steps.get_sha.outputs.latest_sha }}"
```

### 3.2 配置 GitHub Secrets

进入你的 GitHub 仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret 名称 | 值 | 说明 |
|---|---|---|
| `EDGEONE_DEPLOY_HOOK_URL` | `https://api.edgeone.com/pages/deploy/webhook/xxx...` | 从 3.1 获取的部署钩子 URL |

### 3.3 启用 GitHub Actions

1. 确保仓库的 Actions 已启用（默认开启）
2. 推送代码后，进入 **Actions** 标签页确认 Workflow 已被识别
3. 可以手动点击 **Run workflow** 测试一次

---

## 四、方案详解

### 4.1 变更检测原理

利用 GitHub REST API 的 commits 端点获取指定分支的最新 commit SHA：

```
GET https://api.github.com/repos/{owner}/{repo}/commits/{branch}
Accept: application/vnd.github.sha
```

该请求仅返回 SHA 字符串（如 `22d105a...`），非常轻量。通过将当前 SHA 与上次记录的 SHA 对比，即可判断仓库是否有新提交。

### 4.2 SHA 持久化方案

| 方案 | 优点 | 缺点 | 本方案选择 |
|------|------|------|-----------|
| Actions Cache | 官方支持、自动管理 | Cache 有 7 天过期限制 | ✅ 使用 |
| Artifact | 无过期限制 | 每次需上传/下载，较繁琐 | — |
| 仓库文件提交 | 永久保存 | 需要额外 commit，污染历史 | — |
| 外部存储（KV/DB） | 灵活可靠 | 需要额外服务 | — |

> **Cache 过期处理**：如果超过 7 天未触发（例如 schedule 未运行），cache 可能失效。此时 `.last_sha` 文件不存在，Workflow 会判定为"有更新"并触发部署——这是安全的兜底行为（最多多部署一次）。

### 4.3 Cron 时间说明

| Cron 表达式 | 含义 | UTC 时间 | 北京时间 |
|---|---|---|---|
| `0 */2 * * *` | 每 2 小时整点 | 00:00, 02:00, ..., 22:00 | 08:00, 10:00, ..., 06:00 |
| `0 0-22/2 * * *` | 同上 | 同上 | 同上 |
| `30 */2 * * *` | 每 2 小时 +30 分钟 | 00:30, 02:30, ... | 08:30, 10:30, ... |

> ⚠️ GitHub Actions 使用 **UTC 时区**，且 scheduled jobs 通常有 **15~30 分钟** 的延迟，不会精确到秒。

### 4.4 EdgeOne Deploy Hook 调用

```bash
curl -X POST "${EDGEONE_DEPLOY_HOOK_URL}" \
  -H "Content-Type: application/json"
```

- 方法：`POST`
- 无需认证头（URL 本身包含唯一 token）
- 可附带 body 传递部署信息（可选）

---

## 五、增强方案（可选）

### 5.1 添加通知机制

当触发部署时，通过 Webhook 发送通知（如企业微信、飞书、钉钉）：

```yaml
      - name: Send notification on deploy
        if: steps.check_update.outputs.should_deploy == 'true'
        run: |
          curl -X POST "${{ secrets.WEBHOOK_NOTIFY_URL }}" \
            -H "Content-Type: application/json" \
            -d '{
              "msg_type": "text",
              "content": {
                "text": "🚀 EdgeOne Pages 已触发重新部署\n仓库: ${{ env.TARGET_REPO }}\n新 SHA: ${{ steps.get_sha.outputs.latest_sha }}\n时间: '"$(date -u '+%Y-%m-%d %H:%M:%S UTC')"'"
              }
            }'
```

### 5.2 记录部署日志

将每次部署记录到文件并作为 Artifact 保存：

```yaml
      - name: Save deployment log
        if: steps.check_update.outputs.should_deploy == 'true'
        run: |
          mkdir -p logs
          echo "$(date -u '+%Y-%m-%dT%H:%M:%SZ') | ${{ steps.get_sha.outputs.latest_sha }} | ${{ env.TARGET_REPO }}" >> logs/deploy.log

      - name: Upload deployment log
        if: steps.check_update.outputs.should_deploy == 'true'
        uses: actions/upload-artifact@v4
        with:
          name: deploy-log
          path: logs/deploy.log
```

### 5.3 防止频繁部署（防抖）

如果目标仓库在短时间内频繁更新，可以添加"冷静期"限制：

```yaml
      - name: Check cooldown period
        id: cooldown
        run: |
          # 读取上次部署时间，至少间隔 30 分钟才允许再次部署
          COOLDOWN_SECONDS=1800
          NOW=$(date +%s)
          
          if [ -f .last_deploy_time ]; then
            LAST_DEPLOY=$(cat .last_deploy_time)
            ELAPSED=$((NOW - LAST_DEPLOY))
            if [ ${ELAPSED} -lt ${COOLDOWN_SECONDS} ]; then
              echo "⏳ Cooldown active, skipping (${ELAPSED}s < ${COOLDOWN_SECONDS}s)"
              echo "skip=true" >> $GITHUB_OUTPUT
            else
              echo "skip=false" >> $GITHUB_OUTPUT
            fi
          else
            echo "skip=false" >> $GITHUB_OUTPUT
          fi
```

---

## 六、常见问题

### Q1: GitHub Actions 的 scheduled job 不触发？

- 确保仓库有最近 60 天内的 commit 活动（fork 项目需要在自己账号下创建新 commit）
- schedule 只在默认分支（main/master）上的 workflow 生效
- GitHub 对 schedule 有执行保障但不保证精确时间，延迟 15~30 分钟属正常

### Q2: 免费额度够用吗？

| 资源 | 免费额度（私有仓库） | 本方案消耗 |
|------|---------------------|-----------|
| 计算时间 | 2000 分钟/月 | 每次约 1 分钟，12 次/天 × 30 天 = 360 分钟/月 |
| 完全够用 | ✅ | |

> 公开仓库的 Actions 使用完全免费，无时间限制。

### Q3: 如果目标仓库是私有仓库怎么办？

本方案中的目标仓库 `1c7/chinese-independent-developer` 是公开仓库，无需认证即可获取 commit SHA。如果未来需要监听私有仓库，需要在 Workflow 中配置 `GITHUB_TOKEN` 或个人访问令牌。

### Q4: Deploy Hook URL 泄露了怎么办？

1. 立即到 EdgeOne Pages 控制台删除该部署钩子
2. 重新创建一个新的部署钩子
3. 更新 GitHub Secret 中的值
4. 检查 EdgeOne 部署记录是否有异常部署

---

## 七、完整文件清单

```
your-repo/
├── .github/
│   └── workflows/
│       └── edgeone-redeploy.yml    # 主 Workflow 文件
├── .last_sha                        # 自动生成，记录上次 SHA（gitignore）
└── README.md
```

`.gitignore` 建议添加：

```
.last_sha
logs/
```

---

## 八、总结

本方案的核心思路是：

1. **GitHub Actions** 充当高精度定时器（每 2 小时）
2. 通过 **GitHub API** 获取目标仓库最新 commit SHA
3. 利用 **Cache** 持久化上次 SHA，实现增量检测
4. 仅在检测到更新时调用 **EdgeOne Deploy Hook** 触发部署
5. 整个过程免费、自动化、无需额外服务器

这样即使原平台的 cron 限制为 1 天，也能通过 GitHub Actions 实现更频繁的调度需求。

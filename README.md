# EdgeOne Pages 自动重新部署

基于 GitHub Actions 定时任务，监听目标 GitHub 仓库更新并自动触发 EdgeOne Pages 部署钩子（Deploy Hook）重新部署。

## 背景

EdgeOne Pages 免费套餐的定时触发器最小间隔为 1 天（86400s），无法满足更频繁的调度需求。本方案用 **GitHub Actions 的 `schedule` 触发器**（最小精度 5 分钟）作为外部调度器，每 2 小时检查目标仓库是否有更新，仅在检测到新提交时调用 Deploy Hook 触发重新部署，避免无效部署。

## 工作原理

```
GitHub Actions (schedule, 每2小时)
        │
        ▼
GET 目标仓库最新 commit SHA
        │
        ▼
对比上次缓存的 SHA（actions/cache）
        │
   ┌────┴─────┐
 有更新      无更新
   │           │
   ▼           ▼
 POST       结束（不部署）
 Deploy Hook
   │
   ▼
更新缓存的 SHA
```

1. `schedule` 每 2 小时（UTC）触发工作流。
2. 调用 GitHub REST API 获取目标仓库指定分支的最新 commit SHA。
3. 通过 `actions/cache` 读取上次记录的 SHA 并对比；不一致则判定为「有更新」。
4. 有更新时 `POST` EdgeOne Pages Deploy Hook 触发重新部署，校验 HTTP 2xx 成功。
5. 部署后把最新 SHA 写回缓存。
6. Cache 有 7 天过期限制：若 cache 失效，`.last_sha` 不存在，会判定为「有更新」触发部署——这是安全的兜底（最多多部署一次）。

## 文件说明

| 文件 | 说明 |
|------|------|
| `.github/workflows/edgeone-redeploy.yml` | 主工作流 |
| `.gitignore` | 忽略自动生成的 `.last_sha` 与 `logs/` |

## 使用前配置

### 1. 获取 EdgeOne Pages 部署钩子 URL

1. 登录 [EdgeOne Pages 控制台](https://edgeone.cloud.tencent.com/) → 进入你的项目 → **项目设置**。
2. 找到 **部署钩子（Webhook）** 模块，点击 **新建部署钩子**。
3. 输入名称（如 `github-watch-deploy`），选择要触发的分支（如 `master` / `main`）。
4. 复制生成的 URL，格式类似：

   ```
   https://api.edgeone.com/pages/deploy/webhook/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   ```

> ⚠️ 部署钩子 URL 与项目唯一关联，无需额外认证。请妥善保护，避免泄露；如怀疑泄露立即删除并重新生成。

### 2. 配置 GitHub Secret

进入本仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**：

| Secret 名称 | 值 | 说明 |
|---|---|---|
| `EDGEONE_DEPLOY_HOOK_URL` | `https://api.edgeone.com/pages/deploy/webhook/xxx...` | 上一步获取的部署钩子 URL |

### 3. 自定义目标仓库（可选）

默认监听 `1c7/chinese-independent-developer` 的 `master` 分支。如需修改，编辑工作流文件顶部的 `env`：

```yaml
env:
  TARGET_REPO: "owner/repo"
  TARGET_BRANCH: "main"
```

> 目标仓库为公开仓库时，读取 commit SHA 无需额外认证。若监听私有仓库，需在工作流中配置 `GITHUB_TOKEN` 或个人访问令牌。

## 测试

1. 推送代码后，进入仓库 **Actions** 标签页确认工作流已被识别。
2. 在 **EdgeOne Auto Redeploy** 工作流页面点击 **Run workflow** 手动触发一次，验证：
   - 首次运行（无缓存）会判定为「有更新」并触发 Deploy Hook；
   - 无新提交时再次运行会输出 `✅ No changes detected, skip deployment`。

## 注意事项

- GitHub Actions 的 `schedule` 使用 **UTC 时区**，且 scheduled job 通常有 **15~30 分钟** 延迟，不会精确到秒。
- 公开仓库的 Actions 使用完全免费，无时间限制。
- Deploy Hook URL 含唯一令牌，已通过 Secret 注入，切勿硬编码进文件或提交到仓库。

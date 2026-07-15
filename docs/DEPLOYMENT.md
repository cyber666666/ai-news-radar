# 部署说明

V1 正式部署只使用 GitHub Actions + GitHub Pages。无需数据库、用户系统、Docker 或常驻本地进程。

## 目标地址

- 仓库：`https://github.com/cyber666666/ai-news-radar`
- GitHub Pages：`https://cyber666666.github.io/ai-news-radar/`
- Actions：`https://github.com/cyber666666/ai-news-radar/actions/workflows/update-news.yml`

Pages 地址只有在仓库设置中启用后才可访问。

## 1. 启用 GitHub Actions

fork 默认可能暂停上游 workflow：

1. 打开仓库 **Actions** 页；
2. 如出现 “Workflows aren’t being run on this forked repository”，点击启用；
3. 打开 **Update AI News Snapshot**；
4. 使用 **Run workflow** 在 `master` 手动运行一次；
5. 确认任务完成，并产生新的 `chore: update ai news snapshot` 提交。

工作流保留 `workflow_dispatch`，并支持可选布尔输入 `force_tikhub`。没有 TikHub Key 时不要启用该输入。

## 2. 定时任务与并发

```yaml
schedule:
  - cron: "0 */4 * * *"
concurrency:
  group: update-ai-news-${{ github.ref }}
  cancel-in-progress: true
```

- GitHub cron 使用 UTC；
- 计划为 UTC 00:00、04:00、08:00、12:00、16:00、20:00；
- UTC+8 与 UTC 相差两个 4 小时间隔，因此北京时间仍是 00:00、04:00、08:00、12:00、16:00、20:00；
- GitHub schedule 可能因平台负载延迟，不保证精确到分钟；页面应以 JSON `generated_at` 为实际更新时间；
- 同一分支只保留最新快照任务，旧任务仍在运行时会被取消，避免重复提交旧数据。

## 3. 启用 GitHub Pages

1. 打开 **Settings -> Pages**；
2. **Build and deployment** 的 Source 选择 **Deploy from a branch**；
3. Branch 选择 `master`，目录选择 `/ (root)`；
4. 保存并等待首次部署完成；
5. 打开 `https://cyber666666.github.io/ai-news-radar/`；
6. 检查移动默认视图和 `/classic/` 经典视图；
7. 确认页面更新时间等于最新 `data/latest-24h.json` 的 `generated_at`。

当前项目不需要单独的 Pages build workflow；Pages 直接发布 `master` 根目录静态文件。

## 4. GitHub Secrets

在 **Settings -> Secrets and variables -> Actions** 中配置。不要把值写进仓库、Issue、Actions 日志或生成 JSON。

### V1 私人来源

| 名称 | 必需 | 说明 |
| --- | --- | --- |
| `FOLLOW_OPML_B64` | 个人 OPML 阶段必需 | 私人 `feeds/follow.opml` 的单行 Base64；未设置时使用公开示例 OPML |

本地生成 Secret 值，不把输出保存到仓库：

```bash
base64 < feeds/follow.opml | tr -d '\n'
```

将输出粘贴到 GitHub Secret 后清理终端滚屏；不要把值写入说明文档或聊天。

### 可选增强

| 名称 | 用途 |
| --- | --- |
| `DEEPSEEK_API_KEY` | 标题增强、翻译、推荐理由和 persona LLM 评分 |
| `X_BEARER_TOKEN` | 官方 X API；V1 默认不启用 |
| `SOCIALDATA_API_KEY` | SocialData X 搜索；V1 默认不启用 |
| `TIKHUB_API_KEY` | TikHub 抖音/小红书；完成账号评估前不启用 |
| `AGENTMAIL_API_KEY` | AgentMail 元数据摘要；默认不启用 |
| `AGENTMAIL_INBOX_ID` | AgentMail inbox 标识；默认不启用 |
| `EMAIL_DIGEST_ENABLED` | 邮箱摘要总开关；默认不设置 |
| `EMAIL_DIGEST_PUBLISH` | 明确允许发布脱敏摘要；默认不设置 |

可选非敏感调参使用 GitHub Actions Variables，完整清单见 `examples/advanced-sources.env.example` 与 `docs/CONFIG_REFERENCE.md`。

## 5. 部署验证

手动运行后检查：

```bash
curl -fsS https://cyber666666.github.io/ai-news-radar/data/latest-24h.json
curl -fsS https://cyber666666.github.io/ai-news-radar/data/source-status.json
curl -fsS https://cyber666666.github.io/ai-news-radar/
curl -fsS https://cyber666666.github.io/ai-news-radar/classic/
```

验收点：

- Actions 任务成功；
- `data/*.json` 产生新提交；
- Pages 返回 HTTP 200；
- 页面实际更新时间更新；
- `source-status.json.failed_sites` 可解释；
- 单个失败源没有阻塞提交和 Pages；
- Actions 日志、JSON 和 Git diff 中无密钥、Cookie、Token 或私人 OPML 内容。

## 6. 本地验证

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
python -m py_compile scripts/update_news.py scripts/persona_score.py
python -m pytest -q
python scripts/update_news.py --output-dir /tmp/ai-news-radar-data --window-hours 24 --rss-opml feeds/follow.example.opml
python scripts/persona_score.py --data-dir /tmp/ai-news-radar-data
```

不要用私人 OPML 生成准备提交到公开仓库的数据，除非已经确认输出不包含私人来源信息。

## 7. 当前环境变量变更

阶段 1–3 没有新增环境变量、Secret 或 Variable；只复用现有配置入口。

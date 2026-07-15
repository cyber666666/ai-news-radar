# 日常运维说明

## 更新节奏

- 自动更新：每 4 小时一次；
- 计划时点：00:00、04:00、08:00、12:00、16:00、20:00；
- 时区：GitHub cron 按 UTC 解释；本项目的 4 小时间隔在 Asia/Shanghai 得到同一组时点；
- 实际更新时间：以 `data/latest-24h.json.generated_at` 为准；
- 手动更新：Actions -> Update AI News Snapshot -> Run workflow。

## 每日快速检查

1. 打开 Pages，确认更新时间不是旧值；
2. 查看“精选/全量”能否切换；
3. 查看 `data/source-status.json`；
4. 关注 `failed_sites`、`zero_item_sites`、`rss_opml.failed_feeds` 和 `empty_advanced_sources`；
5. 确认最新 Actions 任务有对应数据提交；
6. 抽查 3–5 条原文链接、发布时间和来源标签。

单源失败不等于整轮失败。只有以下情况应视为发布阻塞：

- `update_news.py` 或 persona 步骤非零退出；
- 核心 JSON 未生成或不可解析；
- 数据没有 staged/committed；
- push 失败且重试后仍失败；
- Pages 无法读取核心 JSON；
- 日志或生成数据包含敏感信息。

## 本地手动验证

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt

python -m py_compile scripts/update_news.py scripts/persona_score.py
python -m pytest -q
node --check assets/app.js
node --check classic/assets/app.js

python scripts/update_news.py \
  --output-dir /tmp/ai-news-radar-preview/data \
  --window-hours 24 \
  --archive-days 21 \
  --rss-opml feeds/follow.example.opml

python scripts/persona_score.py --data-dir /tmp/ai-news-radar-preview/data
```

若要在浏览器检查，不要覆盖仓库的生产 `data/`；把静态文件和临时数据放进独立预览目录后运行：

```bash
python -m http.server 8080 --bind 127.0.0.1 --directory /tmp/ai-news-radar-preview
```

打开 `http://127.0.0.1:8080/` 和 `http://127.0.0.1:8080/classic/`。

## 来源故障排查

### 内置公开来源

查看 `source-status.json.sites[]`：

- `ok`：本轮函数是否正常结束；
- `item_count`：抓取条目数；
- `duration_ms`：耗时；
- `error`：脱敏错误文本。

单个内置来源由 `collect_all()` 单独捕获异常。先复现该来源，不要为了一个来源修改整个管线或放宽全局过滤。

### OPML/RSS

查看：

- `rss_opml.failed_feeds`；
- `rss_opml.zero_item_feeds`；
- `rss_opml.skipped_feeds`；
- `rss_opml.replaced_feeds`；
- `rss_opml.feeds[]` 的耗时与错误。

每个 Feed 单独超时并隔离。处理顺序：确认官方 Feed URL -> 本地单源测试 -> 检查时间戳 -> 必要时标为观察/跳过。不要因一个 Feed 失败删除整个 OPML。

### 高级来源

X、SocialData、TikHub 和 AgentMail 未配置 Key 时显示为 disabled/skipped，这是正常降级，不是失败。

若启用：

- 先检查 Key 是否存在，不打印 Key；
- 检查费用/数量上限；
- 检查运行间隔状态 `data/paid-source-state.json`；
- 配额耗尽或第三方 API 失败时保留其他公开来源；
- 在完成具体账号评估前，不批量增加社交平台抓取器。

## Actions 故障排查

按步骤定位：

1. **Checkout/Install**：依赖或 GitHub 服务问题；
2. **Prepare OPML**：只确认 Secret 是否存在，不输出或下载 Secret 内容；
3. **Update data**：结合 `source-status.json` 区分单源失败和进程失败；
4. **Persona scoring**：无 DeepSeek Key 应规则降级，不应失败；
5. **Commit and push**：检查权限、分支保护和 rebase 重试。

工作流 `permissions.contents=write` 必须保留，否则数据无法回写。并发组按分支隔离，`cancel-in-progress: true` 保证只发布最新快照。

## Pages 故障排查

1. 确认 Pages Source 是 `master` + `/ (root)`；
2. 确认最新数据提交已在 `master`；
3. 直接访问 `/data/latest-24h.json`；
4. 对比 JSON `generated_at` 和页面更新时间；
5. 检查浏览器网络请求是否为 404；
6. 等待 Pages 部署缓存刷新后再复查。

## 安全检查

提交前：

```bash
git status --short
git diff --check
git diff --cached
```

重点确认：

- `feeds/follow.opml` 未被跟踪；
- `.env` 未被跟踪；
- 代码、文档、日志和 JSON 无 API Key、Token、Cookie、Authorization Header 或私人 OPML 内容；
- 错误只记录错误类型/消息，不记录请求头；
- `data/email-digest.json` 仅在明确启用发布时提交。

## 变更纪律

- 来源、persona、调度分别提交；
- 每个阶段运行相关测试；
- 不为格式或“更漂亮”重构无关代码；
- 新来源先进入清单并评估，再决定 OPML、现有适配器或最小 fetcher；
- 运行至少 7 天后再决定 V1.1。

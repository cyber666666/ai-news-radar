# 个人 AI 行业资讯雷达 V1：仓库审计

审计日期：2026-07-16
审计提交：`3c78d874910e2e1ab45443a50ee657027396769f`

## 结论

当前项目已经具备 V1 所需的主体能力：静态双视图、单进程数据管线、内置公开来源、私人 OPML、来源状态、故事合并、persona 和 Actions 回写数据。V1 应以配置、文档和小范围修正为主，不需要重写前端、引入数据库、用户系统、推荐模型、多 Agent 或 Docker。

远端 fork 尚未形成部署基线：公开 GitHub API 返回 0 个 Actions 工作流/运行记录，仓库元数据为 `has_pages=false`，Pages API 返回 404。必须先启用 fork 的 Actions 和 Pages，才能完成线上验收。

## 当前架构

```text
公开内置来源 + OPML + 可选密钥源
               |
               v
scripts/update_news.py
  抓取 -> 归一化 -> 24h 窗口 -> AI 相关性 -> 去重 -> 故事合并
               |
               +--> data/*.json
               |
scripts/persona_score.py
  规则降级或 DeepSeek persona 评分
               |
               +--> daily-brief.json / top3-personas.json
               |
               v
index.html + assets/*            classic/index.html + classic/assets/*
移动默认视图                    经典桌面视图
               |
               v
GitHub Pages（master 根目录）
```

- 后端形态：无服务端、无数据库；Python 定时生成静态 JSON。
- 前端形态：根目录移动视图与 `/classic/` 经典视图，共享同一 `data/` 数据契约。
- 部署形态：GitHub Actions 在 `master` 生成并提交 `data/`，GitHub Pages 从分支根目录发布。
- 个性化入口：`feeds/follow.opml`、`FOLLOW_OPML_B64`、`personas/*.md` 及可选环境变量。

## 数据抓取入口

- 主入口：`python scripts/update_news.py --output-dir data --window-hours 24`。
- 私人 OPML：追加 `--rss-opml feeds/follow.opml`，可用 `--rss-max-feeds` 限制数量。
- persona：`python scripts/persona_score.py --data-dir data`。
- `collect_all()` 顺序执行 14 组内置来源，每组单独 `try/except` 并写状态，因此单个内置来源失败不会中止全局。
- `fetch_opml_rss()` 对 OPML Feed 使用线程池；每个 Feed 单独捕获异常并产生独立状态。
- WaytoAGI、X API、SocialData、TikHub 和 AgentMail 均有独立状态/降级路径。

## 信息源配置位置

| 类型 | 位置 | 用途 |
| --- | --- | --- |
| 内置公开来源 | `scripts/update_news.py` 中 `collect_all()` 与各 `fetch_*` | 官方、媒体、聚合、社区与公开 JSON/页面 |
| 公开 OPML 示例 | `feeds/follow.example.opml` | 无私人 Secret 时的演示回退 |
| 私人 OPML | `feeds/follow.opml`（被 `.gitignore` 忽略） | 本地个人订阅 |
| 私人 OPML Secret | `FOLLOW_OPML_B64` | Actions 解码生成 `feeds/follow.opml` |
| 可选高级源 | `examples/advanced-sources.env.example` 与环境变量 | X、SocialData、TikHub、AgentMail、DeepSeek |
| persona | `personas/*.md` | 评分立场、区间与点评语气 |
| 来源策略 | `docs/SOURCE_COVERAGE.md` | 新来源接入优先级与维护边界 |

## GitHub Actions 工作流

文件：`.github/workflows/update-news.yml`

- 触发：`workflow_dispatch`（含 `force_tikhub`）和 cron `17 * * * *`，当前为每小时第 17 分钟，而 README 仍写“每 30 分钟”，文档与实现不一致。
- 并发：`update-ai-news-${{ github.ref }}`，`cancel-in-progress: true`。
- 权限：`contents: write`。
- 超时：40 分钟。
- 步骤：checkout -> Python 3.11 -> 安装依赖 -> 准备 OPML -> 更新数据 -> persona -> 提交并推送。
- 数据提交：`git add data/`；邮箱摘要仅在 `EMAIL_DIGEST_PUBLISH=1` 时强制提交。
- 推送冲突：最多重试三次并 rebase。
- Pages 未由 workflow 独立部署；设计依赖 GitHub Pages 的“从 master 根目录发布”设置。

## 页面读取的数据文件

两套前端使用 `Promise.allSettled` 或等价降级方式读取：

- `data/latest-24h.json`：精选/主数据与生成时间；
- `data/latest-24h-all.json`：全量模式；
- `data/stories-merged.json`：完整故事池；
- `data/daily-brief.json`：每日精选与 persona 字段；
- `data/top3-personas.json`：移动视图的可选 TOP3 多 persona 增强；
- `data/source-status.json`：来源状态与覆盖；
- `data/waytoagi-7d.json`：社区更新。

`data/latest-24h-all-raw.json`、`data/archive.json`、`data/merge-log.json`、`data/paid-source-state.json`、`data/persona-cache.json` 和 `data/title-zh-cache.json` 为管线/审计/缓存数据，不是主要前端入口。

## 需要修改的最小文件集合

阶段 1–3：

- `docs/PERSONAL_AI_RADAR_V1_PRD.md`：把附带 DOCX 固化为仓库需求基线；
- `docs/REPOSITORY_AUDIT.md`：记录本审计；
- `.github/workflows/update-news.yml`：改为每 4 小时，保留手动触发并核对并发；
- `docs/DEPLOYMENT.md`：记录 Actions、Pages、Secrets、UTC 与手动验证；
- `docs/OPERATIONS.md`：记录本地运行、失败源排查和日常维护。

后续收到个人来源后才进入：

- `docs/MY_SOURCE_INVENTORY.md`；
- 私人 `feeds/follow.opml`（不得提交）或必要的公开模板；
- 个人 persona 文件；
- 只有现有适配器无法覆盖且来源已通过评估时，才修改 `scripts/update_news.py` 与相关测试。

## 当前测试命令

完整本地验证：

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
python -m py_compile scripts/update_news.py scripts/persona_score.py
python -m pytest -q
node --check assets/app.js
node --check classic/assets/app.js
git diff --check
```

原版抓取与页面：

```bash
python scripts/update_news.py --output-dir data --window-hours 24 --rss-opml feeds/follow.example.opml
python scripts/persona_score.py --data-dir data
python -m http.server 8080
```

## 风险点

1. 远端基线未启用：fork 当前无 Actions 运行记录且 Pages 未启用。
2. 调度文档漂移：workflow 是每小时，README 写每 30 分钟，PRD 要求每 4 小时。
3. GitHub cron 只使用 UTC；若业务期望北京时间 00/04/08/12/16/20，应使用 UTC 0/4/8/12/16/20 对应的表达式或明确接受 UTC 计划。六个时点本身每 4 小时循环，时区只影响语义标注。
4. `cancel-in-progress: true` 会取消仍在运行的旧任务，适合快照任务；改为 4 小时后重叠概率下降，但应保留同分支单实例约束。
5. 抓取依赖大量外部公开端点，网络、限流、结构变更和历史时间戳都可能产生零条目或失败，必须以 `source-status.json` 为准，不以进程退出码替代来源健康判断。
6. 无 DeepSeek Key 时会规则降级；标题增强、真实推荐理由和 LLM persona 不可作为无 Key 基线的成功条件。
7. 高级源可能付费且噪声较高；X、SocialData、TikHub 默认应关闭，收到具体账号并完成逐项评估前不开发批量抓取器。
8. `data/` 体积很大，完整 clone 代价高；本地维护宜使用浅克隆/稀疏检出，生成数据时再按需检出或在独立输出目录验证。
9. 私人 OPML、Key、Cookie 和 Token 必须只通过 Secret/环境变量提供；生成状态和异常文本也要持续检查是否泄漏请求头或敏感值。
10. 仓库当前未提供 `docs/DEPLOYMENT.md`、`docs/OPERATIONS.md` 和 `.env.example`；阶段 3 先补前两者，`.env.example` 在引入个人/新增环境变量时再最小补充。

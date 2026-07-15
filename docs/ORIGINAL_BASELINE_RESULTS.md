# 原版运行基线记录

验证日期：2026-07-16

## 环境

- macOS / Apple Silicon；
- Python 3.12.13；
- pip 25.0.1；
- 依赖安装命令：`.venv/bin/python -m pip install -r requirements-dev.txt`；
- 未设置 DeepSeek、X、SocialData、TikHub、AgentMail 或私人 OPML Secret；
- 抓取使用公开 `feeds/follow.example.opml`。

## 验证命令

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt

.venv/bin/python -m py_compile scripts/update_news.py scripts/persona_score.py
.venv/bin/python -m pytest -q
node --check assets/app.js
node --check classic/assets/app.js
git diff --check

.venv/bin/python scripts/update_news.py \
  --output-dir ../work/baseline-preview/data \
  --window-hours 24 \
  --archive-days 21 \
  --rss-opml feeds/follow.example.opml

.venv/bin/python scripts/persona_score.py \
  --data-dir ../work/baseline-preview/data

.venv/bin/python -m http.server 8080 \
  --bind 127.0.0.1 \
  --directory ../work/baseline-preview
```

## 测试结果

| 检查 | 结果 |
| --- | --- |
| `py_compile` | 通过 |
| Python 测试 | `231 passed in 0.87s` |
| 移动前端 JavaScript 语法 | 通过 |
| 经典前端 JavaScript 语法 | 通过 |
| `git diff --check` | 通过 |

## 抓取结果

生成时间：`2026-07-15T17:12:19.179453Z`（北京时间 2026-07-16 01:12）。

| 数据 | 数量 |
| --- | ---: |
| `latest-24h.json` 精选条目 | 171 |
| `latest-24h-all.json` 全量条目 | 183 |
| `latest-24h-all-raw.json` 原始条目 | 603 |
| 原始条目 URL/标题去重后 | 561 |
| `stories-merged.json` 故事 | 163 |
| `daily-brief.json` 日报条目 | 20 |
| 故事合并事件 | 8 |
| WaytoAGI 7 天条目 | 33 |
| 归档条目 | 3296 |

`update_news.py` 末尾的部分 `Wrote:` 行使用过滤或去重前的局部列表长度，可能与 JSON 顶层统计不同；验收统一以生成 JSON 的顶层字段为准。

## 来源状态

`source-status.json` 结果：

- 成功来源：16；
- 失败来源：0；
- 成功但零条目来源：0；
- 示例 OPML：10/10 成功；
- OPML 失败、跳过、替换或零条目：均为 0。

内置/汇总来源：

| 来源 | 抓取条目 | 状态 |
| --- | ---: | --- |
| Official AI Updates | 174 | 成功 |
| Curated Media | 31 | 成功 |
| AI Breakfast | 6 | 成功 |
| Follow Builders | 40 | 成功 |
| TechURLs | 405 | 成功 |
| Buzzing | 50 | 成功 |
| Info Flow | 50 | 成功 |
| BestBlogs | 1 | 成功 |
| Zeli | 71 | 成功 |
| Hacker News | 1 | 成功 |
| AI HubToday | 25 | 成功 |
| AIbase | 20 | 成功 |
| AI HOT | 97 | 成功 |
| NewsNow | 123 | 成功 |
| WaytoAGI | 3 个 24h 条目 | 成功 |
| OPML RSS | 2200 个 Feed 原始条目 | 成功 |

本次没有失败来源。高级来源未提供 Key，按设计安全跳过：

- X API：`no_bearer_token`；
- SocialData：`no_api_key`；
- TikHub：`no_api_key`；
- AgentMail：未启用。

这些是“未配置”，不是抓取失败，不进入 `failed_sites`。

## Persona 降级

未设置 `DEEPSEEK_API_KEY` 时：

```text
persona: scored=0 cached=0 skipped=0 failed=0
```

脚本使用规则分写回 `daily-brief.json`，并生成空的 `top3-personas.json` 增强数据；核心页面不受影响。

## 页面验证

浏览器实测：

| 页面 | 结果 |
| --- | --- |
| `/` 移动默认视图 | 正常加载，60 个文章节点，显示 `16/16 源正常` |
| `/classic/` 经典视图 | 正常加载，60 个文章节点，显示 `16/16 源正常` |
| 375px 手机视口 | 无横向溢出 |
| 数据请求 | `latest-24h`、`stories-merged`、`daily-brief`、`source-status`、`waytoagi-7d`、`top3-personas` 均 HTTP 200 |
| 浏览器 console | 无 warning/error |

页面显示更新时间为 `07/16 01:12`，与 JSON `generated_at` 一致。

## 基线结论

原版在无任何 API Key 的情况下可完成完整抓取、规则 persona 降级、静态页面展示和全部测试。阶段 3 不需要修改抓取业务逻辑，只需建立 fork 的 Actions/Pages 基线、调整每 4 小时调度，并补部署/运维文档。

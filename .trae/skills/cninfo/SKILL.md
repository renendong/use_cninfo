---
name: "cninfo"
description: "Fetch A-share announcements from cninfo.com.cn on demand (PDF + full-text Markdown, cached). Invoke for 年报/季报/公告/减持/重组 or any cninfo disclosure request."
---

# cninfo — A 股公告按需抓取

把巨潮资讯网（cninfo.com.cn）的公开公告按需变成本地 PDF + 全文 Markdown。**单次执行、命中缓存即秒回、不做后台同步**。

## 何时使用

当用户提出以下需求时调用本 skill：

- 获取某只 A 股指定年份的定期报告：年报 / 一季报 / 半年报 / 三季报
- 查看某只股票某段时间的全部公告
- 按板块 + 日期查看全市场公告切片
- 按标签（减持、重组、股东大会等）或标题关键词搜索公告
- 查询 / 管理公告缓存，或查询股票代码对应的 cninfo `orgId`

**不要用于**：实时行情、财务指标计算、投资建议、非 cninfo 来源的数据。

## 核心原则

1. **按需单次**：只执行用户当前需要的查询，不启动后台任务、不做全量回补。
2. **先列表后下载**：只需要标题/日期时不要加 `--download`；确定要正文才下载。
3. **JSON 优先**：列表类命令加 `--json`，便于解析字段；`fetch-report` 默认就是 JSON。
4. **不要自行拼 cninfo 请求**：接口有 `pageSize=30`、`stock=<code>,<orgId>`、UTC 时区等约束，统一交给本工具处理。
5. **引用来源**：回答中涉及公告内容时，说明来自 cninfo，并给出本地 `md_path` 或公告 `ann_id`。

## 前置检查

```bash
# 1) 优先检查已安装命令
command -v cninfo && cninfo --version

# 2) 没有命令时，从仓库根目录用模块方式
PYTHONPATH=src python -m cninfo.cli --help

# 3) 若模块报 ModuleNotFoundError（如 fitz / requests），安装依赖后重试
pip install -e .
```

`pip install -e .` 会安装 `requests` 与 `pymupdf`。若环境不允许安装，明确告知用户缺少依赖，不要自行改用其它数据源。

也可以使用 `uv tool install -e .` 安装到独立环境。

可选分类器（仅 `search --type/--sub-type` 需要）：

```bash
git clone https://github.com/rollysys/announcement_filter.git
cd announcement_filter && pip install -e .
```

未安装分类器时，改用 `--keyword` 或 `list`，不要阻塞其它任务。

## 命令选择

| 用户需求 | 使用命令 | 是否下载 |
|---|---|---|
| 某股某年定期报告 | `fetch-report` | 自动下载 1 份 |
| 某股一段时间公告 | `fetch-stock` | 默认否，加 `--download` |
| 某板块某日/某段全市场 | `list` | 默认否，加 `--download` |
| 按标签/关键词搜索 | `search` | 默认否，加 `--download` |
| 缓存统计/校验/清理 | `cache` | 否 |
| 查代码对应 orgId | `orgid` | 否 |

### 触发示例

| 用户说法 | 应执行 |
|---|---|
| “拿一下贵州茅台 2024 年报” | `fetch-report 600519 --year 2024 --kind annual` |
| “茅台 2024 年 4 月有哪些公告” | `fetch-stock 600519 --since 2024-04-01 --until 2024-04-30 --json` |
| “2024-04-29 深市所有年报” | `list --plate sz --category annual --date 2024-04-29 --limit 50 --json` |
| “找茅台 2024 年的减持计划” | `search --stock 600519 --since 2024-01-01 --until 2024-12-31 --type shareholder --sub-type reduce_plan --json` |
| “缓存占多大空间” | `cache stats` |

## 标准操作

### 1. 获取定期报告（最常用）

```bash
cninfo fetch-report 600519 --year 2024 --kind annual
```

- `kind`：`annual`（年报）/ `q1`（一季报）/ `h1`（半年报）/ `q3`（三季报）
- 板块默认按代码首位推测：`0/3 → sz`、`6 → sh`、`4/8/9 → bj`
- 推测不准时显式加 `--plate sz|sh|bj`
- 强制重下加 `--force`

返回 JSON，关键字段：

```json
{
  "ann_id": "...",
  "ts_code": "600519.SH",
  "ann_date": "20240412",
  "title": "贵州茅台2024年年度报告",
  "pdf_path": ".../pdf/600519.SH/20240412__xxx.pdf",
  "md_path": ".../md/600519.SH/20240412__xxx.md",
  "total_pages": 286,
  "extracted_pages": 286,
  "text_chars": 412580,
  "cache_hit": false
}
```

拿到 `md_path` 后，用文件读取工具阅读正文；文件包含 YAML frontmatter + 全文。

### 2. 查看某股时间窗公告

```bash
# 仅列表
cninfo fetch-stock 600519 --since 2024-04-01 --until 2024-04-30 --json

# 需要正文时
cninfo fetch-stock 600519 --since 2024-04-01 --until 2024-04-30 --download --json
```

注意：没有 `--limit`，会翻完整个时间窗。时间窗很大时先提醒用户请求量。

### 3. 全市场切片

```bash
# 某日深市全部年报
cninfo list --plate sz --category annual --date 2024-04-29 --limit 50 --json

# 某段时间标题含关键词
cninfo list --plate sh --since 2024-04-01 --until 2024-04-30 \
  --keyword "重大资产重组" --json
```

- `--category` 支持 `annual/q1/h1/q3` 别名或原生 `category_xxx_szsh`
- 务必用 `--limit` 控制返回量
- `--date` 与 `--since/--until` 二选一

### 4. 标签 / 关键词搜索

```bash
# 标签过滤（需 announcement_filter）
cninfo search --stock 600519 --since 2024-01-01 --until 2024-12-31 \
  --type shareholder --sub-type reduce_plan --json

# 仅关键词（不依赖分类器）
cninfo search --plate sz --date 2024-04-29 --keyword "重组" --limit 20 --json
```

- `--stock` 模式必须同时给 `--since` 和 `--until`
- `--plate` 模式必须给 `--date` 或 `--since/--until`
- 标签模式会逐条调用分类器；全市场大范围搜索较慢，先用 `--limit`

### 5. 缓存与 orgId

```bash
cninfo cache stats
cninfo cache verify
cninfo cache prune --older-than 365d

cninfo orgid 600519
cninfo orgid 600519 --refresh
cninfo orgid 600519 --cache-only
```

缓存默认在 `~/.cache/cninfo/`，可用 `CNINFO_CACHE_DIR` 覆盖：

```text
<root>/orgid_map.json
<root>/pdf/<ts_code>/<ann_date>__<ann_id>.pdf
<root>/md/<ts_code>/<ann_date>__<ann_id>.md
<root>/meta/<ann_id>.json
```

## 阅读与呈现

- **大文件**：年报可能 200+ 页。先用 Grep 定位章节/关键词，再用 Read 的 `offset`/`limit` 读取，不要整文件灌入上下文。
- **只读需要的内容**：优先读 frontmatter（`title`、`ann_date`、`total_pages`）和用户关心的章节。
- **扫描件**：`extracted_pages=0` 且 `text_chars=0` 表示 PDF 是图片扫描件。告知用户 PDF 已落盘、正文无法提取、本工具不内置 OCR。
- **缓存命中**：`cache_hit=true` 表示未发网络请求，可直接使用已有文件。
- **输出给用户**：给出 `ann_id`、`ann_date`、`title`、`md_path`；正文按需摘要，不要倾倒全文。

## 错误处理

| 错误 | 含义 | 处理 |
|---|---|---|
| `OrgIdNotFound` | 代码错误 / 已退市 / topSearch 变更 | `cninfo orgid <code> --refresh`；确认代码和板块 |
| `no <kind> report body found` | 窗口外 / 板块猜错 / 只有摘要 | 显式 `--plate`；用 `fetch-stock` 查看原始标题 |
| `announcement_filter not installed` | 未装分类器 | 改用 `--keyword`，或按前置检查安装 |
| `cache verify` 输出 warn | 三件套不一致 | 按 `ann_id` 定位后用 `--force` 重抓 |
| PDF 404 / 超时 / 503 | 上游限流或老 URL 失效 | 稍后重试；缩小范围；不要连续轰炸 |
| `ValueError: must specify either date or both since+until` | `list` 未给时间范围 | 补 `--date` 或 `--since/--until` |

## 约束与安全

- 不做后台增量同步、全量回补、定时任务。
- 全市场查询必须加 `--limit`；大批量下载前先向用户确认。
- 不下载与用户问题无关的公告。
- 不编造公告内容；只引用实际读取到的文件内容。
- 工具已内置约 0.4s 的翻页间隔，不要额外并发请求同一端点。
- 本 skill 只提供数据获取能力，不构成投资建议。

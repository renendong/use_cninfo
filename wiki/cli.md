# 命令行接口：`cli.py`

入口：

```toml
[project.scripts]
cninfo = "cninfo.cli:main"
```

也支持 `python -m cninfo.cli`（文件末尾有 `if __name__ == "__main__"` 保护）。`main(argv=None) -> int` 返回退出码。

---

## 1. 全局约定

### 1.1 输出

| 方式 | 规则 |
|---|---|
| 默认 | 动态列宽表格，列间两空格，表头下一条分隔线 |
| `--json` | `json.dumps(ensure_ascii=False, indent=2)` |
| 无数据 | 表格打印 `(no rows)` |
| 错误 | `error: <message>` 写入 **stderr** |

`fetch-report` 没有 `--json` 开关，**始终输出单个 JSON 对象**。

### 1.2 退出码

| 码 | 含义 | 来源 |
|---|---|---|
| `0` | 成功 | 各 `cmd_*` |
| `1` | 一般错误 | `_err()`：orgId 查不到、参数组合非法、分类器异常 |
| `2` | 未找到报告 | `fetch-report` 遍历后无匹配本体 |
| `3` | 缓存不一致 | `cache verify` 发现告警 |
| `2` | 用法错误 | argparse 自身（缺少必填参数、非法 choices） |

> 退出码 2 有两种语义，脚本化调用时需结合 stderr 判断。

### 1.3 版本

```bash
cninfo --version   # cninfo-cli 0.1.0
```

---

## 2. `fetch-report` — 单股指定定期报告

```bash
cninfo fetch-report <code> --year <Y> --kind <annual|q1|h1|q3> [--plate sz|sh|bj] [--force]
```

| 参数 | 必填 | 说明 |
|---|---|---|
| `code` | 是 | 6 位股票代码 |
| `--year` | 是 | 报告所属财年，整数 |
| `--kind` | 是 | `annual` / `q1` / `h1` / `q3` |
| `--plate` | 否 | 不传则按代码首位推测 |
| `--force` | 否 | 忽略缓存重新下载 |

**流程**：推测/采用板块 → `find_periodic_report` → `fetch_announcement` → 输出 `FetchResult.to_dict()`。

**输出字段**：`ann_id`、`sec_code`、`ts_code`、`ann_date`、`title`、`category`、`pdf_url`、`pdf_path`、`md_path`、`meta_path`、`total_pages`、`extracted_pages`、`text_chars`、`cache_hit`。

**失败**：

| 情况 | 退出码 | 信息 |
|---|---|---|
| orgId 查不到 | 1 | `error: orgId not found ...` |
| 无匹配报告本体 | 2 | `error: no <kind> report body found for <code> year=<Y> (plate=<plate>)` |
| 代码非法导致 `guess_plate` 抛错 | 未捕获 traceback | — |

```bash
cninfo fetch-report 600519 --year 2024 --kind annual
cninfo fetch-report 301580 --year 2024 --kind h1 --plate sz
```

---

## 3. `fetch-stock` — 单股时间窗全公告

```bash
cninfo fetch-stock <code> --since <YYYY-MM-DD> --until <YYYY-MM-DD> \
  [--plate ...] [--kind ...] [--download] [--force] [--json]
```

| 参数 | 必填 | 说明 |
|---|---|---|
| `code` | 是 | 6 位代码 |
| `--since` / `--until` | 是 | 日期区间 |
| `--plate` | 否 | 默认推测 |
| `--kind` | 否 | 仅过滤某类定期报告的 category |
| `--download` | 否 | 逐条下载并解析；默认只列列表 |
| `--force` | 否 | 配合 `--download` 强制重下 |
| `--json` | 否 | JSON 输出 |

**行字段**

| 模式 | 字段 |
|---|---|
| 仅列表 | `ann_date`、`ann_id`、`title`、`adjunct_size_kb` |
| `--download` | 追加 `cache_hit`、`md_path`、`pages`、`text_chars` |

**注意**

- 没有 `--limit`，会翻完整个时间窗；公告多时请求量较大。
- `--force` 不带 `--download` 无实际作用。
- 表格在 `--download` 时展示 `cache_hit`、`pages`、`text_chars`，`md_path` 只在 JSON 中可见。

```bash
cninfo fetch-stock 600519 --since 2024-04-01 --until 2024-04-30
cninfo fetch-stock 600519 --since 2024-04-01 --until 2024-04-30 --download --json
```

---

## 4. `list` — 全市场切片

```bash
cninfo list --plate <sz|sh|bj> [--category ...] [--date D | --since D --until D] \
  [--keyword K] [--limit N] [--download] [--force] [--json]
```

| 参数 | 必填 | 说明 |
|---|---|---|
| `--plate` | 是 | 板块 |
| `--category` | 否 | `annual/q1/h1/q3` 别名，或原生 `category_xxx_szsh` |
| `--date` | 否 | 单日；与 `--since/--until` 二选一 |
| `--since` / `--until` | 否 | 区间 |
| `--keyword` | 否 | cninfo `searchkey` 模糊匹配标题 |
| `--limit` | 否 | 最多条数，`0` 或省略表示不限 |
| `--download` | 否 | 逐条下载并解析 |
| `--force` | 否 | 配合下载 |
| `--json` | 否 | JSON 输出 |

**category 解析**：若值在 `KIND_TO_CATEGORY` 中则映射为原生字符串，否则原样透传。

**行字段**：`ann_date`、`sec_code`、`sec_name`、`ann_id`、`title`；下载后追加 `cache_hit`、`md_path`。

**边界**

- 不传日期/区间时，`iter_market_slice` 抛 `ValueError`，CLI **不捕获**，会打印 traceback。
- 不需要 orgId，因此不会触发 topSearch。
- `--limit` 在满足条数后立即 `break`，可显著减少请求数。

```bash
cninfo list --plate sz --category annual --date 2024-04-29 --limit 20
cninfo list --plate sh --since 2024-04-01 --until 2024-04-30 --keyword "重大资产重组"
```

---

## 5. `search` — 标签 / 关键词搜索

```bash
# 按股票
cninfo search --stock <code> --since D --until D [--plate ...] [--type T] [--sub-type ST] ...

# 按全市场
cninfo search --plate <sz|sh|bj> [--date D | --since D --until D] [--type T] [--sub-type ST] ...

# 通用
[--keyword K] [--limit N] [--download] [--force] [--json]
```

| 参数 | 说明 |
|---|---|
| `--stock` | 6 位代码；此时 `--since` 与 `--until` 必填 |
| `--plate` | 股票模式下可选（默认推测）；全市场模式下必填 |
| `--date` / `--since` / `--until` | 全市场模式的时间范围 |
| `--type` | `argus_legal` 大类，如 `shareholder` |
| `--sub-type` | `argus_legal` 子类，如 `reduce_plan` |
| `--keyword` | 标题关键词 |
| `--limit` | 最多条数 |
| `--download` | 逐条下载 |
| `--force` | 配合下载 |
| `--json` | JSON 输出 |

### 5.1 两种过滤路径

| 条件 | 数据源 | 过滤 |
|---|---|---|
| 指定 `--type` 或 `--sub-type` | stock 或 market 切片 | 调 `classify(title, ann_date)`，精确比较 `type` / `sub_type` |
| 仅 `--keyword` | market：cninfo `searchkey`；stock：客户端 | 客户端再做一次 `keyword in title` |
| 两者都有 | 先标签过滤，再关键词过滤 | 两种条件同时满足 |

**注意**

- `--keyword` 在 **stock 模式**下不会传给 cninfo，只做客户端过滤；在 **market 模式**下先作为 `searchkey` 传给 cninfo，再客户端二次过滤。
- `--date` 在 stock 模式下被忽略；stock 模式要求 `--since` + `--until`。
- `use_classifier` 为假时，输出列不包含 `type/sub_type`。

### 5.2 分类器缺失时的行为

`cli.py` 中 `from cninfo.classify import classify` 是**模块导入**，模块本身可以成功导入（`argus_legal` 是函数内动态导入）。真正的失败发生在首次调用 `classify()`，抛 `ClassifierUnavailable`（`RuntimeError` 子类），被外层 `except Exception` 捕获：

```text
error: classifier error: announcement_filter not installed. Install with:
    git clone https://github.com/rollysys/announcement_filter.git
    cd announcement_filter && pip install -e .
```

退出码 1。未使用 `--type/--sub-type` 的其它命令不受影响。

### 5.3 输出列

基础：`ann_date`、`sec_code`、`sec_name`、`ann_id`、`title`；分类模式追加 `type`、`sub_type`；下载模式追加 `cache_hit`、`md_path`。

```bash
cninfo search --stock 600519 --since 2024-01-01 --until 2024-12-31 \
              --type shareholder --sub-type reduce_plan
cninfo search --plate sz --date 2024-04-29 --keyword "重组" --limit 10
```

---

## 6. `cache` — 缓存管理

### 6.1 `cache stats`

```bash
cninfo cache stats
```

输出 JSON：

```json
{
  "cache_root": "/home/user/.cache/cninfo",
  "pdf_count": 12,
  "md_count": 12,
  "meta_count": 12,
  "orgid_count": 5,
  "pdf_bytes": 12345678,
  "md_bytes": 345678,
  "meta_bytes": 234567,
  "total_bytes": 12932823,
  "total_human": "12.3MB"
}
```

退出码 0。

### 6.2 `cache verify`

```bash
cninfo cache verify
```

- 一致：打印 `ok: cache is consistent`，退出码 **0**。
- 不一致：每条告警打印 `warn: ...`，退出码 **3**。

告警类型：`meta but no pdf` / `meta but no md` / `pdf but no meta` / `md but no meta`。

### 6.3 `cache prune`

```bash
cninfo cache prune [--older-than 365d]
```

| 参数 | 默认 | 说明 |
|---|---|---|
| `--older-than` | `365` | 支持 `365d`、`12h`、`30`；解析后按天，最小 1 |

`_parse_older_than` 规则：

- 后缀 `d` → 天数；
- 后缀 `h` → `max(1, 小时数 // 24)`（不足 1 天按 1 天）；
- 纯数字 → 天数；
- 结果最小 1。

输出 `pruned N announcement(s) older than Nd`，退出码 0。删除基准是 meta 文件 mtime。

---

## 7. `orgid` — 查 secCode → orgId

```bash
cninfo orgid <code> [--refresh] [--cache-only]
```

| 参数 | 说明 |
|---|---|
| `code` | 6 位代码 |
| `--refresh` | 忽略缓存，强制请求 topSearch |
| `--cache-only` | 只读缓存，不联网 |

**分支优先级**：`--refresh` 先判断并直接执行；否则走 `get_orgid(fetch_if_missing=not cache_only)`。

输出：

```json
// 普通 / cache-only
{"sec_code": "600519", "orgId": "gssh0600519"}

// refresh
{"sec_code": "600519", "orgId": "gssh0600519", "source": "cninfo-topSearch"}
```

失败（查不到 / cache-only 缺失）→ stderr + 退出码 1。

```bash
cninfo orgid 600519
cninfo orgid 600519 --cache-only
cninfo orgid 600519 --refresh
```

---

## 8. 错误处理汇总

| 命令 | 捕获的异常 | 未捕获（会 traceback） |
|---|---|---|
| `fetch-report` | `OrgIdNotFound` | `guess_plate` 的 `ValueError`、PDF/HTTP、PyMuPDF |
| `fetch-stock` | `OrgIdNotFound` | `guess_plate` 的 `ValueError`、HTTP、PDF 解析 |
| `list` | 无 | 缺日期范围的 `ValueError`、HTTP |
| `search` | `OrgIdNotFound`（仅构造迭代器时）、`classify` 的任意 `Exception` | 参数校验后的 HTTP、`guess_plate` 的 `ValueError` |
| `cache` | 无 | 文件权限/JSON 损坏（prune 读取 meta） |
| `orgid` | `OrgIdNotFound` | HTTP 错误、JSON 解析错误 |

建议在脚本中同时检查退出码与 stderr，避免把 argparse 的用法错误（码 2）误判为“报告未找到”。

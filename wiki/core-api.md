# 核心 API 层详解：`api.py` 与 `parser.py`

本页覆盖两个“无状态”模块：cninfo 接口适配（[api.py](../src/cninfo/api.py)）与 PDF 解析（[parser.py](../src/cninfo/parser.py)）。

---

## 1. `api.py`

### 1.1 常量

| 常量 | 值 | 说明 |
|---|---|---|
| `CNINFO_QUERY` | `http://www.cninfo.com.cn/new/hisAnnouncement/query` | 公告查询端点 |
| `PDF_BASE` | `http://static.cninfo.com.cn/` | PDF 静态文件前缀 |
| `DEFAULT_HEADERS` | 浏览器 UA + form-urlencoded + AJAX 头 | 不加 UA 可能被简单 throttle |
| `CATEGORY_NDBG` | `category_ndbg_szsh` | 年度报告 |
| `CATEGORY_YJDBG` | `category_yjdbg_szsh` | 一季度报告 |
| `CATEGORY_BNDBG` | `category_bndbg_szsh` | 半年度报告 |
| `CATEGORY_SJDBG` | `category_sjdbg_szsh` | 三季度报告 |
| `KIND_TO_CATEGORY` | `{annual, q1, h1, q3} → 上述 category` | CLI 的 `--kind/--category` 别名映射 |
| `KIND_TO_TITLE_TAIL` | `annual→"年年度报告"` 等 | 标题尾部匹配用 |
| `BEIJING` | `timezone(timedelta(hours=8))` | 北京时间 |
| `PLATE_TO_SUFFIX` | `{sz: SZ, sh: SH, bj: BJ}` | `ts_code` 后缀 |

### 1.2 `query_page(...) -> dict`

```python
query_page(
    *,
    plate: str = "",
    category: str = "",
    se_date: str = "",
    stock: str = "",
    searchkey: str = "",
    page_num: int = 1,
    column: str = "szse",
    timeout: float = 15.0,
    session: requests.Session | None = None,
) -> dict
```

单页查询，返回 cninfo 原始 JSON。

**请求体固定字段**

| 字段 | 值 | 备注 |
|---|---|---|
| `tabName` | `fulltext` | 固定 |
| `pageSize` | `"30"` | **服务端硬限，改大无效** |
| `pageNum` | `str(page_num)` | 从 1 起 |
| `column` | 参数 | `szse` / `sse` |
| `category` | 参数 | 原生 category 字符串 |
| `plate` | 参数 | `sz` / `sh` / `bj` |
| `searchkey` | 参数 | 标题 LIKE |
| `seDate` | 参数 | 格式 `YYYY-MM-DD~YYYY-MM-DD` |
| `stock` | 参数 | 必须 `<6位>,<orgId>` |
| `isHLtitle` | `"true"` | 命中词用 `<em>` 包裹 |

`secid` / `trade` / `sortName` / `sortType` 传空字符串，默认按时间倒序。

**行为与异常**

- 使用 `session or requests` 发送 POST；若传入 `requests.Session` 可复用连接。
- 调用 `raise_for_status()`，非 2xx 抛 `requests.HTTPError`。
- 返回 `r.json()`，可能抛 `requests.JSONDecodeError`。
- 不重试、不 backoff、不做限速（限速在 `query_all`）。

### 1.3 `query_all(...) -> Iterator[dict]`

```python
query_all(
    *,
    plate: str = "",
    category: str = "",
    se_date: str = "",
    stock: str = "",
    searchkey: str = "",
    column: str = "szse",
    sleep_seconds: float = 0.4,
    max_pages: int = 1000,
    session: requests.Session | None = None,
) -> Iterator[dict]
```

翻页迭代器，逐条 `yield` `announcements` 数组中的对象。

**终止条件（任一）**

1. 当前页 `announcements` 为空 → 直接 `return`；
2. `hasMore` 为假 → `return`；
3. 达到 `max_pages`（默认 1000 页，即约 3 万条上限）。

每翻一页（且还会继续时）执行 `time.sleep(sleep_seconds)`，默认 0.4s，用于降低触发反爬的概率。`sleep_seconds=0` 可关闭。

> 注意：`max_pages=1000 × 30 = 30,000`，足以覆盖“单日全市场约 2.6 万条”的场景；更大的时间窗需要显式调大。

### 1.4 字段清洗函数

#### `epoch_ms_to_ann_date(ts_ms: int) -> str`

UTC epoch 毫秒 → 北京时间 `YYYYMMDD`。跨日公告若按 UTC 取日期会早一天，因此必须走本函数。

```python
epoch_ms_to_ann_date(1712880000000)  # "20240412"
```

#### `clean_title(raw: str | None) -> str`

去除 cninfo 在 `searchkey` 命中时插入的 `<em>...</em>` 标签，并 `strip()`。`None` 视为空串。

```python
clean_title("<em>贵州茅台</em>2024年年度报告")  # "贵州茅台2024年年度报告"
```

#### `adjunct_to_url(adjunct_path: str) -> str`

相对路径拼接 `PDF_BASE`，自动去掉开头多余的 `/`。

```python
adjunct_to_url("/finalpage/2024-04-12/123.PDF")
# "http://static.cninfo.com.cn/finalpage/2024-04-12/123.PDF"
```

### 1.5 定期报告判定

#### `is_periodic_report_body(title: str) -> bool`

判断是否为任一类型的定期报告**本体**：

1. 先 `clean_title`；
2. `endswith("摘要")` → 否；
3. 含任一排除词 → 否：`审计报告`、`内部控制`、`提示性公告`、`披露`、`鉴证报告`；
4. 正则 `\d{4}年(年度|第一季度|半年度|第三季度)报告$` 命中 → 是。

#### `is_kind_report_body(title: str, year: int, kind: str) -> bool`

在上一函数基础上进一步限定**年份**与**类型**：

- 排除规则相同；
- `kind` 必须在 `KIND_TO_TITLE_TAIL` 中，否则返回 `False`；
- 标题必须以 `f"{year}{tail}"` 结尾，例如 `2024年年度报告`。

| kind | 匹配尾部 |
|---|---|
| `annual` | `2024年年度报告` |
| `q1` | `2024年第一季度报告` |
| `h1` | `2024年半年度报告` |
| `q3` | `2024年第三季度报告` |

> 标题前缀不影响匹配（如 `贵州茅台2024年年度报告`），因为只检查尾部。
> 同一份报告可能存在两条记录（标题一带前缀一带不带），本函数不负责去重。

### 1.6 代码与板块转换

#### `to_ts_code(sec_code: str, plate: str) -> str`

`("301580", "sz") → "301580.SZ"`。未知 `plate` 抛 `ValueError`。

#### `guess_plate(sec_code: str) -> str`

按首位粗略推测板块：

| 首位 | 板块 |
|---|---|
| `0` / `3` | `sz`（深主板 / 创业板） |
| `6` | `sh`（沪主板 / 科创板） |
| `4` / `8` / `9` | `bj`（北交所） |

非法输入（非 6 位、非数字、首位不在上述集合）抛 `ValueError`。这是启发式规则，严谨场景应显式传 `--plate`。

### 1.7 `fetch_pdf_bytes(...) -> bytes`

```python
fetch_pdf_bytes(url: str, *, timeout: float = 60.0, session=None) -> bytes
```

GET PDF 直链，仅携带 `User-Agent`，`raise_for_status()` 后返回 `r.content`。老 URL 可能 404；本函数不重试。

---

## 2. `parser.py`

### 2.1 `ParsedPdf` 数据类

| 字段/属性 | 类型 | 说明 |
|---|---|---|
| `text` | `str` | 所有页面文本以 `"\n\n"` 连接 |
| `total_pages` | `int` | PDF 总页数 |
| `extracted_pages` | `int` | 至少提取到非空白文本的页数 |
| `text_chars`（property） | `int` | `len(text)` |
| `is_scanned`（property） | `bool` | `total_pages > 0 and extracted_pages == 0` |

### 2.2 `parse_pdf_bytes(data: bytes) -> ParsedPdf`

- 用 `fitz.open(stream=data, filetype="pdf")` 打开内存中的 PDF。
- 逐页 `page.get_text()`；`t.strip()` 非空则 `extracted_pages += 1`。
- `finally` 中关闭文档，避免文件句柄泄漏。
- 非 PDF / 损坏文件会抛 PyMuPDF 异常。

### 2.3 `parse_pdf_file(path: str) -> ParsedPdf`

读取磁盘文件为 bytes，转调 `parse_pdf_bytes`。

### 2.4 扫描件语义

部分 PDF（尤其律所审计意见、内部控制鉴证报告）是图片扫描件，`extracted_pages=0`、`text_chars=0`。此时：

- PDF 仍会被缓存；
- Markdown 正文为空，但 frontmatter 与 meta 仍完整；
- 本工具**不内置 OCR**。

---

## 3. cninfo 接口约束速查

这些是上游硬约束，代码中已适配，不要尝试“修复”：

| 约束 | 影响 | 代码对策 |
|---|---|---|
| `pageSize` 固定 30 | 全量必须翻页 | `query_all` |
| `stock` 必须 `<6位>,<orgId>` | 只传 6 位返回 0 条 | `orgid.stock_param` |
| `announcementTime` 是 UTC epoch ms | 日期可能差一天 | `epoch_ms_to_ann_date` |
| `searchkey` 是模糊 LIKE | 不能精确按代码查 | 客户端二次过滤 / 精确场景用 `stock` |
| 标题含 `<em>` | 文件名会带标签 | `clean_title` |
| 同一公告可能重复 | 列表出现两条 | 调用方按 `(secCode, ann_date, title/adjunctSize)` 去重（未内置） |
| 2024-06 前部分 PDF 可能 404 | 下载失败 | 未内置重试（可后续补） |

请求/响应完整字段说明见 [docs/api_reference.md](../docs/api_reference.md)。

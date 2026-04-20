---
name: sql-data-export
display_name: SQL 数据导出
description: 把 execute_select 查询结果导出成 CSV / Excel / JSON 保存到 Claude Code 本地。适合处理大结果集（> 100 行）、交付报表、二次分析。MCP 侧只读（L1），文件写在 Claude Code 本机。
user-invocable: true
allowed-tools: Bash,Write,mcp__argus__list_agents,mcp__argus__get_agent_config,mcp__argus__execute_select
---

# SQL 数据导出

## 适用场景

- **大结果集**：超过 50 行的数据在聊天里看不方便，直接导成 CSV
- **交付报表**：甲方要"给我一份上周的订单数据" → 导出 xlsx 发过去
- **二次分析**：要在 Excel / Python / SQLite 里做交叉分析

## 流程

### 1. 定位目标
```
list_agents → 找有目标库的 agent
get_agent_config(agent_id) → 看 sql_presets 里有啥 DSN
```

### 2. 明确需求
问清楚：
- **要哪些字段**（别默认 `SELECT *`，往往包含大 JSON 字段，影响性能）
- **时间范围**（近 N 天 / 年月区间）
- **预期行数**（用户说"全部"，先 `SELECT COUNT(*)` 确认规模）
- **格式**：CSV（最通用）/ xlsx（Excel 直接打开）/ JSON（程序处理）

### 3. 先探规模

```python
execute_select(
  agent_id=..., preset=...,
  query="SELECT COUNT(*) FROM orders WHERE created_at > '2026-01-01'"
)
```

- <= 1000 行：可以直接一把拉
- 1000 - 100000：分页拉（LIMIT / OFFSET）
- > 100000：讨论是否真需要全量，或提取 summary / aggregation 代替
- 大字段（JSON / BLOB）：明确列出要的字段，别 SELECT *

### 4. 拉取数据

execute_select 有默认 limit（100），大结果集要显式指定：

```python
execute_select(
  agent_id=..., preset=...,
  query="SELECT id, name, price, created_at FROM orders WHERE created_at > '2026-01-01' ORDER BY id",
  limit=10000,
  timeout=120
)
```

若超过 10000 行，分批：

```python
batch_size = 5000
offset = 0
while True:
    rows = execute_select(..., query=f"SELECT ... ORDER BY id LIMIT {batch_size} OFFSET {offset}")
    if not rows: break
    # 累积到本地
    offset += batch_size
```

### 5. 写本地文件

用 Claude Code 的 Write 工具直接写 CSV：

```
Write(
  file_path="/tmp/orders-2026Q1.csv",
  content="id,name,price,created_at\n1,iPhone,999,2026-01-03\n..."
)
```

或用 Bash 工具跑 Python 脚本生成 xlsx：

```python
import pandas as pd
df = pd.DataFrame(rows, columns=columns)
df.to_excel('/tmp/orders-2026Q1.xlsx', index=False)
```

### 6. 告诉用户路径

```
已导出 2340 行到 /tmp/orders-2026Q1.csv (238 KB)
Excel 版本: /tmp/orders-2026Q1.xlsx
```

## 格式选择

| 格式 | 何时用 | 写法 |
|------|--------|------|
| CSV | 最通用，Excel / 数据库 / Python 都能读 | Write 直接拼字符串 |
| TSV | 字段里有逗号时，避免转义麻烦 | 同上，分隔符换 `\t` |
| JSON | 字段含嵌套结构（JSONB 列、列表） | `json.dumps(rows, ensure_ascii=False)` |
| xlsx | 要给非技术用户看（直接双击打开） | `pandas.to_excel` 或 `openpyxl` |
| Markdown 表 | 行数少，用户只在聊天里看 | 直接 Claude 输出 |

## CSV 转义细节

```python
import csv
with open(path, 'w', newline='', encoding='utf-8-sig') as f:
    w = csv.writer(f)
    w.writerow(columns)
    w.writerows(rows)
```

- `utf-8-sig` 带 BOM，Excel 中文不乱码
- 用 `csv.writer` 自动处理逗号、引号、换行转义
- 别手写 `row.join(',')`，会炸（字段里有逗号就错）

## Excel 格式化（xlsx）

```python
import pandas as pd
from openpyxl.utils import get_column_letter

df = pd.DataFrame(rows, columns=columns)
with pd.ExcelWriter(path, engine='openpyxl') as w:
    df.to_excel(w, sheet_name='orders', index=False)
    ws = w.sheets['orders']
    # 自动列宽
    for i, col in enumerate(columns):
        max_len = max(df[col].astype(str).map(len).max(), len(col))
        ws.column_dimensions[get_column_letter(i+1)].width = min(max_len + 2, 50)
```

## 常见坑

- **不要 SELECT \***：产线库经常有 JSON 字段一列几 KB，几万行就爆
- **JSON 字段**：在 SQL 里 `JSON_EXTRACT` 或 `-> ` 把要的字段抠出来，别整个拉
- **LIMIT + ORDER**：分批拉要有 ORDER BY，否则每批数据可能重叠
- **时区**：`created_at` 是服务器本地时间？UTC？先问清再拉
- **大结果超时**：`timeout=300` 是上限，更大就只能分批
- **execute_select 的默认 limit 是 100**，显式传 `limit=N` 才能拉更多
- Claude Code 里写文件用 `/tmp/` 或用户目录；要交付给用户时让用户指定路径

## 和 Argus Console 的对比

Argus Console → Dashboard → SQL 模块 本身就有"CSV 导出 / Excel 导出 / 全部 CSV 流式导出"功能（v1.27 加的）。**如果用户在 Console 里手动查**，直接点导出按钮最方便。

本 skill 面向的是 **AI 对话里要结构化导出**的场景 —— 用户描述需求、AI 写 SQL、AI 导出、AI 交付文件路径。

## 关联 skill

- `agent-inventory` — 定位目标 agent
- `sql-query` — 基础 SQL 查询（小结果集）
- `bulk-ops` — 批量跨机器数据汇总

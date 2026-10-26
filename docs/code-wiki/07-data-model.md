# 07 · 数据模型

系统的数据库设计遵循“**表结构字典驱动**”：所有表的名称、中文名、字段、类型、列宽都集中在 [tablestructure.py](file:///workspace/instock/core/tablestructure.py)。作业写入、Web 展示、K 线绘图都从这里取字段。

---

## 1. 数据库概览

- 数据库名：`instockdb`（可通过 `db_database` 覆盖）
- 字符集/排序：`utf8mb4` / `utf8mb4_general_ci`
- 建库建表：
  - `init_job.py` 只创建库与 `cn_stock_attention`；
  - 其余表在首次 `insert_db_from_df` 时由 `pandas.to_sql(if_exists='append')` **自动创建**；
  - 插入后若表无主键，会自动 `ALTER TABLE ADD PRIMARY KEY (...)`（通常是 `date + code`）。

大多数表都包含三列基础字段（来自 `TABLE_CN_STOCK_FOREIGN_KEY`）：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `date` | `DATE` | 交易日期 |
| `code` | `VARCHAR(6)` | 股票代码 |
| `name` | `VARCHAR(20)` | 股票名称 |

---

## 2. 物理表清单

### 2.1 基础行情

| 表名 | 中文名 | 关键字段 | 写入作业 |
| --- | --- | --- | --- |
| `cn_stock_spot` | 每日股票数据 | `new_price`、`change_rate`、`volume`、`deal_amount`、`turnoverrate`、`pe9`、`pbnewmrq`、`roe_weight` 等 40+ 列 | `basic_data_daily_job` |
| `cn_etf_spot` | 每日ETF数据 | `new_price`、`change_rate`、`total_market_cap`、`free_cap` 等 | `basic_data_daily_job` |
| `cn_stock_selection` | 综合选股 | 由东财选股器返回的 200+ 指标（`CONCEPT`、`STYLE` 等） | `selection_data_daily_job` |
| `cn_stock_spot_buy` | 基本面选股 | `pe9`、`pbnewmrq`、`roe_weight` 等 | `basic_data_other_daily_job.stock_spot_buy` |

### 2.2 资金与消息面

| 表名 | 中文名 | 说明 |
| --- | --- | --- |
| `cn_stock_fund_flow` | 股票资金流向 | 4 个周期（今日/3日/5日/10日）合并到同一行 |
| `cn_stock_fund_flow_industry` | 行业资金流向 | 按 `name`（行业名）为主键之一 |
| `cn_stock_fund_flow_concept` | 概念资金流向 | 按 `name`（概念名）为主键之一 |
| `cn_stock_bonus` | 股票分红配送 | 报告期由 `get_bonus_report_date()` 决定 |
| `cn_stock_lhb` | 股票龙虎榜 | 东方财富，主键 `date+code` |
| `cn_stock_top` | 股票龙虎榜(新浪) | 新浪统计口径 |
| `cn_stock_blocktrade` | 股票大宗交易 | 收盘后 1~2 小时才有数据 |
| `cn_stock_chip_race_open` | 早盘抢筹数据 | 按 `bid_trust_amount` 排序 |
| `cn_stock_chip_race_end` | 尾盘抢筹数据 | 按 `bid_trust_amount` 排序 |
| `cn_stock_limitup_reason` | 涨停原因揭密 | `title`/`reason` |

### 2.3 计算与选股

| 表名 | 中文名 | 说明 |
| --- | --- | --- |
| `cn_stock_indicators` | 股票指标数据 | 外键 3 列 + `STOCK_STATS_DATA` 全部指标列 |
| `cn_stock_indicators_buy` | 股票指标买入 | 外键 + `rate_1…rate_100` |
| `cn_stock_indicators_sell` | 股票指标卖出 | 外键 + `rate_1…rate_100` |
| `cn_stock_pattern` | 股票K线形态 | 外键 3 列 + 61 个形态字段（`-1/0/1`） |
| `cn_stock_strategy_*` | 10 张策略表 | 每张 = 外键 + `rate_1…rate_100` |

策略表完整列表：

| 表名 | 中文名 |
| --- | --- |
| `cn_stock_strategy_enter` | 放量上涨 |
| `cn_stock_strategy_keep_increasing` | 均线多头 |
| `cn_stock_strategy_parking_apron` | 停机坪 |
| `cn_stock_strategy_backtrace_ma250` | 回踩年线 |
| `cn_stock_strategy_breakthrough_platform` | 突破平台 |
| `cn_stock_strategy_low_backtrace_increase` | 无大幅回撤 |
| `cn_stock_strategy_turtle_trade` | 海龟交易法则 |
| `cn_stock_strategy_high_tight_flag` | 高而窄的旗形 |
| `cn_stock_strategy_climax_limitdown` | 放量跌停 |
| `cn_stock_strategy_low_atr` | 低 ATR 成长 |

### 2.4 用户与预留

| 表名 | 中文名 | 说明 |
| --- | --- | --- |
| `cn_stock_attention` | 我的关注 | `datetime` + `code`，Web 端维护，用于各页面置顶/标红 |
| `cn_stock_cpbd` | 操盘必读 | 字段已定义，当前无作业写入（预留） |

---

## 3. 逻辑模板（不是物理表）

这些常量只用于生成列定义，本身不直接对应一张表：

| 常量 | 用途 |
| --- | --- |
| `TABLE_CN_STOCK_FOREIGN_KEY` | 通用外键三列 `date/code/name` |
| `STOCK_STATS_DATA` | 指标列模板（约 70 列） |
| `STOCK_KLINE_PATTERN_DATA` | 形态列模板（61 列，含 `func`） |
| `TABLE_CN_STOCK_BACKTEST_DATA` | 回测列模板 `rate_1 … rate_100` |
| `CN_STOCK_HIST_DATA` | 历史行情列模板（`date/open/close/high/low/volume/amount/...`） |
| `CN_STOCK_FUND_FLOW` | 资金流周期与列映射 |
| `CN_STOCK_SECTOR_FUND_FLOW` | 板块类型与指标映射 |

---

## 4. 字段定义结构

每个字段是一个字典：

```python
'new_price': {'type': FLOAT, 'cn': '最新价', 'size': 70}
```

| 键 | 含义 | 使用方 |
| --- | --- | --- |
| `type` | SQLAlchemy 类型（`DATE/FLOAT/BIGINT/VARCHAR/SmallInteger/BIT…`） | `database.insert_db_from_df` |
| `cn` | 中文列名 | Web 列头、图表图例 |
| `size` | 列宽（px） | SpreadJS |
| `map` | 数据源代码字段（仅综合选股使用） | `stock_selection.py` |
| `func` | 形态识别函数（仅形态列） | `pattern_recognitions.py` |

辅助函数：

| 函数 | 作用 |
| --- | --- |
| `get_field_cn(key, table)` | 取单个字段中文名 |
| `get_field_cns(cols)` | 生成 SpreadJS 列头（含涨跌幅红绿条件格式） |
| `get_field_types(cols)` | 生成 `{col: type}` 供 `to_sql(dtype=...)` |
| `get_field_type_name(col_type)` | 类型归类为 `datetime/numeric/string` |

---

## 5. 表之间的关系

```
cn_stock_spot (当日快照)
      │ code
      ▼
cn_stock_attention (关注)  ── 用于各表 ORDER BY 置顶
      │
      ▼
cn_stock_indicators ──► cn_stock_indicators_buy / sell
      │                        │
      │                        ▼
cn_stock_pattern         rate_1…rate_100（回测回填）
      │
cn_stock_strategy_* ─────────┘

cn_stock_selection（独立：东财选股器全字段）
cn_stock_fund_flow(_industry/_concept)（独立）
cn_stock_lhb / cn_stock_top / cn_stock_blocktrade（独立）
```

说明：

- `date + code` 是绝大多数表的联合主键；
- 行业/概念资金流使用 `date + name`；
- 策略/指标买卖表通过 `rate_*` 字段与回测作业关联；
- `cn_stock_attention` 被各页面用于 `ORDER BY cdatetime DESC` 实现“关注置顶”。

---

## 6. 回测字段语义

`TABLE_CN_STOCK_BACKTEST_DATA` 由 `RATE_FIELDS_COUNT = 100` 动态生成：

```python
'rate_%s' % i for i in range(1, 101)
```

- `rate_n` = 选股日后第 n 个交易日相对选股日收盘价的涨跌幅（%）；
- `rate_1` 为次日收益，依次向后；
- 回测作业通过 `get_rates()` 计算并 `update_db_from_df` 回填。

---

## 7. 常见运维 SQL

```sql
-- 查看当日选出的策略股票
SELECT date, code, name FROM cn_stock_strategy_enter WHERE date = '2024-01-02';

-- 查看某只股票的指标
SELECT * FROM cn_stock_indicators WHERE code = '600000' ORDER BY date DESC LIMIT 20;

-- 查看关注列表
SELECT * FROM cn_stock_attention;

-- 清理某天数据（重跑前）
DELETE FROM cn_stock_spot WHERE date = '2024-01-02';
```

> 重跑单个作业是安全的：每个保存函数都会先按 `date` 删除旧数据。

# 03 · job 模块（数据作业）

`instock/job/` 负责把 `core` 的抓取/计算能力串成可批量执行的作业。所有作业都遵循同一套约定，便于扩展。

---

## 1. 作业清单

| 文件 | `main()` 做什么 | 产出表 |
| --- | --- | --- |
| [init_job.py](file:///workspace/instock/job/init_job.py) | 检查数据库，不存在则创建库与 `cn_stock_attention` 表 | 数据库、关注表 |
| [basic_data_daily_job.py](file:///workspace/instock/job/basic_data_daily_job.py) | 实时股票行情、实时 ETF 行情 | `cn_stock_spot`、`cn_etf_spot` |
| [selection_data_daily_job.py](file:///workspace/instock/job/selection_data_daily_job.py) | 东方财富综合选股 | `cn_stock_selection` |
| [basic_data_other_daily_job.py](file:///workspace/instock/job/basic_data_other_daily_job.py) | 龙虎榜、分红、个股/板块资金流、早盘抢筹、涨停原因、基本面选股 | 多张表 |
| [indicators_data_daily_job.py](file:///workspace/instock/job/indicators_data_daily_job.py) | 计算指标并二次筛选买卖点 | `cn_stock_indicators(_buy/_sell)` |
| [klinepattern_data_daily_job.py](file:///workspace/instock/job/klinepattern_data_daily_job.py) | K 线形态识别 | `cn_stock_pattern` |
| [strategy_data_daily_job.py](file:///workspace/instock/job/strategy_data_daily_job.py) | 遍历全部策略选股 | `cn_stock_strategy_*` |
| [backtest_data_daily_job.py](file:///workspace/instock/job/backtest_data_daily_job.py) | 回填未来收益率 | 原选股表的 `rate_*` 字段 |
| [basic_data_after_close_daily_job.py](file:///workspace/instock/job/basic_data_after_close_daily_job.py) | 大宗交易、尾盘抢筹 | `cn_stock_blocktrade`、`cn_stock_chip_race_end` |
| [execute_daily_job.py](file:///workspace/instock/job/execute_daily_job.py) | 整体编排入口 | — |

---

## 2. 通用作业模板

每个“保存类”作业都长这样（以实时行情为例）：

```python
def save_nph_stock_spot_data(date, before=True):
    if before:
        return                                   # 批量历史模式下跳过实时数据
    data = stock_data(date).get_data()          # 取数
    table_name = tbs.TABLE_CN_STOCK_SPOT['name']
    if mdb.checkTableIsExist(table_name):       # 幂等：先删当日
        mdb.executeSql(f"DELETE FROM `{table_name}` where `date` = '{date}'")
        cols_type = None
    else:
        cols_type = tbs.get_field_types(...)    # 首次建表用精确类型
    mdb.insert_db_from_df(data, table_name, cols_type, False, "`date`,`code`")
```

要点：

- **`before` 参数**：`save_nph_*`（实时类）在批量历史模式下会被跳过；当前时间模式由 `run_with_args` 显式传 `False`。
- **幂等**：先按 `date` 删除，再插入，可安全重跑。
- **类型推断**：表已存在时 `cols_type=None`，让 pandas 自动推断；首次创建时用 `tablestructure` 中的真实类型。
- **主键**：最后一个参数 `"`date`,`code`"` 用于在插入后补主键。

---

## 3. 批量执行模板 `run_with_args`

[run_template.py](file:///workspace/instock/lib/run_template.py) 根据 `sys.argv` 长度决定运行模式：

| 命令行 | 模式 | 行为 |
| --- | --- | --- |
| `python x.py` | 当前时间 | 计算 `run_date` / `run_date_nph`，按函数名前缀分发 |
| `python x.py 2023-03-01` | 单/枚举日期 | 按逗号拆分，逐个日期提交线程池 |
| `python x.py 2023-03-01 2023-03-21` | 区间日期 | 逐日遍历，仅交易日提交 |

```python
def run_with_args(run_fun, *args):
    if len(sys.argv) == 3:      # 区间
        ... executor.submit(run_fun, run_date, *args)
    elif len(sys.argv) == 2:    # 枚举
        ... executor.submit(run_fun, run_date, *args)
    else:                       # 当前
        run_date, run_date_nph = trd.get_trade_date_last()
        if run_fun.__name__.startswith('save_nph'):
            run_fun(run_date_nph, False)     # 实时数据
        elif run_fun.__name__.startswith('save_after_close'):
            run_fun(run_date, *args)         # 闭盘数据
        else:
            run_fun(run_date_nph, *args)
```

注意：

- **批量模式（指定日期）下实时类函数不会执行**（`before` 默认 `True`），这是刻意的设计——历史日期拿不到“实时”快照。
- 日期判断依赖 `lib.trade_time.is_trade_date`，非交易日自动跳过。
- 区间模式每个日期 `time.sleep(2)`，降低抓取压力。

---

## 4. 整体编排 `execute_daily_job.py`

```python
def main():
    bj.main()                          # 1. 建库
    hdj.main()                         # 2.1 实时行情
    sddj.main()                        # 2.2 综合选股
    with ThreadPoolExecutor() as executor:
        executor.submit(hdtj.main)     # 3.1 非实时基础数据
        # executor.submit(gdj.main)    # 3.2 指标（当前注释）
        # executor.submit(kdj.main)    # 4. K 线形态（当前注释）
        # executor.submit(sdj.main)    # 5. 策略（当前注释）
    # bdj.main()                       # 6. 回测（当前注释）
    acdj.main()                        # 7. 闭盘后数据
```

> ⚠️ 这是理解“为什么默认只跑了一部分数据”的关键。当前仓库把指标、形态、策略、回测四步注释掉了。如果需要完整流水线：
> - 取消对应 `executor.submit(...)` / `bdj.main()` 的注释；或
> - 单独运行各 `*_daily_job.py`。

同时 `execute_daily_job.py` 在导入前把项目根目录加入 `sys.path`，并把日志写到 `instock/log/stock_execute_job.log`。

---

## 5. 各作业细节

### 5.1 `init_job.py`

- `check_database()` 执行 `SELECT 1`；
- 失败则 `create_new_database()`：连到 `mysql` 库执行 `CREATE DATABASE ... utf8mb4`，随后 `create_new_base_table()` 建 `cn_stock_attention`；
- **其他业务表不在此创建**，而是在第一次 `insert_db_from_df` 时由 `DataFrame.to_sql(if_exists='append')` 自动建表。

### 5.2 `basic_data_daily_job.py`

- `save_nph_stock_spot_data(date, before)`：`stock_data(date)` → `cn_stock_spot`；
- `save_nph_etf_spot_data(date, before)`：`fetch_etfs(date)` → `cn_etf_spot`。

### 5.3 `selection_data_daily_job.py`

- `fetch_stock_selection()` → `cn_stock_selection`；
- 删除用的是数据内自带的 `date`（`data.iloc[0]['date']`）。

### 5.4 `basic_data_other_daily_job.py`

| 函数 | 行为 |
| --- | --- |
| `save_nph_stock_lhb_data` | 龙虎榜（东财），随后调用 `stock_spot_buy(date)` |
| `save_nph_stock_top_data` | 龙虎榜（新浪），随后调用 `stock_spot_buy(date)` |
| `save_nph_stock_fund_flow_data` | 拉取 4 个周期（今日/3日/5日/10日）并按 `code` 合并 |
| `save_nph_stock_sector_fund_flow_data` | 行业与概念两个板块，各 3 个周期合并 |
| `save_nph_stock_bonus` | 分红配送 |
| `stock_spot_buy` | 基本面选股：`pe9>0 and pe9<=20 and pbnewmrq<=10 and roe_weight>=15` |
| `stock_chip_race_open_data` | 早盘抢筹 |
| `stock_imitup_reason_data` | 涨停原因 |

资金流合并逻辑：先以周期 0 为主表，后续周期 `drop columns=['name','new_price']` 后按 `code` 左连接（板块按 `name` 连接）。

### 5.5 `indicators_data_daily_job.py`

1. `prepare(date)`：取历史行情 → `run_check()` 并发计算 → 合并日期/代码 → 写 `cn_stock_indicators`；
2. `guess_buy(date)`：SQL 二次筛选“超买/强势”组合 → `cn_stock_indicators_buy`；
3. `guess_sell(date)`：SQL 二次筛选“超卖/弱势”组合 → `cn_stock_indicators_sell`。

买入条件（同时满足）：

```sql
kdjk >= 80 AND kdjd >= 70 AND kdjj >= 100 AND rsi_6 >= 80
AND cci >= 100 AND cr >= 300 AND wr_6 >= -20 AND vr >= 160
```

卖出条件（同时满足）：

```sql
kdjk < 20 AND kdjd < 30 AND kdjj < 10 AND rsi_6 < 20
AND cci < -100 AND cr < 40 AND wr_6 < -80 AND vr < 40
```

> 注意 `wr_6 >= -20` 与 `wr_6 < -80` 的方向差异来自 WILLR 的负值取值范围，阅读时按实际公式理解。

### 5.6 `klinepattern_data_daily_job.py`

- `run_check` 并发调用 `kpr.get_pattern_recognition`，`calc_threshold=12`（只看最近 12 根 K 线）；
- 只保留至少命中一种形态的股票；
- 写 `cn_stock_pattern`。

### 5.7 `strategy_data_daily_job.py`

- 遍历 `tbs.TABLE_CN_STOCK_STRATEGIES`，每个策略一个线程；
- `run_check` 并发判断每只股票，命中则收集 `(date, code, name)`；
- 结果表结构 = 外键列 + 回测列，便于后续回测回填；
- 对 `check_high_tight` 预取机构上榜集合，传入 `istop`。

### 5.8 `backtest_data_daily_job.py`

1. 汇总待回测表：`cn_stock_indicators_buy`、`cn_stock_indicators_sell` + 全部策略表；
2. 找出每张表中“最后一列（`rate_100`）为 NULL 且日期小于今天”的记录；
3. 用全量历史行情 `stock_hist_data().get_data()` 计算收益率序列；
4. `update_db_from_df(data_new, table_name, ('date','code'))` 回填。

### 5.9 `basic_data_after_close_daily_job.py`

- `save_after_close_stock_blocktrade_data(date)`：大宗交易；
- `save_after_close_stock_chip_race_end_data(date)`：尾盘抢筹；
- 二者命名以 `save_after_close` 开头，因此当前时间模式下使用 `run_date`（上一个已收盘交易日）。

---

## 6. 新增一个作业的步骤

1. 在 `core/crawling/` 写抓取函数，返回中文列名 `DataFrame`；
2. 在 `core/tablestructure.py` 定义 `TABLE_CN_XXX`（含 `name`/`cn`/`columns`）；
3. 在 `core/stockfetch.py` 增加 `fetch_xxx()`：重命名列、过滤代码；
4. 在 `job/` 新建 `xxx_daily_job.py`，复制“删除-插入”模板；
5. （可选）在 `singleton_stock_web_module_data.py` 注册页面；
6. 在 `execute_daily_job.py` / `cron` 中加入调度。

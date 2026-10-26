# 02 · core 模块

`instock/core/` 是整个系统的领域层：**抓取 → 指标 → 形态 → 策略 → 回测 → 可视化**。本节按子模块说明职责、关键类与函数。

---

## 1. 目录与职责

| 路径 | 职责 |
| --- | --- |
| `core/stockfetch.py` | 数据获取门面：统一字段名、过滤 A 股/非 ST、组织历史行情缓存 |
| `core/crawling/` | 各数据源的原始接口封装（东方财富、新浪） |
| `core/eastmoney_fetcher.py` | 东方财富 HTTP 会话封装（Cookie、代理、重试、连接池） |
| `core/indicator/calculate_indicator.py` | 技术指标计算（TA-Lib + 自定义公式） |
| `core/pattern/pattern_recognitions.py` | K 线形态识别 |
| `core/strategy/` | 10 个选股策略 |
| `core/backtest/rate_stats.py` | 未来 N 日收益率回测 |
| `core/kline/` | K 线图（Bokeh）、指标页签、筹码分布算法 |
| `core/tablestructure.py` | 表结构 / 字段字典 / 策略与形态注册表 |
| `core/singleton_*.py` | 各类单例缓存 |
| `core/web_module_data.py` | Web 页面元信息模型 |

---

## 2. 数据获取门面 `stockfetch.py`

[stockfetch.py](file:///workspace/instock/core/stockfetch.py) 是作业层唯一直接依赖的数据入口。它做了三件事：

1. 调用 `crawling` 的原始接口；
2. 把中文列名重命名为 `tablestructure` 定义的英文字段；
3. 过滤 A 股代码与无效价格（`is_not_st` 已提供，但当前抓取链路未启用 ST 过滤）。

### 2.1 过滤工具

| 函数 | 说明 |
| --- | --- |
| `is_a_stock(code)` | 保留 `600/601/603/605/000/001/002/003/300/301` 开头 |
| `is_not_st(name)` | 排除 `*ST` / `ST` 开头（已定义，当前未被调用） |
| `is_open(price)` | 排除 `NaN`（退市/停牌） |
| `is_open_with_line(price)` | 排除 `'-'` |

### 2.2 主要抓取函数

| 函数 | 数据源 | 产出表 |
| --- | --- | --- |
| `fetch_stocks_trade_date()` | 新浪交易日历 | `set[date]` |
| `fetch_stocks(date)` | 东方财富 A 股实时行情 | `cn_stock_spot` |
| `fetch_etfs(date)` | 东方财富 ETF 实时行情 | `cn_etf_spot` |
| `fetch_stock_selection()` | 东方财富选股器 | `cn_stock_selection` |
| `fetch_stocks_fund_flow(index)` | 个股资金流（今日/3日/5日/10日） | `cn_stock_fund_flow` |
| `fetch_stocks_sector_fund_flow(index_sector, index_indicator)` | 行业/概念资金流 | `cn_stock_fund_flow_industry/concept` |
| `fetch_stocks_bonus(date)` | 分红配送 | `cn_stock_bonus` |
| `fetch_stock_lhb_data(date)` | 东方财富龙虎榜 | `cn_stock_lhb` |
| `fetch_stock_top_data(date)` | 新浪龙虎榜 | `cn_stock_top` |
| `fetch_stock_top_entity_data(date)` | 近 90 天机构上榜统计 | 供 `high_tight_flag` 策略使用 |
| `fetch_stock_blocktrade_data(date)` | 大宗交易 | `cn_stock_blocktrade` |
| `fetch_stock_chip_race_open/end(date)` | 早盘/尾盘抢筹 | `cn_stock_chip_race_open/end` |
| `fetch_stock_limitup_reason(date)` | 涨停原因 | `cn_stock_limitup_reason` |
| `fetch_stock_hist(data_base, date_start, is_cache)` | 个股日线 | 返回 `DataFrame` |
| `fetch_etf_hist(...)` | ETF 日线 | 返回 `DataFrame` |

### 2.3 历史行情与本地缓存

```python
def fetch_stock_hist(data_base, date_start=None, is_cache=True):
    date, code = data_base[0], data_base[1]
    data = stock_hist_cache(code, date_start, None, is_cache, 'qfq')
    data['p_change'] = tl.ROC(data['close'].values, 1)   # 涨跌幅
    data['volume']  = data['volume'] * 100               # 手 -> 股
```

`stock_hist_cache()` 的缓存策略：

- 目录：`instock/cache/hist/<YYYYMM>/<start_date>/`
- 文件：`<code><adjust>.gzip.pickle`
- 命中则直接 `pd.read_pickle`，否则调用 `stock_zh_a_hist` 抓取并写缓存。
- 缓存目录由 [cron.monthly/run_monthly](file:///workspace/cron/cron.monthly/run_monthly) 定期清理。

### 2.4 历史区间

历史抓取窗口由 `lib.trade_time.get_trade_hist_interval(date)` 给出：**往前 3 年**；若目标日期是今天且处于盘中，则禁用缓存（`is_cache=False`）。

---

## 3. 抓取层 `crawling/`

每个文件封装一个数据源/接口，**输出的列名是中文**，由 `stockfetch` 重命名。统一使用全局 `eastmoney_fetcher` 实例发送请求。

| 文件 | 代表函数 | 说明 |
| --- | --- | --- |
| `stock_hist_em.py` | `stock_zh_a_spot_em()`, `stock_zh_a_hist()` | A 股实时行情、历史日线、分钟线 |
| `fund_etf_em.py` | `fund_etf_spot_em()`, `fund_etf_hist_em()` | ETF 实时与历史行情 |
| `stock_selection.py` | `stock_selection()` | 东方财富选股器（分页拉取） |
| `stock_fund_em.py` | `stock_individual_fund_flow_rank()`, `stock_sector_fund_flow_rank()` | 个股/板块资金流排名 |
| `stock_lhb_em.py` | `stock_lhb_detail_em()`, `stock_lhb_jgmmtj_em()` | 龙虎榜明细、机构买卖统计 |
| `stock_lhb_sina.py` | `stock_lhb_ggtj_sina()` | 新浪龙虎榜个股统计 |
| `stock_dzjy_em.py` | `stock_dzjy_mrtj()` | 大宗交易每日统计 |
| `stock_fhps_em.py` | `stock_fhps_em()` | 分红配送 |
| `stock_chip_race.py` | `stock_chip_race_open/end()` | 早盘/尾盘抢筹 |
| `stock_limitup_reason.py` | `stock_limitup_reason()` | 涨停原因 |
| `trade_date_hist.py` | `tool_trade_date_hist_sina()` | 交易日历 |
| `stock_cpbd.py` | `stock_cpbd_em()`, `stock_zjlx_em()` | 操盘必读（当前未被作业调用，预留扩展） |

### 3.1 东方财富会话封装 `eastmoney_fetcher`

[eastmoney_fetcher.py](file:///workspace/instock/core/eastmoney_fetcher.py) 类 `eastmoney_fetcher`：

| 成员 | 说明 |
| --- | --- |
| `_get_cookie()` | Cookie 优先级：环境变量 `EAST_MONEY_COOKIE` > `config/eastmoney_cookie.txt` > 内置默认值 |
| `_create_session()` | 连接池 50、重试 3 次（429/5xx）、统一 UA/Referer |
| `make_request(url, params, retry=3, timeout=10)` | GET，失败随机延迟重试 |
| `make_post_request(...)` | POST |
| `update_cookie(new_cookie)` | 运行期更新 Cookie |

代理来自 `core/singleton_proxy.py` 的 `proxys` 单例：读取 `config/proxy.txt`，每行一个，`get_proxies()` 随机返回一个 `{"http":..., "https":...}`。

---

## 4. 指标计算 `indicator/calculate_indicator.py`

核心函数：

```python
get_indicators(data, end_date=None, threshold=120, calc_threshold=None) -> DataFrame
get_indicator(code_name, data, stock_column, date=None, calc_threshold=90) -> Series
```

- `get_indicators`：在整段历史上批量计算所有指标列，返回最近 `threshold` 行。
- `get_indicator`：取指定日期当天的**最后一行**，组装为 `[date, code, <各指标>]` 的 `Series`，供作业并发调用。
- 所有 `NaN/Inf` 统一置 `0.0`，避免入库异常。
- 部分指标刻意使用 `stockstats` 的计算口径（如 CR、VR、DMI、CCI、MFI），以对齐同花顺/通达信。

覆盖指标大类：

| 类别 | 字段 |
| --- | --- |
| 趋势 | `macd/macds/macdh`、`trix`、`tema`、`dma`、`ma6/10/12/20/24/50/200` |
| 摆动 | `kdjk/kdjd/kdjj`、`rsi/rsi_6/rsi_12/rsi_24`、`wr_6/10/14`、`cci/cci_84` |
| 通道 | `boll/boll_ub/boll_lb`、`ene/ene_ue/ene_le` |
| 量价 | `vr`、`obv`、`mfi`、`vwma`、`vol_5/vol_10` |
| 波动 | `tr`、`atr`、`supertrend`、`vhf` |
| 其他 | `sar`、`psy`、`br/ar`、`emv`、`bias`、`dpo`、`rvi`、`fi`、`wt1/wt2`、`ppo`、`stochrsi_k/d` |

> 指标字段全集定义在 [STOCK_STATS_DATA](file:///workspace/instock/core/tablestructure.py#L302-L376)，`cn_stock_indicators` 表 = 外键字段 + 指标字段。

---

## 5. K 线形态 `pattern/pattern_recognitions.py`

```python
get_pattern_recognitions(data, stock_column, end_date=None, threshold=120, calc_threshold=None)
get_pattern_recognition(code_name, data, stock_column, date=None, calc_threshold=12) -> Series | None
```

- `stock_column` 来自 `tablestructure.STOCK_KLINE_PATTERN_DATA['columns']`，每个字段含 `func`（TA-Lib 的 `CDLxxx`）。
- 对每列执行 `func(open, high, low, close)`，结果为 `-1/0/1`。
- `get_pattern_recognition` 只在该股票**任一形态非 0** 时返回一行，减少写库量。

形态共 61 种，注册表见 [STOCK_KLINE_PATTERN_DATA](file:///workspace/instock/core/tablestructure.py#L417-L537)。

---

## 6. 选股策略 `strategy/`

每个策略是一个纯函数，签名统一：

```python
def check(code_name, data, date=None, threshold=60) -> bool
```

- `code_name = (date, code, name)`
- `data` 为该股票历史日线 `DataFrame`
- 内部先按 `date` 截断，再取最近 `threshold` 行判断。

| 文件 | 函数 | 策略 | 表名 |
| --- | --- | --- | --- |
| `enter.py` | `check_volume` | 放量上涨 | `cn_stock_strategy_enter` |
| `keep_increasing.py` | `check` | 均线多头 | `cn_stock_strategy_keep_increasing` |
| `parking_apron.py` | `check` | 停机坪 | `cn_stock_strategy_parking_apron` |
| `backtrace_ma250.py` | `check` | 回踩年线 | `cn_stock_strategy_backtrace_ma250` |
| `breakthrough_platform.py` | `check` | 突破平台 | `cn_stock_strategy_breakthrough_platform` |
| `low_backtrace_increase.py` | `check` | 无大幅回撤 | `cn_stock_strategy_low_backtrace_increase` |
| `turtle_trade.py` | `check_enter` | 海龟交易法则 | `cn_stock_strategy_turtle_trade` |
| `high_tight_flag.py` | `check_high_tight` | 高而窄的旗形 | `cn_stock_strategy_high_tight_flag` |
| `climax_limitdown.py` | `check` | 放量跌停 | `cn_stock_strategy_climax_limitdown` |
| `low_atr.py` | `check_low_increase` | 低 ATR 成长 | `cn_stock_strategy_low_atr` |

策略在 [TABLE_CN_STOCK_STRATEGIES](file:///workspace/instock/core/tablestructure.py#L391-L415) 中集中注册，作业层遍历该列表自动跑全部策略 —— **新增策略只需新建文件并注册一行**。

`high_tight_flag.check_high_tight` 额外接收 `istop` 参数（是否近 90 天有 2 次以上机构上榜），由作业层预取后传入。

---

## 7. 回测 `backtest/rate_stats.py`

```python
get_rates(code_name, data, stock_column, threshold=101) -> Series
```

- 从选股日 `start_date` 开始，取后续最多 101 个交易日；
- 计算 `sum_pct_change = 100 * (close - close1) / close1`；
- 依次写入 `rate_1 … rate_100`，不足补 `None`；
- 结果通过 `update_db_from_df` 回填到原选股表。

回测字段定义见 [TABLE_CN_STOCK_BACKTEST_DATA](file:///workspace/instock/core/tablestructure.py#L298-L300)。

---

## 8. K 线可视化 `kline/`

### 8.1 `visualization.py`

`get_plot_kline(code, stock, date, stock_name)` 生成一整页 Bokeh 图表：

1. 计算 `threshold=360` 的指标 + `120` 根 K 线形态；
2. 绘制 K 线（红涨绿跌）、MA5/10/20/50/200 均线、成交量柱；
3. 右侧绘制**筹码分布**（调用 `cyq.js` 的 CustomJS 回调）；
4. 底部按 [indicator_web_dic.py](file:///workspace/instock/core/kline/indicator_web_dic.py) 生成多个指标页签；
5. 叠加形态标签（买点红色、卖点绿色）与形态复选框；
6. 附带“关注/取关”“东方财富行情”“资料”“扫雷评级”等按钮；
7. 返回 `{"script": ..., "div": ...}` 供模板嵌入。

> 注意到依赖 `curdoc()`（Bokeh 文档级 API），该函数适合在 Tornado Handler 中渲染。

### 8.2 `cyq.py` · 筹码分布算法

类 `CYQCalculator(kdata, accuracy_factor=150, crange=120, cyq_days=210)`：

- `calc(index)`：以选中 K 线为终点，回溯 `cyq_days=210` 个交易日；
- 将价格区间按 `accuracy_factor` 离散化，按换手率衰减历史筹码，并按“三角分布”叠加当日成交量；
- 输出：筹码分布 `x/y`、获利比例 `benefit_part`、平均成本 `avg_cost`、90%/70% 成本区间与集中度。

`cyq.js` 是与之配套的浏览器端实现，用于图表交互（悬停切换日期时即时重算）。

### 8.3 `indicator_web_dic.py`

`indicators_dic` 列表定义指标页签：`title`、说明 HTML（`desc`）、需要绘制的字段元组（`dic`）。新增指标页只需在此追加配置。

---

## 9. 单例与页面模型

| 文件 | 类 | 说明 |
| --- | --- | --- |
| `singleton_stock.py` | `stock_data` | 缓存当日 `fetch_stocks(date)` 结果 |
| | `stock_hist_data` | 多线程抓取全部股票历史行情 |
| `singleton_trade_date.py` | `stock_trade_date` | 缓存交易日集合 |
| `singleton_proxy.py` | `proxys` | 缓存代理列表，随机取用 |
| `singleton_stock_web_module_data.py` | `stock_web_module_data` | 构建所有 Web 页面元信息 |
| `web_module_data.py` | `web_module_data` | 单个页面元信息（POJO） |

`stock_hist_data` 的实现要点：

```python
class stock_hist_data(metaclass=singleton_type):
    def __init__(self, date=None, stocks=None, workers=16):
        _subset = stock_data(date).get_data()[['date','code','name']]
        stocks = [tuple(x) for x in _subset.values]
        with ThreadPoolExecutor(max_workers=workers) as executor:
            ...  # 并发 fetch_stock_hist
```

`workers=16`，并在作业侧对指标/形态/策略再用 `workers=40` 的线程池并发。

---

## 10. 表结构 `tablestructure.py`

该文件是全系统的“单一事实源”，包含：

- 所有 `TABLE_CN_*` 表定义；
- 列模板：`STOCK_STATS_DATA`（指标列）、`STOCK_KLINE_PATTERN_DATA`（形态列）、`TABLE_CN_STOCK_BACKTEST_DATA`（回测列）、`CN_STOCK_HIST_DATA`（历史行情列）；
- 注册表：`TABLE_CN_STOCK_STRATEGIES`、`CN_STOCK_FUND_FLOW`、`CN_STOCK_SECTOR_FUND_FLOW`；
- 辅助函数：`get_field_cn` / `get_field_cns` / `get_field_types` / `get_field_type_name`。

详细表清单见 [07-data-model.md](file:///workspace/docs/code-wiki/07-data-model.md)。

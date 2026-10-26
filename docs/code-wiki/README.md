# InStock 股票系统 · Code Wiki

> 本文档面向希望理解、二次开发或部署 **InStock 股票系统** 的开发者。
> 内容基于当前仓库源码（版本号见 [version.py](file:///workspace/instock/lib/version.py)）整理，覆盖整体架构、模块职责、关键类/函数、依赖关系与运行方式。

---

## 1. 项目是什么

InStock 是一套 **A 股量化分析 + 选股 + 回测 + 可视化 + 自动交易** 的个人投资辅助系统。核心能力：

| 能力 | 说明 |
| --- | --- |
| 数据抓取 | 从东方财富、新浪财经抓取每日股票/ETF 行情、资金流、龙虎榜、大宗交易、分红配送、抢筹、涨停原因等 |
| 技术指标 | 基于 TA-Lib + Pandas 计算 30+ 类指标（MACD、KDJ、BOLL、RSI、DMI、ATR、Supertrend…） |
| K 线形态 | 基于 TA-Lib 的 61 种 `CDLxxx` 形态识别，输出 -1/0/1 |
| 筹码分布 | 复刻“成本分布（CYQ）”，结果对齐东方财富 |
| 策略选股 | 内置 10 个选股策略，并通过“策略表”配置化扩展 |
| 回测验证 | 对指标/策略选出的股票计算未来 N 日收益率 |
| Web 展示 | Tornado + GC.Spread.Sheets + Bokeh（K 线图） |
| 自动交易 | 基于 `easytrader` 的事件驱动交易机器人（默认仅打新） |
| 批量作业 | 任意日期 / 枚举日期 / 区间日期批量跑数 |

> 免责声明：股市有风险，本项目仅用于学习与研究。

---

## 2. 技术栈

- 语言：Python 3.11
- 数据处理：`pandas`、`numpy`
- 技术指标/形态：`TA-Lib`
- 存储：MySQL / MariaDB（`SQLAlchemy` + `PyMySQL`）
- Web：`tornado`、`bokeh`（K 线 + 筹码图）、`GC.Spread.Sheets`（前端表格控件）
- 抓取：`requests`（统一封装 `eastmoney_fetcher`）
- 自动交易：`easytrader`、`logbook`
- 其他：`arrow`、`py_mini_racer`、`beautifulsoup4`、`tqdm`

完整依赖见 [requirements.txt](file:///workspace/requirements.txt)。

---

## 3. 文档目录

| 文档 | 内容 |
| --- | --- |
| [01-architecture.md](file:///workspace/docs/code-wiki/01-architecture.md) | 整体架构、分层、数据流、目录结构、模块依赖图、核心设计模式 |
| [02-core-module.md](file:///workspace/docs/code-wiki/02-core-module.md) | `core/`：抓取层、单例缓存、指标、形态、策略、筹码与 K 线可视化 |
| [03-job-module.md](file:///workspace/docs/code-wiki/03-job-module.md) | `job/`：每日作业编排、批量日期、回测流程 |
| [04-web-module.md](file:///workspace/docs/code-wiki/04-web-module.md) | `web/`：Tornado 路由、Handler、视图字典、模板与前端 |
| [05-trade-module.md](file:///workspace/docs/code-wiki/05-trade-module.md) | `trade/`：事件引擎、时钟引擎、策略模板、自动交易 |
| [06-lib-module.md](file:///workspace/docs/code-wiki/06-lib-module.md) | `lib/`：数据库封装、交易日历、批量模板、单例元类、配置项 |
| [07-data-model.md](file:///workspace/docs/code-wiki/07-data-model.md) | 数据库表结构、字段字典、表自动创建机制 |
| [08-deployment-and-run.md](file:///workspace/docs/code-wiki/08-deployment-and-run.md) | 安装、运行、Docker、Cron、Supervisor、日志与排错 |

---

## 4. 一分钟了解运行方式

```bash
# 1) 安装依赖（需先安装 MySQL 与 TA-Lib C 库）
python -m pip install -r requirements.txt

# 2) 修改数据库连接（instock/lib/database.py 或环境变量）
#    db_host / db_user / db_password / db_database / db_port

# 3) 跑一次完整作业（首次会自动建库建表）
cd instock/job
python execute_daily_job.py

# 4) 启动 Web（默认 9988 端口）
cd instock/web
python web_service.py
# 浏览器访问 http://localhost:9988/

# 5) 启动自动交易（可选，Windows）
cd instock/trade
python trade_service.py
```

批量时间运行：

```bash
python execute_daily_job.py 2023-03-01                      # 单个日期
python execute_daily_job.py 2023-03-01,2023-03-02           # 枚举日期
python execute_daily_job.py 2023-03-01 2023-03-21           # 区间日期
```

详细的部署与定时任务见 [08-deployment-and-run.md](file:///workspace/docs/code-wiki/08-deployment-and-run.md)。

---

## 5. 关键设计一句话总结

1. **分层解耦**：`job`（编排）→ `core/stockfetch`（统一数据入口）→ `core/crawling`（具体网站抓取）→ `lib/database`（落库）；`web` 只读库。
2. **配置即表结构**：所有表结构集中在 `core/tablestructure.py`，作业、Web 视图、建表都由它驱动。
3. **单例共享**：用 `singleton_type` 元类缓存“当日行情 / 历史行情 / 交易日历 / Web 视图字典 / 代理池”，避免重复抓取与重复构造。
4. **多线程**：作业内部用 `ThreadPoolExecutor` 按股票并发；批量日期也按日期并发。
5. **流程化回测**：选股结果表预留 `rate_1…rate_100` 字段，回测作业回填未来收益率。

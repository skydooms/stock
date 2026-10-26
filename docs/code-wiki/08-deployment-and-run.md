# 08 · 部署与运行

本文说明从零安装、配置、运行、容器化部署以及常见问题排查。

---

## 1. 环境要求

| 组件 | 版本/说明 |
| --- | --- |
| Python | 3.11（项目开发版本） |
| MySQL / MariaDB | 建议最新版，字符集 `utf8mb4` |
| TA-Lib C 库 | **必须先安装 C 共享库**，再 `pip install TA-Lib` |
| 操作系统 | Windows / Linux / macOS；自动交易仅 Windows |
| 可选 | Tesseract（交易验证码识别）、Navicat（数据库管理） |

TA-Lib 安装参考官方：<https://ta-lib.org/install/>（Windows 安装包、macOS Homebrew、Linux deb 包）。

---

## 2. 常规安装

```bash
# 1) 安装依赖
python -m pip install -r requirements.txt
```

依赖清单见 [requirements.txt](file:///workspace/requirements.txt)：`numpy`、`pandas`、`TA-Lib`、`SQLAlchemy`、`PyMySQL`、`tornado`、`bokeh`、`requests`、`easytrader`、`logbook`、`arrow`、`beautifulsoup4`、`py_mini_racer`、`mini-racer`、`tqdm`、`pycryptodome` 等。

### 2.1 配置数据库

编辑 [database.py](file:///workspace/instock/lib/database.py#L14-L19)：

```python
db_host = "localhost"
db_user = "root"
db_password = "root"
db_database = "instockdb"
db_port = 3306
db_charset = "utf8mb4"
```

也支持环境变量覆盖（Docker `-e`）：`db_host`、`db_user`、`db_password`、`db_database`、`db_port`。

### 2.2 配置代理（可选）

编辑 `instock/config/proxy.txt`，每行一个：

```
127.0.0.1:7860
52.13.248.29:3128
abc:123456@65.1.244.232:3128
```

- 为空 = 不使用代理；
- 修改后需重启程序才生效；
- 抓取时随机选择一个代理（`core/singleton_proxy.py`）。

### 2.3 配置东方财富 Cookie（推荐）

优先级：环境变量 > 文件 > 内置默认值。

```bash
# 环境变量（推荐）
export EAST_MONEY_COOKIE="你的Cookie"
# 或编辑文件
instock/config/eastmoney_cookie.txt
```

Cookie 通常几天到几周过期，失效后需重新获取。

### 2.4 自动交易（可选，仅 Windows）

1. 安装券商同花顺客户端；
2. 安装 Tesseract 并加入 `PATH`；
3. 编辑 `instock/config/trade_client.json`：

```json
{"user": "账号", "password": "密码", "exe_path": "C:/gfzqrzrq/xiadan.exe"}
```

4. 修改 [trade_service.py](file:///workspace/instock/trade/trade_service.py#L20) 中的 `broker`（如 `gf_client`、`ht_client`、`universal_client`），详见 [usage.md](file:///workspace/instock/trade/usage.md)。

---

## 3. 运行方式（本机）

### 3.1 数据作业

```bash
cd instock/job
python execute_daily_job.py                     # 当前时间
python execute_daily_job.py 2023-03-01          # 单个日期
python execute_daily_job.py 2023-03-01,2023-03-02   # 枚举日期
python execute_daily_job.py 2023-03-01 2023-03-21   # 区间日期
```

Windows 可直接运行 `instock/bin/run_job.bat`，Linux/Mac 用 `instock/bin/run_job.sh`。

单功能作业（均支持批量日期）：

```bash
python init_job.py
python basic_data_daily_job.py
python basic_data_other_daily_job.py
python indicators_data_daily_job.py
python klinepattern_data_daily_job.py
python strategy_data_daily_job.py
python backtest_data_daily_job.py
python basic_data_after_close_daily_job.py
python selection_data_daily_job.py
```

> ⚠️ 当前 `execute_daily_job.py` 中指标/形态/策略/回测步骤被注释，若需要完整流水线请取消注释或单独运行上述脚本（详见 [03-job-module.md](file:///workspace/docs/code-wiki/03-job-module.md)）。

### 3.2 Web 服务

```bash
cd instock/web
python web_service.py
# 访问 http://localhost:9988/
```

### 3.3 交易服务

```bash
cd instock/trade          # 必须在 trade 目录下运行
python trade_service.py
```

图表化的运行入口：

| 脚本 | 作用 |
| --- | --- |
| `instock/bin/run_job.bat` / `run_job.sh` | 运行整体作业 |
| `instock/bin/run_web.bat` / `run_web.sh` | 启动 Web |
| `instock/bin/run_trade.bat` | 启动交易 |
| `instock/bin/restart_web.sh` | 杀掉旧 Web 进程（容器内用） |

---

## 4. Docker 部署

镜像：`mayanghua/instock`（<https://hub.docker.com/r/mayanghua/instock>）。

### 4.1 构建

[build.sh](file:///workspace/docker/build.sh) 会同步源码到 `docker/stock` 并构建 `latest` 与 `YYYYMM` 双标签，然后推送。

```bash
cd docker
sh build.sh
```

[Dockerfile](file:///workspace/docker/Dockerfile) 要点：

- 基础镜像 `python:3.11-slim-bullseye`；
- 设置时区 `Asia/Shanghai`、`PYTHONPATH=/data/InStock`；
- 编译安装 TA-Lib；
- 复制 `stock` → `/data/InStock`；
- 写入 crontab（见第 5 节）；
- `ENTRYPOINT ["supervisord", ...]`。

### 4.2 运行（推荐：数据库独立容器）

```bash
docker network create InStockService

docker run -d --name InStockDbService \
  --network InStockService \
  -v /data/mariadb/data:/var/lib/instockdb \
  -e MYSQL_ROOT_PASSWORD=root \
  library/mariadb:latest

docker run -dit --name InStock --network=InStockService \
  -p 9988:9988 \
  -v /data/instockproxy.txt:/data/InStock/instock/config/proxy.txt \
  -v /data/eastmoneycookie.txt:/data/InStock/instock/config/eastmoney_cookie.txt \
  -e db_host=InStockDbService \
  mayanghua/instock:latest
```

也提供 [docker-compose.yml](file:///workspace/docker/docker-compose.yml)：

```bash
cd docker
docker compose up -d
```

### 4.3 Docker 环境变量

| 变量 | 说明 |
| --- | --- |
| `db_host` | 数据库主机 |
| `db_user` | 用户名 |
| `db_password` | 密码 |
| `db_database` | 数据库名 |
| `db_port` | 端口 |

---

## 5. 定时任务（容器内 Cron）

Dockerfile 写入 `/var/spool/cron/crontabs/root`：

```cron
*/30 9,10,11,13,14,15 * * 1-5  /bin/run-parts /etc/cron.hourly      # 盘中每 30 分钟
30 17 * * 1-5                  /bin/run-parts /etc/cron.workdayly   # 工作日 17:30 整体作业
30 10 * * 3,6                  /bin/run-parts /etc/cron.monthly    # 周三/周六 10:30 清理缓存
```

对应脚本：

| 脚本 | 实际命令 |
| --- | --- |
| [cron.hourly/run_hourly](file:///workspace/cron/cron.hourly/run_hourly) | `python3 .../basic_data_daily_job.py`（实时行情） |
| [cron.workdayly/run_workdayly](file:///workspace/cron/cron.workdayly/run_workdayly) | `python3 .../execute_daily_job.py`（整体作业） |
| [cron.monthly/run_monthly](file:///workspace/cron/cron.monthly/run_monthly) | `rm -rf .../cache/hist/*`（清理历史缓存） |

`run_cron.sh` 会导出编码/PYTHONPATH 环境变量，并以前台方式运行 `cron -f`，以便被 supervisor 守护。

---

## 6. 进程守护（Supervisor）

[supervisord.conf](file:///workspace/supervisor/supervisord.conf) 定义三个程序：

| program | 命令 | 自动重启 | priority |
| --- | --- | --- | --- |
| `run_job` | `run_job.sh`（整体作业） | `false` | 100 |
| `run_web` | `run_web.sh`（Web 服务） | `true` | 500 |
| `run_cron` | `run_cron.sh`（cron） | `true` | 900 |

容器启动时：先跑一次整体作业 → 启动 Web → 启动 cron。管理端口 `9001`（`inet_http_server`，默认无认证，注意安全）。

常用命令：

```bash
supervisorctl -c /data/InStock/supervisor/supervisord.conf status
supervisorctl -c /data/InStock/supervisor/supervisord.conf restart run_web
```

---

## 7. 日志

| 日志 | 位置 | 内容 |
| --- | --- | --- |
| 数据作业 | `instock/log/stock_execute_job.log` | 抓取、计算、入库、异常 |
| Web 服务 | `instock/log/stock_web.log` | Web 异常 |
| 交易服务 | `instock/log/stock_trade.log` | 交易主引擎 |
| 策略日志 | `instock/log/<策略名>.log` | 各策略自定义 |
| Supervisor | `/tmp/supervisord.log` | 进程守护日志 |

Docker 内查看：

```bash
docker exec -it InStock bash
cat /data/InStock/instock/log/stock_execute_job.log
cat /data/InStock/instock/log/stock_web.log
```

---

## 8. 常见问题排查

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| `ImportError: No module named talib` | 未装 TA-Lib C 库 | 先装 C 库再 `pip install TA-Lib` |
| 数据库连接失败 | 账号/地址/库名错误 | 检查 `database.py` 或环境变量；确认 MySQL 已启动 |
| 抓取返回空/报错 | 被限流 | 配置 Cookie、代理；降低并发或增加延迟 |
| Cookie 失效 | 过期 | 重新获取并更新 `eastmoney_cookie.txt` 或环境变量 |
| Web 页面无数据 | 当天作业未跑 / 日期不对 | 先跑 `basic_data_daily_job.py`，确认 `date` 参数 |
| 批量历史数据日期错乱 | 单例只按首次参数初始化 | 作业内已用 `data['date'] = date_str` 修正；自定义作业需注意 |
| 整体作业没有指标/策略数据 | `execute_daily_job.py` 中相关步骤被注释 | 取消注释或单独运行对应脚本 |
| 大宗交易为空 | 收盘后 1~2 小时才有数据 | 17:00 之后再抓 |
| 交易服务启动失败 | 非 Windows / 未装券商客户端 / 验证码识别失败 | 检查 `trade_client.json`、`exe_path`、Tesseract |
| 缓存导致数据不更新 | 历史行情本地缓存 | 删除 `instock/cache/hist/<日期目录>` 或等待月度清理 |

---

## 9. 二次开发速查

| 需求 | 改动点 |
| --- | --- |
| 新增数据源 | `core/crawling/` 新增抓取函数 + `stockfetch.py` 增加 `fetch_xxx` |
| 新增数据表 | `core/tablestructure.py` 定义 `TABLE_CN_XXX` |
| 新增每日作业 | `job/` 复制“删除-插入”模板，并在 `execute_daily_job.py` 编排 |
| 新增选股策略 | `core/strategy/` 新增 `check()`，在 `TABLE_CN_STOCK_STRATEGIES` 注册 |
| 新增 K 线形态 | 在 `STOCK_KLINE_PATTERN_DATA` 增加字段与 `func`（TA-Lib `CDLxxx`） |
| 新增 Web 页面 | `singleton_stock_web_module_data.py` 增加 `web_module_data` |
| 新增指标页签 | `core/kline/indicator_web_dic.py` 增加配置 |
| 新增交易策略 | `trade/strategies/` 新建继承 `StrategyTemplate` 的 `Strategy` 类 |

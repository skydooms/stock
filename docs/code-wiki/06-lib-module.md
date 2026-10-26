# 06 · lib 模块与配置

`instock/lib/` 提供与业务无关的基础能力：数据库访问、交易日历、批量日期模板、单例元类、AES 工具、版本号。同时本节说明 `instock/config/` 下的配置文件。

---

## 1. 目录

| 文件 | 职责 |
| --- | --- |
| `database.py` | MySQL 连接与通用读写函数 |
| `torndb.py` | Tornado 风格的轻量 MySQL 封装（Web 使用） |
| `trade_time.py` | 交易日历、交易时段判断、报告期计算 |
| `run_template.py` | 批量日期入口 `run_with_args` |
| `singleton_type.py` | 线程安全单例元类 |
| `crypto_aes.py` | AES 加解密工具类 |
| `version.py` | 版本号 `__version__` |

---

## 2. `database.py`

[database.py](file:///workspace/instock/lib/database.py) 同时支持**配置文件默认值**与**环境变量覆盖**（Docker `-e`）。

### 2.1 连接配置

```python
db_host = "localhost"
db_user = "root"
db_password = "root"
db_database = "instockdb"
db_port = 3306
db_charset = "utf8mb4"

# 环境变量优先
_db_host = os.environ.get('db_host')
...
```

派生出三种连接形式：

| 变量 | 用途 |
| --- | --- |
| `MYSQL_CONN_URL` | SQLAlchemy URL（`mysql+pymysql://...`） |
| `MYSQL_CONN_DBAPI` | `pymysql.connect(**...)` 字典 |
| `MYSQL_CONN_TORNDB` | Web 端 `torndb.Connection(**...)` |

### 2.2 核心函数

| 函数 | 签名 | 说明 |
| --- | --- | --- |
| `engine()` | `-> Engine` | 默认库的 SQLAlchemy 引擎 |
| `engine_to_db(to_db)` | `-> Engine` | 指定库的引擎 |
| `get_connection()` | `-> pymysql.Connection` | 原生连接（`autocommit=True`） |
| `insert_db_from_df(...)` | `(data, table_name, cols_type, write_index, primary_keys, indexs=None)` | 插入默认库 |
| `insert_other_db_from_df(...)` | 同上，首参 `to_db` | 插入指定库 |
| `update_db_from_df(data, table_name, where)` | — | 按 `where` 字段逐行更新 |
| `checkTableIsExist(table_name)` | `-> bool` | 查询 `information_schema` |
| `executeSql(sql, params)` | — | 增删改 |
| `executeSqlFetch(sql, params)` | `-> rows` | 查询多行 |
| `executeSqlCount(sql, params)` | `-> int` | 查询计数 |

### 2.3 插入逻辑（重点）

`insert_other_db_from_df` 的行为：

1. `data.to_sql(name=..., if_exists='append', dtype=cols_type, index=write_index)`；
2. 若 `cols_type` 为 `None` → 让 pandas 推断；
3. 若 `cols_type` 为空 dict → 所有列用 `NVARCHAR(255)`；
4. 插入后检查主键，没有则执行：

```sql
ALTER TABLE `table` ADD PRIMARY KEY (<primary_keys>);
ALTER TABLE `table` ADD INDEX IN<k>(<v>);
```

这解释了为什么 `init_job` 只需创建一张表：**其余表在首次写入时自动创建**，并随后补上主键（如 `` `date`,`code` ``）。

### 2.4 更新逻辑

`update_db_from_df` 手工拼 SQL（非参数化），逐行 `UPDATE`：

- `where` 中的列用于定位（如 `date`、`code`）；
- 其他列作为 `SET` 目标；
- `None`/`NaN` 写为 `NULL`。

> 由于是字符串拼接，调用方需保证数据来自可信的数据库/接口；回测场景数据均已内部生成。

---

## 3. `torndb.py`

[torndb.py](file:///workspace/instock/lib/torndb.py) 是 Tornado 旧版 `tornado.database` 的独立移植，基于 `pymysql`。

`class Connection` 主要方法：

| 方法 | 说明 |
| --- | --- |
| `query(sql, *params)` | 返回 `list[Row]` |
| `get(sql, *params)` | 返回单行，找不到抛异常 |
| `execute(sql, *params)` | 执行并返回影响行数 |
| `execute_lastrowid` / `execute_rowcount` | 返回 lastrowid / rowcount |
| `executemany` / `executemany_lastrowid` / `executemany_rowcount` | 批量执行 |
| `reconnect()` / `close()` | 重连/关闭 |

`class Row(dict)` 支持 `row.column_name` 属性访问，便于模板渲染。

Web 端在 `Application.__init__` 中创建唯一的 `torndb.Connection`，`BaseHandler.db` 每次访问前做 `SELECT 1` 健康检查。

---

## 4. `trade_time.py`

[trade_time.py](file:///workspace/instock/lib/trade_time.py) 是交易日/交易时段的唯一来源。

### 4.1 交易日

| 函数 | 说明 |
| --- | --- |
| `is_trade_date(date)` | 是否交易日（基于单例 `stock_trade_date`） |
| `get_previous_trade_date(date, count=1)` | 向前第 N 个交易日 |
| `get_one_previous_trade_date(date)` | 前一个交易日 |
| `get_next_trade_date(date)` | 后一个交易日 |
| `get_trade_date_last()` | 返回 `(run_date, run_date_nph)` 二元组 |

`get_trade_date_last` 语义：

- 今天是交易日且未收盘 → `run_date=上一交易日`，`run_date_nph=今天`；
- 今天是交易日但未开盘 → `run_date=上一交易日`，`run_date_nph=上一交易日`；
- 今天不是交易日 → 两者都为上一交易日。

`run_date` 用于“已收盘数据”，`run_date_nph` 用于“实时数据”。

### 4.2 交易时段

```python
OPEN_TIME     = ((09:15, 11:30), (13:00, 15:00))
PAUSE_TIME    = (11:30, 12:59:30)
CONTINUE_TIME = (12:59:30, 13:00)
CLOSE_TIME    = 15:00
```

`is_tradetime`、`is_pause`、`is_continue`、`is_closing`（默认 14:54:30）、`is_close`、`is_open`（09:30）。

### 4.3 其他

| 函数 | 说明 |
| --- | --- |
| `get_trade_hist_interval(date)` | 历史抓取窗口：`date - 3 年` ~ `date`；返回 `(start, is_cache)` |
| `get_quarterly_report_date()` | 当前应参考的季报日期（`YYYYMMDD`） |
| `get_bonus_report_date()` | 分红配送报告期 |

---

## 5. `run_template.py`

[run_template.py](file:///workspace/instock/lib/run_template.py) 提供统一批量入口 `run_with_args(run_fun, *args)`，行为已在 [03-job-module.md](file:///workspace/docs/code-wiki/03-job-module.md) 详述。核心是按函数名前缀决定日期：

- `save_nph_*` → 实时数据，当前模式下传 `before=False`；
- `save_after_close_*` → 闭盘数据，使用 `run_date`；
- 其他 → 使用 `run_date_nph`。

---

## 6. `singleton_type.py`

```python
class singleton_type(type):
    single_lock = RLock()
    def __call__(cls, *args, **kwargs):
        with singleton_type.single_lock:
            if not hasattr(cls, "_instance"):
                cls._instance = super().__call__(*args, **kwargs)
        return cls._instance
```

使用方式：`class Foo(metaclass=singleton_type)`。注意：**首次创建后参数被忽略**，这是批量作业中需要手动修正日期的根因。

---

## 7. `crypto_aes.py`

[crypto_aes.py](file:///workspace/instock/lib/crypto_aes.py) 基于 `pycryptodome` 的 AES 工具：

- `MData`：bytes/str/base64/hex 互转；
- `AESCryptor(key, mode, iv, paddingMode, characterSet)`：支持 `MODE_CBC` / `MODE_ECB`，填充方式 `NoPadding`、`ZeroPadding`、`PKCS5Padding`、`PKCS7Padding`；
- 提供 `encryptFrom*` / `decryptFrom*` 系列方法。

> 当前业务代码中未被直接引用，属于通用工具/预留能力。

---

## 8. 配置文件 `instock/config/`

| 文件 | 内容 | 说明 |
| --- | --- | --- |
| `proxy.txt` | 代理列表，每行 `ip:port` 或 `user:pass@ip:port` | 空文件 = 不使用代理；修改后需重启 |
| `eastmoney_cookie.txt` | 东方财富 Cookie | 降低被限流概率；环境变量 `EAST_MONEY_COOKIE` 优先 |
| `trade_client.json` | 交易账号/密码/客户端路径 | `{"user","password","exe_path"}` |

`config/.gitignore` 与各目录 `.gitignore` 用于避免把本地敏感配置提交到仓库。

### 8.1 配置优先级

```
环境变量 > config 文件 > 代码内置默认值
```

- 数据库：`db_host/db_user/db_password/db_database/db_port`；
- Cookie：`EAST_MONEY_COOKIE`；
- 代理：仅文件（`proxy.txt`）。

---

## 9. `version.py`

```python
__version__ = "4.0.0"
```

用于 Web 首页展示（`HomeHandler` 传入模板）。发版时更新。

# 05 · trade 模块（自动交易）

`instock/trade/` 是一个轻量事件驱动交易机器人，基于 `easytrader` 驱动券商客户端（如同花顺 `xiadan.exe`）。默认只内置“打新股”策略与一个示例策略。

> ⚠️ 涉及真实资金，请务必在模拟环境验证。README 明确提示：交易日 10:00 会触发打新，不需要请删除 `stagging.py` 或不启动交易服务。

---

## 1. 结构与职责

```
trade/
├── trade_service.py                 # 入口
├── strategies/                      # 策略目录（运行时扫描）
│   ├── stagging.py                  # 打新股
│   └── stratey1.py                  # 示例策略
├── robot/
│   ├── engine/
│   │   ├── main_engine.py           # 主引擎：加载策略、连接账户、生命周期
│   │   ├── event_engine.py          # 事件引擎：队列 + 分发
│   │   └── clock_engine.py          # 时钟引擎：时间/间隔事件
│   └── infrastructure/
│       ├── strategy_template.py     # 策略基类
│       ├── strategy_wrapper.py      # 多进程策略包装（预留）
│       └── default_handler.py       # logbook 日志封装
└── usage.md                         # easytrader 使用说明
```

---

## 2. 启动流程

```python
# trade_service.py
def main():
    broker = 'gf_client'                              # 广发证券
    log_handler = DefaultLogHandler(name='交易服务', log_type='file',
                                    filepath=log_filepath)
    m = MainEngine(broker, need_data, log_handler)    # need_data = config/trade_client.json
    m.is_watch_strategy = True                        # 热重载策略
    m.load_strategy()
    m.start()
```

`MainEngine.start()`：

1. 启动 `EventEngine`；
2. `sleep(10)` 等待账户加载；
3. 启动 `ClockEngine`；
4. 注册各引擎的 `stop` 到关闭流程。

> 运行时工作目录必须是 `instock/trade/`（因为 `MainEngine.load_strategy()` 用相对路径 `'strategies'` 扫描策略）。

---

## 3. 核心类

### 3.1 `MainEngine` — 主引擎

[main_engine.py](file:///workspace/instock/trade/robot/engine/main_engine.py)

| 成员 | 说明 |
| --- | --- |
| `__init__(broker, need_data, log_handler, tzinfo)` | `easytrader.use(broker)` + `prepare(need_data)` 登录；创建事件/时钟引擎 |
| `start()` | 启动事件引擎、等待账户、启动时钟引擎 |
| `load_strategy(names=None)` | 扫描 `strategies/*.py`，导入并实例化 `Strategy` |
| `load(names, strategy_file)` | 单文件加载/热重载（按 mtime 判断是否变化） |
| `strategy_listen_event(strategy, "listen"/"unlisten")` | 把 `strategy.clock` 注册到时钟事件 |
| `_shutdown(sig, frame)` | 捕获 `SIGINT/SIGTERM/SIGHUP/SIGQUIT`，依次执行 before/main/after shutdown 与策略 `shutdown()` |

无 `broker`/`need_data` 时进入“无交易模式”，便于脱离券商调试。

### 3.2 `EventEngine` — 事件引擎

[event_engine.py](file:///workspace/instock/trade/robot/engine/event_engine.py)

- `Queue` 存事件；
- 后台线程 `__run` 取事件，每个事件再开一个线程执行 `__process`；
- `defaultdict(list)` 维护 `event_type -> [handler]`；
- 对外：`register / unregister / put / start / stop`。

### 3.3 `ClockEngine` — 时钟引擎

[clock_engine.py](file:///workspace/instock/trade/robot/engine/clock_engine.py)

- 事件类型：`clock_tick`；
- 每 `sleep_time=1s` 触发一次 `tock()`；
- 非交易日自动暂停（`trade_time.is_trade_date`）；
- 默认注册的时刻事件：

| 类型 | 时间 | 作用 |
| --- | --- | --- |
| `open` | 09:00 | 置 `trading_state=True` |
| `pause` | 11:30 | 午休 |
| `continue` | 13:00 | 下午开盘 |
| `close` | 15:00 | 置 `trading_state=False` |

- 默认注册的间隔事件：`0.5 / 1 / 5 / 15 / 30 / 60` 分钟；
- `ClockMomentHandler`：一次性时刻，触发后滚动到下个交易日；
- `ClockIntervalHandler`：按秒取模判断是否到点，可限定仅交易时段触发；
- `push_event_type` 会把 `Clock(trading_state, clock_event)` 放入事件引擎。

### 3.4 `StrategyTemplate` — 策略基类

[strategy_template.py](file:///workspace/instock/trade/robot/infrastructure/strategy_template.py)

```python
class StrategyTemplate:
    name = 'DefaultStrategyTemplate'
    def __init__(self, user, log_handler, main_engine): ...
    def init(self): ...
    def strategy(self): ...
    def clock(self, event): ...
    def log_handler(self): return None      # 可覆盖以自定义日志
    def shutdown(self): ...
```

约定：`clock(event)` 中通过 `event.data.clock_event == self.name` 过滤本策略关注的时钟事件，然后执行 `strategy()`。

### 3.5 `DefaultLogHandler`

[default_handler.py](file:///workspace/instock/trade/robot/infrastructure/default_handler.py) 用 `logbook` 封装，支持 `stdout` 或 `file` 输出。

### 3.6 `ProcessWrapper`（预留）

[strategy_wrapper.py](file:///workspace/instock/trade/robot/infrastructure/strategy_wrapper.py) 用 `multiprocessing.Queue` + 子进程隔离策略，当前主引擎未启用。

---

## 4. 内置策略

### 4.1 `stagging.py` — 打新股

```python
def init(self):
    self.clock_engine.register_moment('stagging', dt.time(10, 0, 0, tzinfo=tz.tzlocal()))
    self.clock_engine.register_interval(1.5, trading=False)

def strategy(self):
    self.user.auto_ipo()
```

- 每个交易日 **10:00** 触发；
- 调用 `easytrader` 的 `auto_ipo()` 一键打新。

### 4.2 `stratey1.py` — 示例策略

演示买/卖/查持仓：

```python
buy_stock  = [('000001', 1, 100)]
sell_stock = [('000001', 100, 100)]
self.user.buy(...); self.user.sell(...); self.log.info(self.user.balance)
```

⚠️ 该文件仅为模板，**请勿直接实盘运行**。

---

## 5. 配置

`config/trade_client.json`：

```json
{
  "user": "888888888888",
  "password": "888888",
  "exe_path": "C:/gfzqrzrq/xiadan.exe"
}
```

- `broker` 取值参考 [usage.md](file:///workspace/instock/trade/usage.md)：`gf_client`（广发）、`ht_client`（华泰）、`gj_client`（国金）、`universal_client`（通用同花顺）、`ths`、`xq`（雪球）等。
- 验证码识别依赖 `tesseract`（需加入 PATH）。

---

## 6. 日志与生命周期

- 日志文件：`instock/log/stock_trade.log`（主引擎），策略可自定义 `log_handler()` 输出到 `instock/log/<策略名>.log`；
- 关闭流程：`before_shutdown` → `main_shutdown`（停止引擎）→ 等待线程收敛 → 各策略 `shutdown()` → `after_shutdown`。

---

## 7. 新增策略

```python
# strategies/my_strategy.py
from instock.trade.robot.infrastructure.strategy_template import StrategyTemplate

class Strategy(StrategyTemplate):
    name = 'my_strategy'

    def init(self):
        # 注册时钟：每天 14:50
        self.clock_engine.register_moment(self.name, dt.time(14, 50, tzinfo=tz.tzlocal()))

    def clock(self, event):
        if event.data.clock_event == self.name:
            self.strategy()

    def strategy(self):
        # self.user.buy(...) / self.user.sell(...)
        pass
```

放入 `strategies/` 即会自动加载；因 `is_watch_strategy=True`，修改保存后会自动热重载（生产环境不建议）。

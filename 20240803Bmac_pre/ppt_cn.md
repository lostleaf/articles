# BMAC v2: 架构、原理及展望

BMAC (**B**inance **M**arketdata **A**sync **C**lient)，即币安异步实盘行情框架，是由 lostleaf 主导，分享会成员共同参与开发的币安数据开源项目。

BMAC 项目基于 MIT 协议开源，用户可自由使用和修改，仅需保留原作者声明。

特别感谢 [LeoTsai](https://bbs.quantclass.cn/user/37703) 和 [中央勤暴菊](https://bbs.quantclass.cn/user/51516) 的贡献。

# 实盘交易的软件工程

从软件工程的角度来看，中心化交易所可视为一个大型交易服务端，而量化交易的实盘软件则是一个自动交易的**客户端软件**，其架构如下图所示：

![](image/image.001.jpeg)

整个实盘交易系统可分为三大模块：数据、策略和执行：

- 数据模块：负责获取实盘行情，仅与交易所进行单向交互。BMAC 属于此模块，用于将从交易所获取的实时行情进行解析。

- 策略模块：可细分为因子模块和仓位模块
    - BMAC 提供了配套的 BmacKit 因子开发包，用于辅助计算因子求值
    - 仓位模块则因策略而异，不同策略具有不同的仓位管理逻辑

- 执行模块：负责与交易所进行双向交互，执行多轮下单并接收成交回报，直至调整至仓位模块所要求的仓位。

# BMAC 环境与配置

详细配置说明请参考[帖子 44366](https://bbs.quantclass.cn/thread/44366)

## Conda 环境

Binance DataTool 包含 `environment.yml` 文件。

创建并激活 Conda 环境（默认环境名为 crypto）：

```
conda env create --file environment.yml
conda activate crypto
```

BMAC 基于 Python asyncio 技术，主要依赖以下库：

- `aiohttp`: REST API 请求
- `websockets`: WebSocket 数据接收
- `pandas`: DataFrame 转换与硬盘输出
- `fire`: 命令行封装

## 配置

创建基础目录（如 `~/udeli_1m`），并编写配置文件 `config.json`：

![alt text](config.png)

最小化配置示例（USDT 交割合约 1 分钟线）：

```json
{
    "interval": "1m",
    "trade_type": "usdt_deli"
}
```

## 运行

通过入口点 `python cli.py bmac start` 启动。

例如，运行 BMAC 实时获取上述 USDT 交割合约 1 分钟线行情数据：

```
python cli.py bmac start ~/udeli_1m
```

## 运行阶段1: 初始化历史数据

通过多轮历史数据下载，每轮更新 499 根 K 线，以最大化权重效率（参考[帖子34266(zdq)](https://bbs.quantclass.cn/thread/34266)），数据以 DataFrame 格式保存：

```
================== Start Bmac V2 2024-08-03 12:50:03 ===================
🔵 interval=1m, type=usdt_deli, num_candles=1500, funding_rate=False, keep_symbols=None
🔵 Candle data dir /home/admin/udeli_1m/usdt_deli_1m, initializing
🔵 Exchange info data dir /home/admin/udeli_1m/exginfo_1m, initializing
--------------- Init history round 1 2024-08-03 12:50:30 ---------------
Server time: 2024-08-03 12:50:30.122000+08:00, Used weight: 2
Symbol range: BTCUSDT_240927 -- ETHUSDT_241227
--------------- Init history round 2 2024-08-03 12:50:30 ---------------
Server time: 2024-08-03 12:50:30.261000+08:00, Used weight: 9
Symbol range: BTCUSDT_240927 -- ETHUSDT_241227
--------------- Init history round 3 2024-08-03 12:50:30 ---------------
Server time: 2024-08-03 12:50:30.421000+08:00, Used weight: 14
Symbol range: BTCUSDT_240927 -- ETHUSDT_241227
--------------- Init history round 4 2024-08-03 12:50:30 ---------------
Server time: 2024-08-03 12:50:30.599000+08:00, Used weight: 20
Symbol range: BTCUSDT_240927 -- ETHUSDT_241227
✅ 4 finished, 0 left

✅ History initialized, Server time: 2024-08-03 12:50:30.775000+08:00, Used weight: 25
```

## 运行阶段2: 实时行情更新

通过 WebSocket 接收实时行情数据：

```
Create WS listen group 0, 1 symbols
Create WS listen group 1, 1 symbols
Create WS listen group 3, 1 symbols
Create WS listen group 5, 1 symbols
====== Bmac 1m usdt_deli update Runtime=2024-08-03 12:51:00+08:00 ======
✅ 2024-08-03 12:51:00.000132+08:00, Exchange infos updated

2024-08-03 12:51:00.046084+08:00, 0/4 symbols ready
2024-08-03 12:51:01.006133+08:00, 1/4 symbols ready
2024-08-03 12:51:02.008255+08:00, 1/4 symbols ready
2024-08-03 12:51:03.009653+08:00, 1/4 symbols ready
✅ 2024-08-03 12:51:04.010863+08:00, all symbols ready

🔵 Last updated ETHUSDT_241227 2024-08-03 12:51:03.067731+08:00
====== Bmac 1m usdt_deli update Runtime=2024-08-03 12:52:00+08:00 ======
...
```

## 运行目录结构

- exginfo_1m: 可交易标的信息
- usdt_deli_1m: K 线行情
    - ready file: 文件锁
    - pqt 文件: 以 parquet 格式序列化的 Pandas DataFrame

![](folder.png)

## 核心参数

两个核心参数：

`interval`: K 线时间周期，支持币安官方提供的周期，如 1m、5m、1h、4h 等。

`trade_type`: 交易标的类型
- `usdt_spot`: USDT 本位现货，如 `BTCUSDT`、`ETHUSDT` 等
- `btc_spot`: BTC 本位现货，如 `ETHBTC` 等
- `usdt_perp`: USDT 本位永续，如 `BTCUSDT` 永续等
- `coin_perp`: 币本位永续，如 `BTCUSD` 币本位永续等

详细说明请参考[帖子44366](https://bbs.quantclass.cn/thread/44366)的**核心参数**部分。

## 可选参数

`num_candles`: 保留 K 线数量，默认 1500，上限为 10000。

`funding_rate`: 是否获取资金费率，默认为 False。

`keep_symbols`: symbol 白名单，如设置则仅获取白名单内的 symbol，默认为 None。

`save_type`: K 线数据存储格式，默认为 parquet，也可选择 feather。

`dingding`: 钉钉配置，默认为 None。

```json
"dingding": {
    "error": {
        "access_token": "f...",
        "secret": "SEC..."
    }
}
```

详细说明请参考[帖子44366](https://bbs.quantclass.cn/thread/44366)的**可选参数**部分。

# BMAC 原理简介

## 初始化历史数据

与邢大基础课程原理类似，不涉及 WebSocket。

通过 REST API 获取历史数据，需注意控制权重，分批获取。

参考[帖子35389](https://bbs.quantclass.cn/thread/35389)。

## 实盘数据更新：多协程，生产者-消费者架构

生产者负责接收和提供数据：

- `CandleListener`: 通过 WebSocket 接收行情数据推送，基于 python-binance 实现
- `RestFetcher`: 通过 REST API 拉取行情数据，同时作为 K 线拉取命令的消费者
- `PeriodAlarm`: 发出拉取 ExgInfo、检查数据完整性的命令，相当于 Runtime 循环

消费者 `Dispatcher` 负责处理生产者提供的数据，只有消费者可以访问硬盘，防止读写冲突：

- 执行拉取 ExgInfo 命令，写入硬盘，当有变动时调整 `CandleListener` 订阅
- 处理行情数据、资金费率等，写入硬盘
- 检查行情数据完整性，如有缺失，发出 K 线拉取命令

生产者和消费者通过队列进行通信：
- 主队列 `main_que`: 生产者和 `Dispatcher` 之间的通信
- REST 队列 `rest_que`: `Dispatcher` 和 `RestFetcher` 之间的通信

## 数据通路

![](image/image.002.jpeg)

# BmacKit：因子计算开发包

[帖子 44984](https://bbs.quantclass.cn/thread/44984)

## BmacSingleSymbolCalculator

单标的多因子计算器，适用于时序趋势类策略：

```python
class BmacSingleSymbolCalculator:

    def __init__(self,
                 symbol: str,
                 candle_reader: CandleFileReader,
                 factor_cfgs: list,
                 package: str = 'factor',
                 bmac_expire_sec: int = 40):
        """
        symbol: 标的名称
        candle_reader: K 线存放目录的 CandleFileReader
        factor_cfgs: 因子列表，例如 [('PctChg', 100), ('TrdNumMeanV1', 80)]
        package: 因子包名，默认为 'factor'
        bmac_expire_sec: BMAC 超时时间(秒)，默认 40 秒
        """
        ...

    async def calc_factors(self, run_time: datetime, symbol=None) -> pd.DataFrame:
        """
        run_time: 当前周期时间戳
    
        返回值: 包含给定 symbol 所有周期所有因子的 DataFrame
        """
        ...
```

## BmacSingleSymbolCalculator 案例

```python
# 导入 BmacKit 
from bmac_kit import BmacSingleSymbolCalculator, CandleFileReader, now_time
# 运行周期
TIME_INTERVAL = '5m'
# BMAC 目录
CANDLE_DIR = '../usdt_perp_5m_all_v2/usdt_perp_5m'
# 因子列表
FACTOR_LIST = [('PctChg', 100), ('TrdNumMeanV1', 80)]
```

```python
# 当前 run_time
run_time = next_run_time(TIME_INTERVAL)
# 初始化 CandleFileReader
candle_reader = CandleFileReader(CANDLE_DIR, 'parquet')
# 初始化 BmacKit 因子计算器
calc = BmacSingleSymbolCalculator('BTCUSDT', candle_reader, FACTOR_LIST)
# 测试因子计算
df_factor_single = await calc.calc_factors(run_time)
```

## BmacSingleSymbolCalculator 计算结果

![alt text](single.png)

得益于 WebSocket 的使用，BTCUSDT 因子计算可在 1 秒内完成。

## BmacAllMarketCalculator

全市场多标的多因子计算器，适用于截面选币类策略：

```python
class BmacAllMarketCalculator(BmacSingleSymbolCalculator):

    def __init__(self,
                 exginfo_reader: CandleFileReader,
                 candle_reader: CandleFileReader,
                 factor_cfgs: list,
                 package: str = 'factor',
                 bmac_expire_sec: int = 40):
        """
        exginfo_reader: exchange info 存放目录的 CandleFileReader
        candle_reader: K 线存放目录的 CandleFileReader
        factor_cfgs: 因子列表，例如 [('PctChg', 100), ('TrdNumMeanV1', 80)]
        package: 因子包名，默认为 'factor'
        bmac_expire_sec: BMAC 超时时间(秒)，默认 40 秒
        """

    async def calc_all_factors(self, run_time: datetime) -> pd.DataFrame:
        """
        run_time: 当前周期时间戳
    
        返回值: 包含给定全市场 run_time 周期所有因子的 DataFrame
        """
```

## BmacAllMarketCalculator 案例

导入和因子定义与 BmacSingleSymbolCalculator 相同：

```python
# 当前 run_time
run_time = next_run_time(TIME_INTERVAL)

# 初始化 CandleFileReader
exginfo_reader = CandleFileReader(EXGINFO_DIR, 'parquet')
candle_reader = CandleFileReader(CANDLE_DIR, 'parquet')

# 初始化 BmacKit 因子计算器
all_calc = BmacAllMarketCalculator(exginfo_reader, candle_reader, FACTOR_LIST)

# 测试因子计算
df_factor_all = await all_calc.calc_all_factors(run_time)
```

异步计算全市场因子，几乎没有额外延迟。

![alt](update.png)

![alt](all_market.png)

## 反思, BMAC v2 足够好吗？

BMAC v2 的优势：使用 WebSocket，减少权重消耗

BMAC v2 的局限性：
- 单线程多协程架构
- 存在硬盘 IO 瓶颈

建议的优化架构（适用于中高频交易/日内CTA等）—— 微服务化（多进程）：
- 基于消息队列实现服务间通信，例如 ZMQ pub/sub 模式
- 行情接收端：基于 WebSocket + REST 获取行情，通过 ZMQ pub 广播
- 记录端：通过 ZMQ sub 接收并录制历史行情，写入硬盘/数据库
- 策略端：从硬盘或数据库初始化历史行情，再通过 ZMQ sub 接收行情进行交易
- BmacKit：考虑使用流式在线计算，可能需要放弃 Pandas 并采用 (JIT)编译型语言

注：此架构不适用于超高频交易（如币圈做市类策略）。

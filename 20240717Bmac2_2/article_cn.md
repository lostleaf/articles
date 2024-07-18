# 【BMAC2.0-正传（上）】BMAC2的配置与使用

BMAC，币安行情数据异步客户端，是币安数据框架 Binance DataTool 中的实盘数据服务。Binance DataTool 是由 lostleaf.eth 主导开发，分享会的同学们参与贡献的开源项目，目前托管在 GitHub [github.com/lostleaf/binance_datatool](https://github.com/lostleaf/binance_datatool)，欢迎 Star & Fork。

本文主要介绍 BMAC2 的参数设置与实盘使用。

## 运行环境

Binance DataTool 自带运行环境配置文件 `environment.yml`。

使用以下命令即可创建 conda 环境并激活，环境名默认为 `crypto`：

```bash
conda env create --file environment.yml
conda activate crypto
```

BMAC 主要依赖于 Pandas、aiohttp 和 websockets。与 BHDS 不同，BMAC 并不依赖于 `aria2`。

## 配置

要使用 BMAC，首先编写配置文件。

第一步是建立一个新文件夹，作为基础目录，例如 `~/udeli_1m`。

然后在新建的文件夹下编写配置文件 `config.json`，一个最小化的配置如下：

```json
{
    "interval": "1m",
    "trade_type": "usdt_deli"
}
```

BMAC 将根据该配置接收 USDT 本位交割合约 1 分钟 K 线。

## 运行

Binance DataTool 的入口点统一为 `cli.py`，BMAC2 的入口点为 `python cli.py bmac start`，例如：

```bash
python cli.py bmac start ~/udeli_1m
```

运行时会打印日志如下：

```
================== Start Bmac V2 2024-07-17 19:33:21 ===================
🔵 interval=1m, type=usdt_deli, num_candles=1500, funding_rate=False, keep_symbols=None
🔵 Candle data dir /Users/lostleaf/udeli_1m/usdt_deli_1m, initializing
🔵 Exchange info data dir /Users/lostleaf/udeli_1m/exginfo_1m, initializing
--------------- Init history round 1 2024-07-17 19:33:30 ---------------
Server time: 2024-07-17 19:33:30.805000+08:00, Used weight: 7
Symbol range: BTCUSDT_240927 -- ETHUSDT_241227
--------------- Init history round 2 2024-07-17 19:33:31 ---------------
Server time: 2024-07-17 19:33:31.609000+08:00, Used weight: 14
Symbol range: BTCUSDT_240927 -- ETHUSDT_241227
--------------- Init history round 3 2024-07-17 19:33:31 ---------------
Server time: 2024-07-17 19:33:31.876000+08:00, Used weight: 20
Symbol range: BTCUSDT_240927 -- ETHUSDT_241227
--------------- Init history round 4 2024-07-17 19:33:32 ---------------
Server time: 2024-07-17 19:33:32.139000+08:00, Used weight: 29
Symbol range: BTCUSDT_240927 -- ETHUSDT_241227
✅ 4 finished, 0 left

✅ History initialized, Server time: 2024-07-17 19:33:32.404000+08:00, Used weight: 36

Create WS listen group 1, 1 symbols
Create WS listen group 3, 1 symbols
Create WS listen group 5, 1 symbols
Create WS listen group 6, 1 symbols
====== Bmac 1m usdt_deli update Runtime=2024-07-17 19:34:00+08:00 ======
✅ 2024-07-17 19:34:00.000823+08:00, Exchange infos updated

2024-07-17 19:34:00.188033+08:00, 0/4 symbols ready
2024-07-17 19:34:01.008114+08:00, 1/4 symbols ready
2024-07-17 19:34:02.008851+08:00, 2/4 symbols ready
2024-07-17 19:34:03.010863+08:00, 2/4 symbols ready
✅ 2024-07-17 19:34:04.012457+08:00, all symbols ready

🔵 Last updated ETHUSDT_241227 2024-07-17 19:34:03.047949+08:00
====== Bmac 1m usdt_deli update Runtime=2024-07-17 19:35:00+08:00 ======
✅ 2024-07-17 19:35:00.000843+08:00, Exchange infos updated

2024-07-17 19:35:00.741359+08:00, 0/4 symbols ready
2024-07-17 19:35:01.009967+08:00, 1/4 symbols ready
2024-07-17 19:35:02.011727+08:00, 1/4 symbols ready
2024-07-17 19:35:03.008437+08:00, 1/4 symbols ready
✅ 2024-07-17 19:35:04.012293+08:00, all symbols ready

🔵 Last updated ETHUSDT_241227 2024-07-17 19:35:03.073522+08:00
```

由于使用了西大的 LogKit，在命令行下不仅有 Emoji 提示，还有不同颜色，为西大点赞。

如日志所示，BMAC 会首先通过 REST API 初始化历史数据，然后通过订阅 websocket 更新数据。

运行中的目录结构如下：

```
udeli_1m
├── config.json
├── exginfo_1m
│   ├── exginfo.pqt
│   └── exginfo_20240717_193700.ready
└── usdt_deli_1m
    ├── BTCUSDT_240927.pqt
    ├── BTCUSDT_240927_20240717_193700.ready
    ├── BTCUSDT_241227.pqt
    ├── BTCUSDT_241227_20240717_193700.ready
    ├── ETHUSDT_240927.pqt
    ├── ETHUSDT_240927_20240717_193700.ready
    ├── ETHUSDT_241227.pqt
    └── ETHUSDT_241227_20240717_193700.ready
```

## 核心参数

BMAC2 主要包含两个核心参数，`interval` 和 `trade_type`，分别代表 K 线时间周期和交易标的类型。

其中 `interval` 可以是 `1m`、`5m`、`1h`、`4h` 等币安官方支持的周期。

`trade_type` 可选项较多，定义如下，包括不同类型的现货，U本位合约与币本位合约。

```python
{
    # spot
    'usdt_spot': (TradingSpotFilter(quote_asset='USDT', keep_stablecoins=False), 'spot'),
    'usdc_spot': (TradingSpotFilter(quote_asset='USDC', keep_stablecoins=False), 'spot'),
    'btc_spot': (TradingSpotFilter(quote_asset='BTC', keep_stablecoins=False), 'spot'),

    # usdt_futures
    'usdt_perp': (TradingUsdtFuturesFilter(quote_asset='USDT', types=['PERPETUAL']), 'usdt_futures'),
    'usdt_deli': (TradingUsdtFuturesFilter(quote_asset='USDT', types=DELIVERY_TYPES), 'usdt_futures'),
    'usdc_perp': (TradingUsdtFuturesFilter(quote_asset='USDC', types=['PERPETUAL']), 'usdt_futures'),

    # 仅包含 ETHBTC 永续合约，属于 U 本位合约
    'btc_perp': (TradingUsdtFuturesFilter(quote_asset='BTC', types=['PERPETUAL']), 'usdt_futures'),

    # 兼容 V1
    'usdt_swap': (TradingUsdtFuturesFilter(quote_asset='USDT', types=['PERPETUAL']), 'usdt_futures'),

    # coin_futures
    'coin_perp': (TradingCoinFuturesFilter(types=['PERPETUAL']), 'coin_futures'),
    'coin_deli': (TradingCoinFuturesFilter(types=DELIVERY_TYPES), 'coin_futures'),

    # 兼容 V1
    'coin_swap': (TradingCoinFuturesFilter(types=['PERPETUAL']), 'coin_futures'),
}
```

## 可选参数

BMAC2 包含多个可选参数，参考 `handler.py` 中参数定义如下：

```python
# 可选参数

# 保留 K 线数量, 默认1500
self.num_candles = cfg.get('num_candles', 1500)
# 是否获取资金费率，默认否
self.fetch_funding_rate = cfg.get('funding_rate', False)
# http 超时时间，默认 5 秒
self.http_timeout_sec = int(cfg.get('http_timeout_sec', 5))
# K 线闭合超时时间，默认 15 秒
self.candle_close_timeout_sec = int(cfg.get('candle_close_timeout_sec', 15))
# symbol 白名单，如有则只获取白名单内的 symbol，默认无
self.keep_symbols = cfg.get('keep_symbols', None)
# K 线数据存储格式，默认 parquet，也可为 feather
save_type = cfg.get('save_type', 'parquet')
# 钉钉配置，默认无
self.dingding = cfg.get('dingding', None)
# rest fetcher 数量
self.num_rest_fetchers = cfg.get('num_rest_fetchers', 8)
# websocket listener 数量
self.num_socket_listeners = cfg.get('num_socket_listeners', 8)
```

也可以参考 `bmac_example` 目录下的例子进行配置，例如 `bmac_example/usdt_perp_5m_all/config.json.example`。

较为重要的可选参数包括：`num_candles`、`funding_rate`、`dingding`。

如果有老板习惯使用 BMAC1 的 `feather` 格式，可以将 `save_type` 改为 `feather`。

如果仅仅需要特定的交易标的，可以设置 `keep_symbols`。

小提示：BMAC2 默认限制 `num_candles` 参数最大值为 10000。该限制通过 `handler.py` 中的 `NUM_CANDLES_MAX_LIMIT` 常数实现，超过则会报错。当然，由于 BMAC 属于开源项目，这只是一个善意的警告 —— 过大的 `num_candles` 有可能严重影响 BMAC2 的运行效率并导致错误。如果您确实有需求并愿意自行承担相应后果，您可以修改该限制为更大的数目甚至无穷大。

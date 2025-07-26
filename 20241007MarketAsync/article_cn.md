# 【BMAC 前传之二】基于 asyncio 实现高效批量下载币安历史 5 分钟线数据，并转换为全 offset 小时线

本文主要介绍如何基于 Python 的 `asyncio` 库，异步封装币安行情 API 和 Quantclass Data API，并在此基础上实现批量下载历史 5 分钟线数据，并将其重采样为全 offset 小时线。

文章主要分为以下 3 个部分：

1. 介绍币安行情 API 的封装 `BinanceFetcher` 和 Quantclass Data API `QuantclassDataApi`；
2. 基于 `BinanceFetcher`，以并发方式批量下载全市场的历史 5 分钟线数据，并将其保存为 pkl 文件，由于 U 本位合约权重限制，该方案已经达到了下载速度的理论最大值；
3. 将上一步保存的本地历史数据重采样（Resample）为全 offset 小时线数据，并使用 `QuantclassDataApi` 获取 Quantclass 提供的小时线数据，对两份数据进行对比分析。

## 数据 API 异步封装

### 币安行情 API `BinanceFetcher`

本节将介绍 BMAC 中的币安行情 API 的异步封装。实际上，该封装分为两层：

1. 对币安数据 API 进行的浅层调用封装（参考了著名的开源项目 `python-binance`）：包括 `BinanceMarketUMFapi`、`BinanceMarketCMDapi` 和 `BinanceMarketSpotApi`。
2. 对币安数据 API 进行进一步的封装和深度解析，即 `BinanceFetcher`。

在本篇文章中，我们将主要使用 `BinanceFetcher`，其主要 API 包括以下几个部分：

- **构造**：首先，通过 `create_aiohttp_session` 创建一个 `aiohttp` 的 `ClientSession` 对象，并使用该对象来构造 `BinanceFetcher`。
- **`get_time_and_weight`**：获取服务器的当前时间以及当前分钟已消耗的 API 权重。
- **`get_exchange_info`**：获取交易规则（Exchange Info）。
- **`get_candle`**：获取 K 线数据，返回结果为 `Pandas` 的 `DataFrame`。

其调用方式如下所示 (ex1.py)：

```python
async def test_binance():
    print('\nTesting Binance api\n')

    # 初始化 aiohttp session 和 U 本位合约 BinanceFetcher
    async with create_aiohttp_session(timeout_sec=3) as session:
        fetcher = BinanceFetcher('usdt_futures', session) # U 本位合约

        # fetcher = BinanceFetcher('spot', session) # 现货

        # 获取服务器时间和已使用的权重
        server_timestamp, weight = await fetcher.get_time_and_weight()
        print(f'Call get_time_and_weight: server_time={server_timestamp}, used_weight={weight}')

        # 获取交易规则 exchange info
        exg_info = await fetcher.get_exchange_info()

        # 打印 BTCUSDT 交易规则
        print('\nCall get_exchange_info, BTCUSDT Exchange Info:')
        pp(exg_info['BTCUSDT'])

        # 获取并打印 BTCUSDT 5 分钟 K 线
        btc_candle = await fetcher.get_candle(symbol='BTCUSDT', interval='5m')
        print('\nCall get_candle, BTCUSDT Candle:')
        print(btc_candle)
```

输出如下

```
Testing Binance api

Call get_time_and_weight: server_time=2024-10-07 12:07:40.571000+00:00, used_weight=15

Call get_exchange_info, BTCUSDT Exchange Info:
{'symbol': 'BTCUSDT',
 'contract_type': 'PERPETUAL',
 'status': 'TRADING',
 'base_asset': 'BTC',
 'quote_asset': 'USDT',
 'margin_asset': 'USDT',
 'price_tick': Decimal('0.10'),
 'lot_size': Decimal('0.001'),
 'min_notional_value': Decimal('100')}

Call get_candle, BTCUSDT Candle:
                                  candle_begin_time     open     high      low    close    volume  quote_volume  trade_num  taker_buy_base_asset_volume  taker_buy_quote_asset_volume
candle_end_time
2024-10-05 18:35:00+00:00 2024-10-05 18:30:00+00:00  61879.6  61896.0  61860.5  61886.5   113.847  7.044376e+06     2700.0                       47.390                  2.932401e+06
2024-10-05 18:40:00+00:00 2024-10-05 18:35:00+00:00  61886.5  61908.0  61860.6  61866.7   132.708  8.213261e+06     3178.0                       55.010                  3.404629e+06
2024-10-05 18:45:00+00:00 2024-10-05 18:40:00+00:00  61866.7  61876.4  61850.0  61860.0   272.957  1.688524e+07     3167.0                      107.109                  6.625791e+06
2024-10-05 18:50:00+00:00 2024-10-05 18:45:00+00:00  61860.0  61869.6  61801.6  61803.3   456.886  2.825147e+07     6118.0                       86.445                  5.344708e+06
2024-10-05 18:55:00+00:00 2024-10-05 18:50:00+00:00  61803.4  61846.3  61802.1  61810.1   243.954  1.508172e+07     4039.0                      123.476                  7.633612e+06
...                                             ...      ...      ...      ...      ...       ...           ...        ...                          ...                           ...
2024-10-07 11:50:00+00:00 2024-10-07 11:45:00+00:00  62861.7  62888.7  62826.8  62852.7   464.550  2.919981e+07     7088.0                      222.564                  1.398886e+07
2024-10-07 11:55:00+00:00 2024-10-07 11:50:00+00:00  62852.8  62864.1  62822.3  62835.8   203.233  1.277135e+07     4467.0                      102.407                  6.435439e+06
2024-10-07 12:00:00+00:00 2024-10-07 11:55:00+00:00  62835.9  63171.0  62831.0  63114.1  3421.162  2.156247e+08    31730.0                     2236.709                  1.409595e+08
2024-10-07 12:05:00+00:00 2024-10-07 12:00:00+00:00  63114.1  63116.5  62970.0  62983.9  1093.081  6.889721e+07    15365.0                      420.038                  2.647430e+07
2024-10-07 12:10:00+00:00 2024-10-07 12:05:00+00:00  62983.8  63000.0  62902.9  62996.6   483.196  3.041208e+07     7142.0                      170.908                  1.075785e+07

[500 rows x 10 columns]
```

### Quantclass 数据 API `QuantclassDataApi`

`QuantclassDataApi` 的主要 API 包括以下几个部分：

- **构造**：首先，通过 `create_aiohttp_session` 创建一个 `aiohttp` 的 `ClientSession` 对象，并使用该对象来构造 `QuantclassDataApi`。请注意，调用 `QuantclassDataApi` 需要提供**葫芦ID**（UUID）和 **API Key**，这些信息可以在个人页面中获取。
- **`aioreq_data_api`**：获取数据的 K 线下载地址以及最新的时间戳。
- **`aioreq_candle_df`**：下载最新数据，并将其转换为 `Pandas` 的 `DataFrame`。

其调用方式如下所示 (ex1.py)：
```python
async def test_quantclass():
    print('\nTesting Quantclass api\n')

    # 初始化 aiohttp session 和 U 本位合约 QuantclassDataApi
    async with create_aiohttp_session(timeout_sec=3) as session:
        quantclass_api = QuantclassDataApi(session, API_KEY, UUID)

        # 获取并打印 0m offset 最新下载地址和时间戳
        url_data = await quantclass_api.aioreq_data_api('0m')
        print('Call aioreq_data_api:')
        pp(url_data)

        # 获取现货 K 线数据
        df_spot = await quantclass_api.aioreq_candle_df(url_data['spot'])
        print('\nCall aioreq_candle_df, spot candles:')
        print(df_spot.head())

        # 获取合约 K 线数据
        df_swap = await quantclass_api.aioreq_candle_df(url_data['swap'])
        print('\nCall aioreq_candle_df, swap candles:')
        print(df_swap.head())
```

输出如下

```
Testing Quantclass api

Call aioreq_data_api:
{'spot': 'https://upyun.quantclass.cn/crypto-realtime/binance-1h/202410081500-spot_1h0m.7z?_upt=ac80946d1728383062',
 'swap': 'https://upyun.quantclass.cn/crypto-realtime/binance-1h/202410081500-swap_1h0m.7z?_upt=1a13e4771728383062',
 'ts': datetime.datetime(2024, 10, 8, 15, 0, tzinfo=<DstTzInfo 'Asia/Shanghai' CST+8:00:00 STD>)}

Call aioreq_candle_df, spot candles:
          candle_begin_time   symbol      open      high       low     close      volume  quote_volume  trade_num  taker_buy_base_asset_volume  taker_buy_quote_asset_volume      tag
0 2024-10-08 05:00:00+00:00  BTCUSDT  62694.90  62723.69  62305.78  62517.35   719.31657  4.495193e+07   125544.0                    305.50545                  1.908714e+07  HasSwap
1 2024-10-08 06:00:00+00:00  BTCUSDT  62517.35  62531.52  62200.00  62509.42   580.02424  3.617911e+07   120901.0                    214.49443                  1.337989e+07  HasSwap
2 2024-10-08 05:00:00+00:00  ETHUSDT   2437.44   2439.04   2419.18   2428.40  8726.54880  2.117588e+07    97232.0                   4276.72020                  1.037482e+07  HasSwap
3 2024-10-08 06:00:00+00:00  ETHUSDT   2428.40   2438.75   2415.80   2435.66  8152.23490  1.977390e+07    93730.0                   4252.85230                  1.031975e+07  HasSwap
4 2024-10-08 05:00:00+00:00  BNBUSDT    569.50    570.00    563.00    565.10  9232.96800  5.225820e+06    25696.0                   4097.68700                  2.319457e+06  HasSwap

Call aioreq_candle_df, swap candles:
          candle_begin_time   symbol      open      high       low     close      volume  quote_volume  trade_num  taker_buy_base_asset_volume  taker_buy_quote_asset_volume     tag
0 2024-10-08 05:00:00+00:00  BTCUSDT  62666.30  62700.00  62280.20  62500.00    9618.996  6.007619e+08   102684.0                     4227.549                  2.640030e+08  NoSwap
1 2024-10-08 06:00:00+00:00  BTCUSDT  62500.10  62513.80  62160.00  62483.50    8164.564  5.089668e+08   112671.0                     4116.852                  2.566678e+08  NoSwap
2 2024-10-08 05:00:00+00:00  ETHUSDT   2436.59   2438.08   2417.91   2427.25  112813.895  2.736156e+08   132500.0                    57041.867                  1.383386e+08  NoSwap
3 2024-10-08 06:00:00+00:00  ETHUSDT   2427.25   2438.42   2414.38   2434.44  105897.214  2.568499e+08   135580.0                    51083.870                  1.239558e+08  NoSwap
4 2024-10-08 05:00:00+00:00  BCHUSDT    326.60    327.06    323.02    324.28    8068.297  2.616326e+06    11032.0                     4507.322                  1.461325e+06  NoSwap
```

## 批量下载并保存全市场的历史 5 分钟线数据

该下载过程可分为以下三个步骤：

1. 初始化各项参数
2. 获取全市场交易对列表（按字母排序）
3. 下载并保存数据

由于不同交易品种的权重需分别计算，我们在 `main` 函数中使用了 `asyncio.gather` 方法，以同时下载现货和 U 本位合约数据。

```python

async def download_history(session: aiohttp.ClientSession, trade_type):
    # 1 初始化各项参数
    fetcher, interval_delta, run_time, max_minute_weight, num_candles, candle_dir, logger = prepare_init(
        session, trade_type)

    logger.info('type=%s, interval=%s, num=%d, max_weight=%d, run_time=%s, candle_dir=%s', trade_type, INTERVAL,
                num_candles, max_minute_weight, run_time, candle_dir)

    # 2 获取交易对列表（按字母排序）
    symbols = await get_trading_usdt_symbols(fetcher)

    logger.info('type=%s, num_symbols=%d, first=%s, last=%s', trade_type, len(symbols), symbols[0], symbols[-1])

    # 至少距离 run_time least_past_sec 秒，防止当前时间距 run_time 太近导致 K 线不闭合
    least_past_sec = 45
    if now_time() - run_time < pd.Timedelta(seconds=least_past_sec):
        t = least_past_sec - (now_time() - run_time).total_seconds()
        await asyncio.sleep(t)

    # 3 下载并保存数据
    await run_download(fetcher, interval_delta, run_time, max_minute_weight, num_candles, candle_dir, logger, symbols,
                       trade_type)


async def main():
    t_start = time.time()
    divider('Start initialize historical candles')

    # 同时下载现货和 U 本位合约数据
    async with create_aiohttp_session(timeout_sec=3) as session:
        await asyncio.gather(download_history(session, 'spot'), download_history(session, 'usdt_futures'))

    t_min = round((time.time() - t_start) / 60, 2)
    divider(f'Finished in {t_min}mins')


if __name__ == '__main__':
    HISTORY_DAYS = 15
    INTERVAL = '5m'
    asyncio.run(main())
```

### 初始化各项参数 `prepare_init`

该函数逻辑相对简单，主要包括创建 `BinanceFetcher`、日志记录器（logger）、存储目录，并计算其他相关参数。

```python
def prepare_init(session, trade_type):
    # 创建一个 BinanceFetcher 实例
    fetcher = BinanceFetcher(trade_type, session)

    # 计算需要获取的K线数据的数量 (5分钟间隔，一天288个K线)
    num_candles = HISTORY_DAYS * 288 + 20

    # 将时间间隔字符串转换为 timedelta 对象
    interval_delta = convert_interval_to_timedelta(INTERVAL)

    # 计算本次运行的 run_time
    run_time = next_run_time(INTERVAL) - interval_delta

    # 获取 API 权重
    max_minute_weight, _ = fetcher.get_api_limits()

    # 获取 logger (西大 log_kit)
    logger = get_logger()

    # 生成保存K线数据的目录名
    candle_dir = os.path.join('data', f'{trade_type}_5m')

    # 如果目录存在，删除该目录
    if os.path.exists(candle_dir):
        shutil.rmtree(candle_dir)

    # 创建新的目录用于存储K线数据
    os.makedirs(candle_dir)

    # 返回初始化后的各项参数
    return fetcher, interval_delta, run_time, max_minute_weight, num_candles, candle_dir, logger
```

### 获取交易对列表 `get_trading_usdt_symbols`

在这一部分，我们将对全市场所有交易对进行过滤，具体步骤为：仅保留状态为 "TRADING" 的 USDT 本位交易对；对于合约交易，只保留类型为 "PERPETUAL" 的永续合约。

代码如下：

```python
def is_valid_symbol(info):
    # 如果交易对的计价资产不是 USDT，返回 False
    if info['quote_asset'] != 'USDT':
        return False

    # 如果交易对的状态不是正在交易(“TRADING”)，返回False
    if info['status'] != 'TRADING':
        return False

    # 对于合约，如果不是永续合约，返回False
    if 'contract_type' in info and info['contract_type'] != 'PERPETUAL':
        return False

    return True  # 如果满足上述所有条件，则返回True
```

通过以下代码调用 `get_exchange_info` 函数，并取出我们需要的交易对

```python
async def get_trading_usdt_symbols(fetcher: BinanceFetcher):
    # 异步获取交易所信息
    exginfo = await fetcher.get_exchange_info()

    # 过滤交易对
    symbols_trading: list = []
    for symbol, info in exginfo.items():
        if is_valid_symbol(info):
            symbols_trading.append(symbol)

    # 返回按字母排序的交易对列表
    return sorted(symbols_trading)
```

### 下载并保存数据 `run_download`

这一部分的逻辑较为复杂，需要循环分批获取每个 symbol 的历史数据。每个循环包含以下 5 个步骤：

1. 获取当前的权重和服务器时间。如果已使用的权重超过最大限额的 90%，则 `sleep` 直至下一分钟。
2. 每轮从剩余 symbol 中选择 80 个，预计消耗权重为 160。
3. 为本轮需要获取的 symbols 创建获取 K 线数据的任务。
4. 并发执行任务并存储获取到的 K 线数据，预计消耗权重为 160。
5. 更新每个 symbol 的状态：如果已经获取了足够的 K 线数据，或者 symbol 上市时间较短导致数据不足，则无需继续获取。

代码如下：

```python
async def run_download(fetcher: BinanceFetcher, interval_delta, run_time, max_minute_weight, num_candles, candle_dir,
                       logger, symbols, trade_type):
    round = 0
    last_begin_time = dict()

    # 循环分批获取每个 symbol 历史数据
    while symbols:
        # 1 获取当前的权重和服务器时间。如果已使用的权重超过最大限额的 90%，则 sleep 直至下一分钟
        server_time, weight = await fetcher.get_time_and_weight()
        if weight > max_minute_weight * 0.9:
            await async_sleep_until_run_time(next_run_time('1m'))
            continue

        # 2 每轮从剩余 symbol 中选择 80 个
        fetch_symbols = symbols[:80]
        round += 1
        server_time = server_time.tz_convert(DEFAULT_TZ)

        logger.debug((f'{trade_type} round {round}, server_time={server_time}, used_weight={weight}, '
                      f'symbols={fetch_symbols[0]} -- {fetch_symbols[-1]}'))

        # 3 为本轮需要获取的 symbols 创建获取 K 线数据的任务
        tasks = []
        for symbol in fetch_symbols:
            # 默认还没有被获取过
            end_timestamp = None

            # 已经获取过，接着上次比上次已经获取过更旧的 limit 根
            if symbol in last_begin_time:
                end_timestamp = (last_begin_time[symbol] - interval_delta).value // 1000000
            t = fetch_and_save_history_candle(candle_dir, fetcher, symbol, num_candles, end_timestamp, run_time)
            tasks.append(t)

        # 4 并发执行任务并存储获取到的 K 线数据，预计消耗权重为 160
        results = await asyncio.gather(*tasks)

        # 5 更新每个 symbol 的状态
        num_finished = 0
        num_not_enough = 0
        for symbol, (not_enough, begin_time, num) in zip(fetch_symbols, results):
            last_begin_time[symbol] = begin_time

            # 如果已经获取了足够的 K 线数据，或者 symbol 上市时间较短导致数据不足，则无需继续获取
            if num >= num_candles or not_enough:
                symbols.remove(symbol)

                if not_enough:
                    logger.warning('%s %s candle not enough, num=%d', trade_type, symbol, num)
                    num_not_enough += 1
                else:
                    num_finished += 1

    # 完成历史 K 线下载
    server_time, weight = await fetcher.get_time_and_weight()
    server_time = server_time.tz_convert(DEFAULT_TZ)
    logger.ok('%s initialized, server_time=%s, used_weight=%d', trade_type, server_time, weight)
```

其中，`fetch_and_save_history_candle` 函数负责从指定交易对的 `end_timestamp` 开始，按时间倒序获取 K 线数据，并将其存储在 `candle_dir` 目录下。

代码如下：

```python
async def fetch_and_save_history_candle(candle_dir, fetcher: BinanceFetcher, symbol, num_candles, end_timestamp,
                                        run_time):
    # 获取API限制中一次能获取的K线数量
    _, once_candles = fetcher.get_api_limits()

    if end_timestamp is None:
        # 如果没有指定结束时间戳，则获取最新的K线数据
        df_new = await fetcher.get_candle(symbol, INTERVAL, limit=once_candles)
    else:
        # 否则获取指定结束时间之前的K线数据
        df_new = await fetcher.get_candle(symbol, INTERVAL, limit=once_candles, endTime=end_timestamp)

    # 判断获取的K线数量是否不足一次能获取的最大数量
    not_enough = df_new.shape[0] < once_candles

    # 过滤数据，使其只保留到指定运行时间的K线，并按时间顺序排序
    df_new = df_new.loc[:run_time].sort_index()

    # 生成保存K线数据的文件路径
    df_path = os.path.join(candle_dir, f'{symbol}.pkl.zst')
    if os.path.exists(df_path):
        # 如果文件已存在，读取旧的K线数据并与新数据拼接
        df_old = pd.read_pickle(df_path)
        df: pd.DataFrame = pd.concat([df_new, df_old])
    else:
        # 如果文件不存在，则只保存新获取的数据
        df = df_new

    # 删除重复的K线数据，以“candle_begin_time”为基准，只保留最新的数据
    df.drop_duplicates(subset='candle_begin_time', keep='first', inplace=True)

    # 只保留最新的num_candles条K线数据
    df = df.iloc[-num_candles:]

    # 将处理后的数据保存为pkl格式文件
    df.to_pickle(df_path)

    # 获取最早的K线时间戳
    min_begin_time = df['candle_begin_time'].min()

    # 获取保存的数据条数
    num = len(df)

    # 返回是否获取的数据不足、最早的K线时间戳和K线数量
    return not_enough, min_begin_time, num
```

### 运行示例

运行 `ex2.py`，下载最近 15 日的 5 分钟线，关键日志如下

```
============ Start initialize historical candles 2024-10-08 00:06:39 =============
🌀 type=spot, interval=5m, num=4340, max_weight=6000, run_time=2024-10-08 00:05:00+08:00, candle_dir=data/spot_5m
🌀 type=usdt_futures, interval=5m, num=4340, max_weight=2400, run_time=2024-10-08 00:05:00+08:00, candle_dir=data/usdt_futures_5m
🌀 type=spot, num_symbols=382, first=1000SATSUSDT, last=ZRXUSDT
spot round 1, server_time=2024-10-08 00:06:39.880000+08:00, used_weight=847, symbols=1000SATSUSDT -- CHRUSDT
🌀 type=usdt_futures, num_symbols=297, first=1000BONKUSDT, last=ZRXUSDT
usdt_futures round 1, server_time=2024-10-08 00:06:40.106000+08:00, used_weight=647, symbols=1000BONKUSDT -- CTSIUSDT
spot round 2, server_time=2024-10-08 00:06:41.109000+08:00, used_weight=1008, symbols=1000SATSUSDT -- CHRUSDT

......

usdt_futures round 23, server_time=2024-10-08 00:07:26.202000+08:00, used_weight=2094, symbols=LTCUSDT -- STMXUSDT
spot round 25, server_time=2024-10-08 00:07:27.654000+08:00, used_weight=2106, symbols=SYSUSDT -- ZRXUSDT
✅ spot initialized, server_time=2024-10-08 00:07:29.138000+08:00, used_weight=2227

usdt_futures round 24, server_time=2024-10-08 00:08:00.040000+08:00, used_weight=1, symbols=LUNA2USDT -- STORJUSDT
🔔 usdt_futures REIUSDT candle not enough, num=2941

......

usdt_futures round 36, server_time=2024-10-08 00:08:10.603000+08:00, used_weight=1535, symbols=SUNUSDT -- ZRXUSDT
✅ usdt_futures initialized, server_time=2024-10-08 00:08:11.425000+08:00, used_weight=1638

==================== Finished in 1.53mins 2024-10-08 00:08:11 ====================
```

整个过程耗时 1.53 分钟，其中由于 U 本位合约 API 已达到权重使用上限，因此该方案已达到下载速度的理论最大值。

数据保存的目录结构如下：

```
.
└─- data
    ├── spot_5m
    │   ├── 1000SATSUSDT.pkl.zst
    │   ├── 1INCHUSDT.pkl.zst
    │   ├── 1MBABYDOGEUSDT.pkl.zst
    │   └── ......
    └── usdt_futures_5m
        ├── 1000BONKUSDT.pkl.zst
        ├── 1000FLOKIUSDT.pkl.zst
        ├── 1000LUNCUSDT.pkl.zst
        └── ......
```

## 转换为全 offset 小时线，并与 Quantclass Data API 对比

### Resample 全 offset 成小时线

对于给定的交易对，使用以下代码将数据 resample 为全 offset 小时线，并将每个 offset 的小时线保存在对应的目录中。

```python
def resample_symbol(symbol, path, resample_dir):
    # 将原始周期和重采样周期转换为timedelta对象
    original_delta = convert_interval_to_timedelta(ORIGINAL_INTERVAL)
    resample_delta = convert_interval_to_timedelta(RESAMPLE_INTERVAL)

    # 计算重采样周期内包含的原始周期数量
    num_offsets = int(round(resample_delta / original_delta))

    # 从磁盘读取原始K线数据
    df: pd.DataFrame = pd.read_pickle(path)

    # 对于每个偏移量进行重采样
    for offset_idx in range(num_offsets):
        df1 = df.reset_index()

        # 计算当前偏移量对应的分钟数
        offset_min = offset_idx * 5

        # 根据指定的重采样周期对数据进行重采样，并指定如何聚合各列数据
        df_resample = df1.resample(RESAMPLE_INTERVAL, offset=f'{offset_min}min', on='candle_begin_time').agg({
            'candle_begin_time': 'first',
            'candle_end_time': 'last',
            'open': 'first',
            'high': 'max',
            'low': 'min',
            'close': 'last',
            'volume': 'sum',
            'quote_volume': 'sum',
            'trade_num': 'sum',
            'taker_buy_base_asset_volume': 'sum',
            'taker_buy_quote_asset_volume': 'sum',
        }).reset_index(drop=True)

        # 过滤掉 Resample 后，首尾时间长度不足的 K 线
        df_resample['duration'] = df_resample['candle_end_time'] - df_resample['candle_begin_time']
        df_resample = df_resample[df_resample['duration'] >= resample_delta]
        df_resample.set_index('candle_end_time', inplace=True)
        df_resample.drop(columns='duration', inplace=True)

        # 生成保存重采样数据的目录
        output_dir = os.path.join(resample_dir, f'{offset_min}m')
        os.makedirs(output_dir, exist_ok=True)
        # 将重采样后的数据保存为pkl格式文件
        df_resample.to_pickle(os.path.join(output_dir, f'{symbol}.pkl.zst'))
```

基于以下代码，对于指定的交易类型，遍历数据文件，调用 `resample_symbol` 函数对原始数据文件进行重采样处理，并将结果保存到新的目录中:

```python
def resample(trade_type):
    print('Resample', trade_type)
    # 生成原始数据和重采样数据的目录路径
    original_dir = os.path.join('data', f'{trade_type}_{ORIGINAL_INTERVAL}')
    resample_dir = os.path.join('data', f'{trade_type}_{RESAMPLE_INTERVAL}_resample')

    # 如果重采样目录已存在，先删除再重新创建
    if os.path.exists(resample_dir):
        shutil.rmtree(resample_dir)
    os.makedirs(resample_dir)

    # 获取所有原始数据文件的路径
    paths = sorted(glob(os.path.join(original_dir, '*.pkl.zst')))
    # 提取每个文件对应的交易对
    symbols = [os.path.basename(p).split('.')[0] for p in paths]

    # 遍历每个交易对和文件路径，进行重采样操作
    for symbol, path in zip(symbols, paths):
        resample_symbol(symbol, path, resample_dir)  # 对每个交易对的数据进行重采样
```

resample 后，生成的数据目录结构如下（省略了每个 offset 目录中的具体文件）：

```
.
└── data
    ├── spot_1h_resample
    │   ├── 0m
    │   ├── 10m
    │   ├── 15m
    │   ├── 20m
    │   ├── 25m
    │   ├── 30m
    │   ├── 35m
    │   ├── 40m
    │   ├── 45m
    │   ├── 50m
    │   ├── 55m
    │   └── 5m
    │       ├── BTCUSDT.pkl.zst
    │       ├── ETHUSDT.pkl.zst
    │       ├── BNBUSDT.pkl.zst
    │       └── ......
    └── usdt_futures_1h_resample
        ├── 0m
        ├── 10m
        ├── 15m
        ├── 20m
        ├── 25m
        ├── 30m
        ├── 35m
        ├── 40m
        ├── 45m
        ├── 50m
        ├── 55m
        └── 5m
            ├── 1000BONKUSDT.pkl.zst
            ├── 1000FLOKIUSDT.pkl.zst
            ├── 1000LUNCUSDT.pkl.zst
            └── ......
```

### 与 Quantclass Data API 对比

基于以下代码，对比我们 resample 出的数据，和 Quantclass 官方提供的带 offset 小时线

```python
async def compare(trade_type):
    # 定义 resample 数据的目录路径，基于交易类型和采样时间间隔
    resample_dir = os.path.join('data', f'{trade_type}_{RESAMPLE_INTERVAL}_resample')

    # 获取该目录下所有的文件夹名（每个文件夹表示一个 offset）
    offset_strs = os.listdir(resample_dir)

    # 定义需要比较的K线数据的字段
    columns = [
        'open', 'close', 'high', 'low', 'volume', 'quote_volume', 'trade_num', 'taker_buy_base_asset_volume',
        'taker_buy_quote_asset_volume'
    ]

    # 创建异步HTTP会话，设置超时时间为3秒
    async with create_aiohttp_session(timeout_sec=3) as session:
        # 初始化Quantclass数据API客户端
        quantclass_api = QuantclassDataApi(session, API_KEY, UUID)
        diffs = []  # 用于存储每个symbol的差异数据

        # 遍历每个 offset 对应的文件夹
        for offset_str in offset_strs:
            # 获取该 offset 的 Quantclass Data API 数据 url
            url_data = await quantclass_api.aioreq_data_api(offset_str)

            # 根据交易类型设置数据类型（现货'spot'或合约'swap'）
            type_ = 'spot' if trade_type == 'spot' else 'swap'

            # 获取 Quantclass 的 K 线数据
            df_quantclass = await quantclass_api.aioreq_candle_df(url_data[type_])

            # 获取该 offset 对应的所有重新采样数据文件路径
            df_paths = glob(os.path.join(resample_dir, offset_str, '*.pkl.zst'))

            # 遍历每个文件（每个文件对应一个 symbol）
            for df_path in df_paths:
                # 从文件路径中提取 symbol 名称
                symbol = os.path.basename(df_path).split('.')[0]

                # 读取 resample 后的小时线数据
                df_resample = pd.read_pickle(df_path)

                # 从 Quantclass 数据中筛选出对应 symbol 的 K 线数据
                df_symbol = df_quantclass[df_quantclass['symbol'] == symbol]

                # 合并 Quantclass 数据和重新采样数据，基于 K 线的起始时间（candle_begin_time）
                df = pd.merge(df_symbol, df_resample, how='left', on='candle_begin_time', suffixes=['_qtc', '_rsp'])

                # 计算每个字段的差异，记录最大差异值
                diff = {'symbol': symbol, 'offset': offset_str}
                for col in columns:
                    # 计算Quantclass与重新采样数据的差异绝对值，取最大值
                    diff[col] = (df[f'{col}_rsp'] / df[f'{col}_qtc'] - 1).abs().max()

                # 将差异数据添加到列表中
                diffs.append(diff)

        # 将所有差异数据转换为DataFrame
        df_diff = pd.DataFrame.from_records(diffs)
        # 打印每个字段的最大差异值
        print(f'\n{trade_type} errors:\n')
        print(df_diff[columns].max())
```

误差如下，误差可以忽略不计:

```
spot errors:

open                            0.000000e+00
close                           0.000000e+00
high                            0.000000e+00
low                             0.000000e+00
volume                          2.220446e-16
quote_volume                    2.220446e-16
trade_num                       0.000000e+00
taker_buy_base_asset_volume     2.220446e-16
taker_buy_quote_asset_volume    2.220446e-16
dtype: float64

usdt_futures errors:

open                            0.000000e+00
close                           0.000000e+00
high                            0.000000e+00
low                             0.000000e+00
volume                          2.220446e-16
quote_volume                    2.220446e-16
trade_num                       0.000000e+00
taker_buy_base_asset_volume     2.220446e-16
taker_buy_quote_asset_volume    2.220446e-16
dtype: float64
```

## 结论

本文从基于 `asyncio` 的 API 封装、币安历史 5 分钟 K 线数据下载，以及全 offset 小时线生成三个方面，逐步介绍了如何高效地下载并生成币安历史全 offset 小时线数据。

由于采用了基于 `asyncio` 的高并发技术，这种方法使得下载历史数据的效率达到了在权重限制下的最大下载速度。

通过与 Quantclass 提供的 Data API 进行对比，使用这种方法 resample 出的全 offset 小时线数据与 Quantclass 提供的最新数据之间的误差可以忽略不计。

基于本文提供的技术进一步开发，添加实时数据更新并接入 Quantclass 实盘框架，可以构建**基于 BMAC 的新一代数据中心**。


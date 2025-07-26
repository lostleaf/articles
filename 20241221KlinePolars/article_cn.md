# 突破 Pandas 性能极限：探究用 Polars 加速 BMAC 分钟偏移实盘计算

本文引入一种更为高效的 Python DataFrame —— Polars，并于 BMAC 实盘情境中阐述如何运用 Polars 加速分钟偏移的 K 线 Resample 计算。 

BMAC 的实盘更新流程共包含四个步骤：

1. 向 Binance REST API 请求 5m K线。
2. 把请求所得的 5m K线 Python 数组数据解析为 DataFrame。
3. 针对 DataFrame 执行 Resample 操作，将其重采样为小时线。
4. 将新生成的小时线写入硬盘。 

精简过的对应代码片段如下：

``` python
# 1. 基于 asyncio 批量请求所有交易对的基础周期(5m) K 线
base_klines_list = await asyncio.gather(*[
    async_retry_getter(fetcher.market_api.aioreq_klines, symbol=s, interval=base_interval, limit=once_update_candles)
    for s in symbols_trading
])

for symbol, klines in zip(symbols_trading, base_klines_list):
    # 2. 将请求到的基础周期(5m) K 线数据（python 2维数组）转化为 DataFrame
    df_base = pandas_parse_kline(klines, base_interval, run_time)

    # 3. 将基础周期(5m) K 线 DF, Resample 成带偏移的 Resample 周期(1h) K 线 DF
    df_resample = calc_resample(df_base, resample_interval, offset_str)

    # 4. 与已有 Resample 周期(1h) K 线 DF 合并，过滤/填充 K 线缺失，最后写入硬盘
    update_resample_candle(resample_mgr, symbol, df_resample, handler.kline_count_1h, run_time, resample_delta)
```

在文本后续部分，将演示如何运用 Polars 对上述步骤中的第 2 步和第 3 步进行加速。

为确保本次测试与实盘环境保持一致，本次测试选用腾讯云轻量级东京区 2C4G 服务器。 

并且 Polars 运行于单线程模式，具体是通过设置 `os.environ['POLARS_MAX_THREADS'] = '1'` 来实现的。 

## 对现货 BTCUSDT 单个交易对测试

针对 BTCUSDT 的单交易对测试，代码为 `test_btc_single.py`

### 从 API 请求单个交易对 K 线 

运用以下代码请求 BTCUSDT 这一单个交易对近期的 99 根 5m K线: 

```python
async def request_btc_kline():
    async with create_aiohttp_session(15) as session:
        binance_spot_api = BinanceMarketSpotApi(session)
        klines = await binance_spot_api.aioreq_klines(symbol='BTCUSDT', interval='5m', limit=99)
    return klines
```

请求到的K线数据格式如下: 

```python
[
    [1734741000000, '97240.00000000', '97406.71000000', '97216.32000000', ...], 
    [1734741300000, '97352.01000000', '97459.68000000', '97258.08000000', ...], 
    [1734741600000, '97458.78000000', '97482.97000000', '97288.00000000', ...],
    ...
]
```

### 将请求到的 K 线数据转化为 DataFrame

这部分代码实现在 `lib.py` 中

#### Pandas 实现

```python
def pandas_parse_kline(klines, time_interval):
    '''
    将 Binance API 返回的 K线 Python 数组数据, 解析为 Pandas DataFrame
    添加 K线结束时间 candle_begin_time
    '''
    columns = [
        'candle_begin_time', 'open', 'high', 'low', 'close', 'volume', 'close_time', 'quote_volume', 'trade_num',
        'taker_buy_base_asset_volume', 'taker_buy_quote_asset_volume', 'ignore'
    ]
    df = pd.DataFrame(klines, columns=columns)
    df.drop(columns=['ignore', 'close_time'], inplace=True)

    dtypes = {
        'candle_begin_time': int,
        'open': float,
        'high': float,
        'low': float,
        'close': float,
        'volume': float,
        'quote_volume': float,
        'trade_num': int,
        'taker_buy_base_asset_volume': float,
        'taker_buy_quote_asset_volume': float
    }

    df = df.astype(dtypes)

    df['candle_begin_time'] = pd.to_datetime(df['candle_begin_time'], unit='ms', utc=True)
    df.sort_values('candle_begin_time', ignore_index=True, inplace=True)
    df['candle_end_time'] = df['candle_begin_time'] + convert_interval_to_timedelta(time_interval)
    return df
```

打印经 Pandas 解析所得的D ataFrame，其格式如下(省略部份列)：

```
          candle_begin_time      open      high       low     close    volume  quote_volume  trade_num  ...
0 2024-12-21 00:40:00+00:00  97458.78  97482.97  97288.00  97288.01  40.62743  3.956740e+06      10487  ...
1 2024-12-21 00:45:00+00:00  97288.01  97467.92  97272.87  97434.45  64.52727  6.283287e+06      10642  ...
2 2024-12-21 00:50:00+00:00  97434.45  97611.99  97434.29  97560.01  41.88013  4.084548e+06       9174  ...
```

#### Polars 实现

Polars 实现中使用了 [Lazy API](https://docs.pola.rs/user-guide/concepts/lazy-api/) 来提升计算效率

```python
def polars_parse_kline(klines, time_interval):
    '''
    将 Binance API 返回的 K线 Python 数组数据, 解析为 Polars DataFrame
    添加 K线结束时间 candle_begin_time
    '''
    schema = {
        'candle_begin_time': pl.Int64,
        'open': pl.Float64,
        'high': pl.Float64,
        'low': pl.Float64,
        'close': pl.Float64,
        'volume': pl.Float64,
        'close_time': None,
        'quote_volume': pl.Float64,
        'trade_num': pl.Int64,
        'taker_buy_base_asset_volume': pl.Float64,
        'taker_buy_quote_asset_volume': pl.Float64,
        'ignore': None
    }

    lf = pl.LazyFrame(klines, schema=schema, orient='row')
    lf = lf.drop('close_time', 'ignore')

    lf = lf.with_columns(pl.col('candle_begin_time').cast(pl.Datetime('ms')).dt.replace_time_zone('UTC'))
    lf = lf.sort('candle_begin_time')

    delta = convert_interval_to_timedelta(time_interval)
    lf = lf.with_columns((pl.col('candle_begin_time') + delta).alias('candle_end_time'))
    df = lf.collect()
    return df
```

打印经 Polars 解析所得的 DataFrame，其格式如下(省略部份列)：

```
┌─────────────────────────┬──────────┬──────────┬──────────┬───┬───────────┬───┬─────────────────────────┐
│ candle_begin_time       ┆ open     ┆ high     ┆ low      ┆ … ┆ trade_num ┆ … ┆ candle_end_time         │
│ ---                     ┆ ---      ┆ ---      ┆ ---      ┆   ┆ ---       ┆   ┆ ---                     │
│ datetime[ms, UTC]       ┆ f64      ┆ f64      ┆ f64      ┆   ┆ i64       ┆   ┆ datetime[ms, UTC]       │
╞═════════════════════════╪══════════╪══════════╪══════════╪═══╪═══════════╪═══╪═════════════════════════╡
│ 2024-12-21 00:40:00 UTC ┆ 97458.78 ┆ 97482.97 ┆ 97288.0  ┆ … ┆ 10487     ┆ … ┆ 2024-12-21 00:45:00 UTC │
│ 2024-12-21 00:45:00 UTC ┆ 97288.01 ┆ 97467.92 ┆ 97272.87 ┆ … ┆ 10642     ┆ … ┆ 2024-12-21 00:50:00 UTC │
│ 2024-12-21 00:50:00 UTC ┆ 97434.45 ┆ 97611.99 ┆ 97434.29 ┆ … ┆ 9174      ┆ … ┆ 2024-12-21 00:55:00 UTC │
└─────────────────────────┴──────────┴──────────┴──────────┴───┴───────────┴───┴─────────────────────────┘
```

### 将 5m K线 DataFrame Resample 成小时线

这部分代码实现在 `lib.py` 中

#### Pandas 实现

```python

def pandas_calc_resample(df: pd.DataFrame, resample_interval: str, offset_str: str) -> pd.DataFrame:
    '''
    将 Pandas K线 DataFrame, Resample 成带 offset 的大周期 K 线
    例如将 5m K线, Resample 成 5分钟偏移的小时线
    '''

    # pandas 中 5m 代表 5 个月，需要转化成 5min
    if offset_str[-1] == 'm' and offset_str[:-1].isdigit():
        offset_str = offset_str.replace('m', 'min')

    resample_delta = convert_interval_to_timedelta(resample_interval)

    df1 = df.reset_index()

    # 根据指定的周期对数据进行 resample，并指定如何聚合各列数据
    df_resample = df1.resample(resample_interval, offset=offset_str, on='candle_begin_time').agg({
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

    # 过滤掉 Resample 后，时间长度不足的 K 线
    df_resample = df_resample[(df_resample['candle_end_time'] - df_resample['candle_begin_time']) >= resample_delta]
    return df_resample
```

#### Polars 实现

Polars 实现中使用了 [Lazy API](https://docs.pola.rs/user-guide/concepts/lazy-api/) 来提升计算效率

```python
def polars_calc_resample(df: pl.DataFrame, resample_interval: str, offset_str: str) -> pd.DataFrame:
    '''
    将 Polars K线 DataFrame, Resample 成带 offset 的大周期 K 线
    例如将 5m K线, Resample 成 5分钟偏移的小时线
    '''
    resample_interval = convert_interval_to_timedelta(resample_interval)
    offset = convert_interval_to_timedelta(offset_str)

    lf = df.lazy()
    lf = lf.group_by_dynamic('candle_begin_time', every=resample_interval, offset=offset).agg([
        pl.col("candle_begin_time").first().alias('candle_begin_time_real'),
        pl.col("candle_end_time").last(),
        pl.col("open").first(),
        pl.col("high").max(),
        pl.col("low").min(),
        pl.col("close").last(),
        pl.col("volume").sum(),
        pl.col("quote_volume").sum(),
        pl.col("trade_num").sum(),
        pl.col("taker_buy_base_asset_volume").sum(),
        pl.col("taker_buy_quote_asset_volume").sum(),
    ])

    # 过滤掉 Resample 后，时间长度不足的 K 线
    lf = lf.filter((pl.col('candle_end_time') - pl.col('candle_begin_time_real')) >= resample_interval)
    lf = lf.drop('candle_begin_time_real')
    return lf.collect()
```

### 测试代码与结果

在本次测试中，针对通过请求 Binance API 获取 BTCUSDT 5m K线数据、将 5m K线解析为 DataFrame 以及对解析获得的 5m DataFrame 执行 Resample 这三个操作分别进行了单独计时。

测试代码如下： 

```python
def test_btc_single():
    print('请求 BTCUSDT 99 根 5m K线')
    t_start = time.perf_counter()
    klines = asyncio.run(request_btc_kline())
    time_ms = (time.perf_counter() - t_start) * 1000
    print(f'请求成功, 耗时={time_ms:.2f}ms')

    print('解析 K线为 Pandas DataFrame')
    t_start = time.perf_counter()
    df_kline_pandas = pandas_parse_kline(klines, '5m')
    time_ms = (time.perf_counter() - t_start) * 1000
    print(f'解析为 Pandas DataFrame 成功, 耗时={time_ms:.2f}ms')

    print('解析 K线为 Polars DataFrame')
    t_start = time.perf_counter()
    df_kline_polars = polars_parse_kline(klines, '5m')
    time_ms = (time.perf_counter() - t_start) * 1000
    print(f'解析为 Polars DataFrame 成功, 耗时={time_ms:.2f}ms')

    print('使用 Pandas Resample K线')
    t_start = time.perf_counter()
    df_resample_pandas = pandas_calc_resample(df_kline_pandas, '1h', '0m')
    time_ms = (time.perf_counter() - t_start) * 1000
    print(f'Pandas Resample K线成功, 耗时 = {time_ms:.2f}ms')

    print('使用 Polars Resample K线')
    t_start = time.perf_counter()
    df_resample_polars = polars_calc_resample(df_kline_polars, '1h', '0m')
    time_ms = (time.perf_counter() - t_start) * 1000
    print(f'Polars Resample K线成功, 耗时 = {time_ms:.2f}ms')

    print('\nPandas Resample 结果')
    print(df_resample_pandas)

    print('\nPolars Resample 结果')
    print(df_resample_polars)
```

测试耗时结果如下:

```
请求 BTCUSDT 99 根 5m K线
请求成功, 耗时=27.37ms
解析 K线为 Pandas DataFrame
解析为 Pandas DataFrame 成功, 耗时=5.71ms
解析 K线为 Polars DataFrame
解析为 Polars DataFrame 成功, 耗时=3.95ms
使用 Pandas Resample K线
Pandas Resample K线成功, 耗时 = 7.97ms
使用 Polars Resample K线
Polars Resample K线成功, 耗时 = 1.35ms
```

可以发现，Polars 将 Python 数组解析为 DataFrame 时，速度比 Pandas 稍快。

而在将 5m K线 Resample 为小时线的计算过程中，Polars 的速度要比 Pandas 快数倍。 

Resample 结果如下（省略部份列），肉眼观察并无区别
```
Pandas Resample 结果
          candle_begin_time           candle_end_time      open      high       low     close      volume  quote_volume ...
1 2024-12-21 01:00:00+00:00 2024-12-21 02:00:00+00:00  97525.77  97668.68  97198.76  97357.23   939.89900  9.158986e+07 ...
2 2024-12-21 02:00:00+00:00 2024-12-21 03:00:00+00:00  97357.23  97610.54  97188.09  97396.00   507.91033  4.947036e+07 ...
3 2024-12-21 03:00:00+00:00 2024-12-21 04:00:00+00:00  97396.00  97487.47  97244.00  97487.46   489.58985  4.765622e+07 ...
4 2024-12-21 04:00:00+00:00 2024-12-21 05:00:00+00:00  97487.47  97643.00  97378.50  97588.00   424.52783  4.139499e+07 ...
5 2024-12-21 05:00:00+00:00 2024-12-21 06:00:00+00:00  97588.00  98692.00  97541.06  98691.98  1167.51919  1.144794e+08 ...
6 2024-12-21 06:00:00+00:00 2024-12-21 07:00:00+00:00  98691.98  98837.49  98318.60  98726.76  1768.19966  1.743267e+08 ...
7 2024-12-21 07:00:00+00:00 2024-12-21 08:00:00+00:00  98726.76  99540.61  98500.00  99040.00  2107.70340  2.087851e+08 ...
8 2024-12-21 08:00:00+00:00 2024-12-21 09:00:00+00:00  99040.00  99187.95  98438.01  98650.00  1024.17691  1.011345e+08 ...

Polars Resample 结果
shape: (8, 11)
┌─────────────────────────┬─────────────────────────┬──────────┬──────────┬───┬──────────────┬───────────┬───┐
│ candle_begin_time       ┆ candle_end_time         ┆ open     ┆ high     ┆ … ┆ quote_volume ┆ trade_num ┆ … │
│ ---                     ┆ ---                     ┆ ---      ┆ ---      ┆   ┆ ---          ┆ ---       ┆   │
│ datetime[ms, UTC]       ┆ datetime[ms, UTC]       ┆ f64      ┆ f64      ┆   ┆ f64          ┆ i64       ┆   │
╞═════════════════════════╪═════════════════════════╪══════════╪══════════╪═══╪══════════════╪═══════════╪═══╡
│ 2024-12-21 01:00:00 UTC ┆ 2024-12-21 02:00:00 UTC ┆ 97525.77 ┆ 97668.68 ┆ … ┆ 9.1590e7     ┆ 132882    ┆ … │
│ 2024-12-21 02:00:00 UTC ┆ 2024-12-21 03:00:00 UTC ┆ 97357.23 ┆ 97610.54 ┆ … ┆ 4.9470e7     ┆ 124669    ┆ … │
│ 2024-12-21 03:00:00 UTC ┆ 2024-12-21 04:00:00 UTC ┆ 97396.0  ┆ 97487.47 ┆ … ┆ 4.7656e7     ┆ 98621     ┆ … │
│ 2024-12-21 04:00:00 UTC ┆ 2024-12-21 05:00:00 UTC ┆ 97487.47 ┆ 97643.0  ┆ … ┆ 4.1395e7     ┆ 74956     ┆ … │
│ 2024-12-21 05:00:00 UTC ┆ 2024-12-21 06:00:00 UTC ┆ 97588.0  ┆ 98692.0  ┆ … ┆ 1.1448e8     ┆ 141134    ┆ … │
│ 2024-12-21 06:00:00 UTC ┆ 2024-12-21 07:00:00 UTC ┆ 98691.98 ┆ 98837.49 ┆ … ┆ 1.7433e8     ┆ 185812    ┆ … │
│ 2024-12-21 07:00:00 UTC ┆ 2024-12-21 08:00:00 UTC ┆ 98726.76 ┆ 99540.61 ┆ … ┆ 2.0879e8     ┆ 179234    ┆ … │
│ 2024-12-21 08:00:00 UTC ┆ 2024-12-21 09:00:00 UTC ┆ 99040.0  ┆ 99187.95 ┆ … ┆ 1.0113e8     ┆ 151068    ┆ … │
└─────────────────────────┴─────────────────────────┴──────────┴──────────┴───┴──────────────┴───────────┴───┘
```

## 对全市场现货交易对测试

### 从 API 请求全市场现货 USDT 交易对 K 线 

测试代码仅对处于正在交易（‘TRADING’状态）的USDT交易对进行了过滤，虽与实盘存在些许偏差，但该偏差可予以忽略。 

``` python
async def request_spot_klines():
    async with create_aiohttp_session(15) as session:
        binance_spot_api = BinanceMarketSpotApi(session)

        print('开始请求 Binance 现货交易对')
        t_start = time.perf_counter()
        exginfo = await binance_spot_api.aioreq_exchange_info()
        sinfos = exginfo['symbols']
        symbols = sorted(x['symbol'] for x in sinfos if x['quoteAsset'] == 'USDT' and x['status'] == 'TRADING')
        time_ms = (time.perf_counter() - t_start) * 1000
        print(f'请求交易对成功, 交易对数量={len(symbols)}, 耗时={time_ms:.2f}ms')

        print('开始请求所有现货 99 根 5m K线')
        t_start = time.perf_counter()
        klines_list = await asyncio.gather(
            *[binance_spot_api.aioreq_klines(symbol=s, interval='5m', limit=99) for s in symbols])
        time_ms = (time.perf_counter() - t_start) * 1000
        print(f'请求所有现货 99 根 5m K线成功, 耗时={time_ms:.2f}ms')
    return symbols, klines_list
```

### 测试代码与结果

```python
def test_spot_all():
    symbols, klines_list = asyncio.run(request_spot_klines())

    print('使用 Pandas 解析并 Resample 全市场 K线中')
    pandas_dfs = dict()
    t_start = time.perf_counter()
    for symbol, klines in zip(symbols, klines_list):
        df_kline_pandas = pandas_parse_kline(klines, '5m')
        df_resample_pandas = pandas_calc_resample(df_kline_pandas, '1h', '0m')
        pandas_dfs[symbol] = df_resample_pandas
    time_ms = (time.perf_counter() - t_start) * 1000
    print(f'Pandas Resample 全市场 K线成功, 耗时={time_ms:.2f}ms')

    print('使用 Pandas 解析并 Resample 全市场 K线中')
    polars_dfs = dict()
    t_start = time.perf_counter()
    for symbol, klines in zip(symbols, klines_list):
        df_kline_polars = polars_parse_kline(klines, '5m')
        df_resample_polars = polars_calc_resample(df_kline_polars, '1h', '0m')
        polars_dfs[symbol] = df_resample_polars
    time_ms = (time.perf_counter() - t_start) * 1000
    print(f'Polars Resample 全市场 K线成功, 耗时={time_ms:.2f}ms')

    print('Pandas Resample ETHUSDT 结果')
    print(pandas_dfs['ETHUSDT'])

    print('Polars Resample ETHUSDT 结果')
    print(polars_dfs['ETHUSDT'])

    print('Pandas Resample SOLUSDT 结果')
    print(pandas_dfs['SOLUSDT'])

    print('Polars Resample SOLUSDT 结果')
    print(polars_dfs['SOLUSDT'])
```

测试耗时结果如下:

```
开始请求 Binance 现货交易对
请求交易对成功, 交易对数量=390, 耗时=256.70ms
开始请求所有现货 99 根 5m K线
请求所有现货 99 根 5m K线成功, 耗时=344.38ms
使用 Pandas 解析并 Resample 全市场 K线中
Pandas Resample 全市场 K线成功, 耗时=3426.75ms
使用 Pandas 解析并 Resample 全市场 K线中
Polars Resample 全市场 K线成功, 耗时=437.46ms
```

在解析 5m K线并 Resample 为小时线的计算过程中，Polars 的速度仍然要比 Pandas 快数倍。 

Resample 结果如下（省略部份列）

```
Pandas Resample ETHUSDT 结果
          candle_begin_time           candle_end_time     open     high      low    close      volume  quote_volume  ...
1 2024-12-21 02:00:00+00:00 2024-12-21 03:00:00+00:00  3473.03  3489.96  3460.00  3474.72  12461.7329  4.333093e+07  ...
2 2024-12-21 03:00:00+00:00 2024-12-21 04:00:00+00:00  3474.72  3478.74  3455.00  3467.64   7912.0257  2.741939e+07  ...
3 2024-12-21 04:00:00+00:00 2024-12-21 05:00:00+00:00  3467.64  3488.82  3460.80  3483.00  12238.1553  4.253991e+07  ...
4 2024-12-21 05:00:00+00:00 2024-12-21 06:00:00+00:00  3483.01  3532.99  3476.95  3532.59  22030.2716  7.717409e+07  ...
5 2024-12-21 06:00:00+00:00 2024-12-21 07:00:00+00:00  3532.59  3534.30  3504.83  3531.95  28776.8992  1.013819e+08  ...
6 2024-12-21 07:00:00+00:00 2024-12-21 08:00:00+00:00  3531.95  3555.18  3523.01  3523.46  22470.5953  7.954860e+07  ...
7 2024-12-21 08:00:00+00:00 2024-12-21 09:00:00+00:00  3523.45  3527.49  3479.33  3484.50  22373.9839  7.828242e+07  ...
Polars Resample ETHUSDT 结果
shape: (7, 11)
┌─────────────────────────┬─────────────────────────┬─────────┬─────────┬───┬──────────────┬───────────┬───┐
│ candle_begin_time       ┆ candle_end_time         ┆ open    ┆ high    ┆ … ┆ quote_volume ┆ trade_num ┆ … │
│ ---                     ┆ ---                     ┆ ---     ┆ ---     ┆   ┆ ---          ┆ ---       ┆   │
│ datetime[ms, UTC]       ┆ datetime[ms, UTC]       ┆ f64     ┆ f64     ┆   ┆ f64          ┆ i64       ┆   │
╞═════════════════════════╪═════════════════════════╪═════════╪═════════╪═══╪══════════════╪═══════════╪═══╡
│ 2024-12-21 02:00:00 UTC ┆ 2024-12-21 03:00:00 UTC ┆ 3473.03 ┆ 3489.96 ┆ … ┆ 4.3331e7     ┆ 66424     ┆ … │
│ 2024-12-21 03:00:00 UTC ┆ 2024-12-21 04:00:00 UTC ┆ 3474.72 ┆ 3478.74 ┆ … ┆ 2.7419e7     ┆ 61296     ┆ … │
│ 2024-12-21 04:00:00 UTC ┆ 2024-12-21 05:00:00 UTC ┆ 3467.64 ┆ 3488.82 ┆ … ┆ 4.2540e7     ┆ 64701     ┆ … │
│ 2024-12-21 05:00:00 UTC ┆ 2024-12-21 06:00:00 UTC ┆ 3483.01 ┆ 3532.99 ┆ … ┆ 7.7174e7     ┆ 98238     ┆ … │
│ 2024-12-21 06:00:00 UTC ┆ 2024-12-21 07:00:00 UTC ┆ 3532.59 ┆ 3534.3  ┆ … ┆ 1.0138e8     ┆ 118567    ┆ … │
│ 2024-12-21 07:00:00 UTC ┆ 2024-12-21 08:00:00 UTC ┆ 3531.95 ┆ 3555.18 ┆ … ┆ 7.9549e7     ┆ 97479     ┆ … │
│ 2024-12-21 08:00:00 UTC ┆ 2024-12-21 09:00:00 UTC ┆ 3523.45 ┆ 3527.49 ┆ … ┆ 7.8282e7     ┆ 97037     ┆ … │
└─────────────────────────┴─────────────────────────┴─────────┴─────────┴───┴──────────────┴───────────┴───┘
Pandas Resample SOLUSDT 结果
          candle_begin_time           candle_end_time    open    high     low   close      volume  quote_volume  ...
1 2024-12-21 02:00:00+00:00 2024-12-21 03:00:00+00:00  195.36  195.88  194.03  194.53  108543.739  2.116400e+07  ...
2 2024-12-21 03:00:00+00:00 2024-12-21 04:00:00+00:00  194.53  194.97  193.50  194.28  111383.126  2.161996e+07  ...
3 2024-12-21 04:00:00+00:00 2024-12-21 05:00:00+00:00  194.29  196.45  194.20  196.38  101991.613  1.991584e+07  ...
4 2024-12-21 05:00:00+00:00 2024-12-21 06:00:00+00:00  196.38  199.24  196.21  199.18  161910.090  3.198895e+07  ...
5 2024-12-21 06:00:00+00:00 2024-12-21 07:00:00+00:00  199.19  199.98  197.29  199.38  225968.785  4.491152e+07  ...
6 2024-12-21 07:00:00+00:00 2024-12-21 08:00:00+00:00  199.39  201.98  198.76  198.81  209512.388  4.191950e+07  ...
7 2024-12-21 08:00:00+00:00 2024-12-21 09:00:00+00:00  198.80  198.97  196.49  196.54  197613.520  3.904066e+07  ...
Polars Resample SOLUSDT 结果
shape: (7, 11)
┌─────────────────────────┬─────────────────────────┬────────┬────────┬───┬──────────────┬───────────┬───┐
│ candle_begin_time       ┆ candle_end_time         ┆ open   ┆ high   ┆ … ┆ quote_volume ┆ trade_num ┆ … │
│ ---                     ┆ ---                     ┆ ---    ┆ ---    ┆   ┆ ---          ┆ ---       ┆   │
│ datetime[ms, UTC]       ┆ datetime[ms, UTC]       ┆ f64    ┆ f64    ┆   ┆ f64          ┆ i64       ┆   │
╞═════════════════════════╪═════════════════════════╪════════╪════════╪═══╪══════════════╪═══════════╪═══╡
│ 2024-12-21 02:00:00 UTC ┆ 2024-12-21 03:00:00 UTC ┆ 195.36 ┆ 195.88 ┆ … ┆ 2.1164e7     ┆ 40874     ┆ … │
│ 2024-12-21 03:00:00 UTC ┆ 2024-12-21 04:00:00 UTC ┆ 194.53 ┆ 194.97 ┆ … ┆ 2.1620e7     ┆ 43669     ┆ … │
│ 2024-12-21 04:00:00 UTC ┆ 2024-12-21 05:00:00 UTC ┆ 194.29 ┆ 196.45 ┆ … ┆ 1.9916e7     ┆ 49685     ┆ … │
│ 2024-12-21 05:00:00 UTC ┆ 2024-12-21 06:00:00 UTC ┆ 196.38 ┆ 199.24 ┆ … ┆ 3.1989e7     ┆ 59230     ┆ … │
│ 2024-12-21 06:00:00 UTC ┆ 2024-12-21 07:00:00 UTC ┆ 199.19 ┆ 199.98 ┆ … ┆ 4.4912e7     ┆ 62396     ┆ … │
│ 2024-12-21 07:00:00 UTC ┆ 2024-12-21 08:00:00 UTC ┆ 199.39 ┆ 201.98 ┆ … ┆ 4.1919e7     ┆ 63015     ┆ … │
│ 2024-12-21 08:00:00 UTC ┆ 2024-12-21 09:00:00 UTC ┆ 198.8  ┆ 198.97 ┆ … ┆ 3.9041e7     ┆ 56954     ┆ … │
└─────────────────────────┴─────────────────────────┴────────┴────────┴───┴──────────────┴───────────┴───┘
```

## 总结

本文通过引入相较于传统Pandas更为高效的Python DataFrame实现方案——Polars，在模拟BMAC实盘K线获取以及重采样（Resample）计算的场景中，实现了计算效率的大幅提升。 

### API 请求耗时对比

|    | 全市场现货交易对 | BTCUSDT 单一交易对 |
|--------|--------------|-------------------|
| 数量 | 390          | 1                 |
| 耗时 | 344\.38ms    | 27\.37ms          |

因采用了 asyncio 进行并发请求，所以请求 390 个现货交易对 K线时，耗时仅为数百毫秒，并不会构成性能瓶颈。

### K 线解析及 Resample 耗时对比

|场景| 耗时 | Pandas | Polars |
|---| ---- | ---- | ---- |
| BTCUSDT 单一交易对| 解析为 DataFrame | 5\.71ms | 3\.95ms |
| BTCUSDT 单一交易对| Resample K线 | 7\.97ms | 1\.35ms |
| 全市场 390 个现货交易对| 解析 DataFrame + Resample | 3426\.75ms | 437\.46ms |


可以看出，无论是针对单一交易对，还是全市场将近 400 个交易对，Polars 相较于 Pandas 均展现出了绝对优势。
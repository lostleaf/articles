# 币安代币拆分合并与重上架的研究

众所周知，保温杯是一个多垃圾币空主流（山寨）币的策略，本文将对垃圾币的拆分合并以及下架后重上架机制作出研究，并对数据处理给出建议。

理论上可以帮助防止因相关机制导致的保温杯策略回测错误。

## 从 VIDT 说起

首先我们看一下 VIDT 这个山寨币现货行情

在 20221031 - 20221109 这段时间有如下行情，从图中看，VIDT 的价格发生了大幅度的跳空

![](1.png)

那么在这中间发生了什么事情呢？根据[币安官方公告](https://www.binance.com/en/support/announcement/binance-will-support-the-vidt-datalink-vidt-token-swap-redenomination-rebranding-plan-to-vidt-dao-vidt-ef40b40af1944bc9b399ad9c52861a04)

- 2022-10-31 03:00 (UTC), 币安将下架 VIDT/USDT, VIDT/BUSD, VIDT/BTC 这几个交易对
- 2022-11-09 08:00 (UTC), 币安将重新上架 for VIDT/USDT, VIDT/BUSD, VIDT/BTC 交易对
- 重新上架后 `1旧代币 = 10新代币`

也即是说，在这段时间里，VIDT 完成了一次**代币拆分**

## 检查 VIDT 数据

让我们再来看一下数据，基于以下代码，载入 BHDS 根据 quantclass 官网币对分类生成的 parquet 数据

``` python
def read(type_, symbol):
    if type_ == 'spot':
        df = pd.read_parquet(os.path.join(SPOT_DIR, f'{symbol}.pqt'))
    elif type_ == 'usdt_futures':
        df = pd.read_parquet(os.path.join(USDT_FUTURES_DIR, f'{symbol}.pqt'))

    return df[['candle_begin_time', 'open', 'high', 'low', 'close', 'volume']]


read('spot', 'VIDTUSDT').loc['2022-10-31 02:00:00+00:00':].head()
```

输出结果如下 

![](2.png)

可以看到，从 20221031 - 20221109，大约 9 天的行情数据是缺失的，其价格也从之前的 0.4731 跌到了 0.044

基于以下代码，将所有行情缺失都找出来

``` python
def check(df):    
    df['time_diff'] = df['candle_begin_time'].diff()

    gaps = []
    idxes = df[df['time_diff'] > df['time_diff'].min()].index
    for idx in idxes:
        tail = df.loc[:idx].tail(2)
        begin_time_before = tail.iloc[0]['candle_begin_time']
        begin_time_after = tail.iloc[1]['candle_begin_time']
        time_gap = begin_time_after - begin_time_before
        price_change = tail.iloc[1]['open'] / tail.iloc[0]['close'] - 1
        gaps.append((begin_time_after, time_gap, price_change))

    return pd.DataFrame(gaps, columns=['relist_time', 'time_gap', 'price_change'])

def check_gaps(type_, symbol):
    df = read(type_, symbol)
    df_result = check(df)
    df_result['type'] = type_
    df_result['symbol'] = symbol
    return df_result

check_gaps('spot', 'VIDTUSDT')
```

输出结果为 

| relist_time               | time_gap        |   price_change | type   | symbol   |
|:--------------------------|:----------------|---------------:|:-------|:---------|
| 2021-09-29 09:00:00+00:00 | 0 days 03:00:00 |    0.000516929 | spot   | VIDTUSDT |
| 2022-11-09 08:00:00+00:00 | 9 days 06:00:00 |   -0.906996    | spot   | VIDTUSDT |
| 2023-03-24 14:00:00+00:00 | 0 days 03:00:00 |   -0.00166482  | spot   | VIDTUSDT |

可以看到，从 VIDT 最初上架至今，一共出现了 3 次数据缺失

其中最久的一次为 9 天，价格下跌了 90%；其他两次缺失仅为 3 小时，价格变化可忽略不计

对与这种情况，数据处理建议如下：

- 对于长达 9 天的代币拆分，将拆分重上架前后两段行情，拆成两个独立的 symbol，计算因子及回测时单独处理
- 对于两次两次 3 小时的缺失，我们可以看作技术失误导致的数据缺失，直接使用一字线填充即刻，即开高低收价格使用前收盘价填充，交易量使用0填充

## 对全量数据的研究

有了基于 VIDT 的研究，我们推广到全量数据进行数据检查

### 被研究的 symbol 范围

首先，研究范围与保温杯大致上一直，核心代码为

``` python
STABLECOINS = {'BKRWUSDT', 'USDCUSDT', 'USDPUSDT', 'TUSDUSDT', 'BUSDUSDT', 'FDUSDUSDT', 'DAIUSDT', 'EURUSDT', 'GBPUSDT',
               'USBPUSDT', 'SUSDUSDT', 'PAXGUSDT', 'AEURUSDT'}

BLACKLIST = {'NBTUSDT'}

def filter_symbols(symbols):
    # 过滤杠杆代币，但要注意 JUP 不能过滤
    lev_symbols = {x for x in symbols if x.endswith(('UPUSDT', 'DOWNUSDT', 'BEARUSDT', 'BULLUSDT')) and x != 'JUPUSDT'}

    # 过滤非 USDT 交易对
    not_usdt_symbols = {x for x in symbols if not x.endswith('USDT')}

    # 加上稳定币和黑名单
    excludes = set.union(not_usdt_symbols, lev_symbols, STABLECOINS, BLACKLIST).intersection(symbols)

    symbols_filtered = sorted(set(symbols) - excludes)
    return symbols_filtered
```

其中，相比保温杯默认设置，增加了 AEUR 这个稳定币，而 NBT 这个币，由于其数据质量实在太差，并且已经下架很长时间，也拉黑

### 现货

基于以下代码，生成所有现货的空缺，并以经验值 2 天作为划分，输出其中数据缺失大于两天的

``` python
symbols = get_filtered_symbols('spot')

dfs = [check_gaps('spot',  symbol) for symbol in symbols]
dfs = [df for df in dfs if len(df)]
df_gap = pd.concat(dfs, ignore_index=True)

threshold = pd.Timedelta(days=2)

df_gap_short = df_gap[df_gap['time_gap'] <  threshold]
df_gap_long = df_gap[df_gap['time_gap'] >= threshold].reset_index(drop=True)

display(df_gap_long)
```

输出如下

| relist_time               | time_gap          |   price_change | type   | symbol    |
|:--------------------------|:------------------|---------------:|:-------|:----------|
| 2023-02-22 08:00:00+00:00 | 6 days 06:00:00   |     -0.99      | spot   | BNXUSDT   |
| 2021-03-19 07:00:00+00:00 | 4 days 01:00:00   |     -0.900002  | spot   | BTCSTUSDT |
| 2021-01-23 02:00:00+00:00 | 4 days 01:00:00   |    999         | spot   | COCOSUSDT |
| 2023-05-12 08:00:00+00:00 | 154 days 06:00:00 |     -0.0899796 | spot   | CVCUSDT   |
| 2021-04-02 04:00:00+00:00 | 4 days 01:00:00   |     99.0076    | spot   | DREPUSDT  |
| 2023-09-22 08:00:00+00:00 | 311 days 04:00:00 |     -0.31909   | spot   | FTTUSDT   |
| 2023-03-10 08:00:00+00:00 | 28 days 06:00:00  |      0.840514  | spot   | KEYUSDT   |
| 2022-05-31 06:00:00+00:00 | 18 days 06:00:00  |  19999         | spot   | LUNAUSDT  |
| 2023-07-21 08:00:00+00:00 | 4 days 06:00:00   |     -0.999     | spot   | QUICKUSDT |
| 2024-03-28 08:00:00+00:00 | 8 days 06:00:00   |     -0.9       | spot   | STRAXUSDT |
| 2021-06-18 04:00:00+00:00 | 4 days 01:00:00   |     -0.999     | spot   | SUNUSDT   |
| 2018-10-19 09:00:00+00:00 | 88 days 06:00:00  |     -0.999945  | spot   | VENUSDT   |
| 2022-11-09 08:00:00+00:00 | 9 days 06:00:00   |     -0.906996  | spot   | VIDTUSDT  |

以上这些币均存在长时间的数据缺失，并且前后价格通常相差10倍到万倍，可以认为均发生了代币拆分或合并，建议将前后拆分成两个不同 symbol

而对于较短的数据缺失，则有

``` python
df_gap_short.describe([.01, .1, .9, .99])
```

输出为

|       | time_gap                  |   price_change |
|:------|:--------------------------|---------------:|
| count | 7754                      | 7754           |
| mean  | 0 days 02:55:12.612844983 |   -2.27507e-05 |
| std   | 0 days 01:51:53.365951751 |    0.00527826  |
| min   | 0 days 02:00:00           |   -0.0414446   |
| 1%    | 0 days 02:00:00           |   -0.0161004   |
| 10%   | 0 days 02:00:00           |   -0.00461637  |
| 50%   | 0 days 02:00:00           |    0           |
| 90%   | 0 days 05:00:00           |    0.00452761  |
| 99%   | 0 days 11:00:00           |    0.0159473   |
| max   | 1 days 10:00:00           |    0.0586466   |

虽然较短的数据缺失发生了 7700 多次，但 90% 的缺失都在 5 小时以下，最长也不超过 1.5 天

并且前后的价格变化率通常不超过 2%，因此这些缺失可以全部用一字线填充

### 合约

合约的数据则相对比较高质量，主要的问题仍然是著名的重上架三兄弟 ICP, BNX, TLM

其他都可以忽略，用一字线填充即可

| relist_time               | time_gap          |   price_change | type         | symbol   |
|:--------------------------|:------------------|---------------:|:-------------|:---------|
| 2019-09-08 19:00:00+00:00 | 0 days 02:00:00   |     0.034477   | usdt_futures | BTCUSDT  |
| 2019-09-09 02:00:00+00:00 | 0 days 03:00:00   |    -0.00721831 | usdt_futures | BTCUSDT  |
| 2019-11-27 10:00:00+00:00 | 0 days 02:00:00   |     0.0691729  | usdt_futures | ETHUSDT  |
| 2021-03-02 02:00:00+00:00 | 0 days 02:00:00   |    -0.00228447 | usdt_futures | BLZUSDT  |
| 2021-03-02 02:00:00+00:00 | 0 days 02:00:00   |     0.00188201 | usdt_futures | CTKUSDT  |
| 2021-03-02 02:00:00+00:00 | 0 days 02:00:00   |    -0.00205245 | usdt_futures | DODOUSDT |
| 2021-03-02 02:00:00+00:00 | 0 days 02:00:00   |     0          | usdt_futures | LITUSDT  |
| 2022-09-27 02:00:00+00:00 | 108 days 17:00:00 |     0.0263975  | usdt_futures | ICPUSDT  |
| 2023-02-22 14:00:00+00:00 | 11 days 15:00:00  |    -0.987751   | usdt_futures | BNXUSDT  |
| 2023-03-30 12:00:00+00:00 | 294 days 04:00:00 |    -0.402594   | usdt_futures | TLMUSDT  |

## 不对拆分合并做处理可能导致的问题

如果不对拆分合并做处理，可能导致因子计算出现严重的错误

以保温杯默认策略计算的 LUNAUSDT PctChange 因子为例

| candle_begin_time   |   PctChange_3 |   PctChange_7 |   PctChange_10 |   PctChange_14 |
|:--------------------|--------------:|--------------:|---------------:|---------------:|
| 2022-05-26 00:00:00 |       0       |             0 |              0 |      -0.999954 |
| 2022-05-27 00:00:00 |       0       |             0 |              0 |      -0.84375  |
| 2022-05-28 00:00:00 |       0       |             0 |              0 |       0        |
| 2022-05-29 00:00:00 |       0       |             0 |              0 |       0        |
| 2022-05-30 00:00:00 |       0       |             0 |              0 |       0        |
| 2022-05-31 00:00:00 |       0       |             0 |              0 |       0        |
| 2022-06-01 00:00:00 |  177399       |        177399 |         177399 |  177399        |
| 2022-06-02 00:00:00 |  130589       |        130589 |         130589 |  130589        |
| 2022-06-03 00:00:00 |  142049       |        142049 |         142049 |  142049        |
| 2022-06-04 00:00:00 |      -0.26841 |        129783 |         129783 |  129783        |

可以看到，由于 LUNA 暴雷重上架，最后 4 行 PctChange 因子的计算出现了严重的错误

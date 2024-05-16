# Symbol 标准化：一种更优的保温杯现货与合约匹配方式

在量化交易策略中，尤其是保温杯策略，一个关键步骤是确保现货和合约的 Symbol 能够正确匹配。这样才能确定可做空的 Symbol 以及相应的空头仓位。

现实情况是，币安的部分 Symbol 命名存在不规则性。例如，现货的 `PEPEUSDT` 对应合约的 `1000PEPEUSDT`，而现货的 `DODOUSDT` 对应合约的 `DODOXUSDT`。

同时，[BHDS 框架](https://bbs.quantclass.cn/thread/40387) 采用了[币安代币拆分合并与重上架的研究
](https://bbs.quantclass.cn/thread/41347) 帖子中的策略，对于那些因拆分合并而重上架历史的代币，进行了 Symbol 拆分，从而增加了命名的不规则性。

至于官方的保温杯框架，则过于简单地将现货 `LUNAUSDT` 与合约 `LUNA2USDT` 直接匹配，并没有考虑到 `LUNAUSDT` 和 `DODOUSDT` 在历史上也曾是永续合约的 Symbol。这种匹配方式显得粗糙。

因此，本文提出一种基于 Symbol 标准化的匹配方案：即先对 Symbol 进行标准化处理，例如将 `PEPEUSDT` 和 `1000PEPEUSDT` 都标准化为 `PEPEUSDT`，然后再基于标准化后的 Symbol 匹配现货和合约，最后根据 K 线的时间戳确定不同历史时期的现货和合约对应关系。

让我们以 LUNA 为例，来探讨这一过程。

## 示例：LUNA

币安目前共有两个与 LUNA 相关的现货交易 Symbol：`LUNAUSDT` 和 `LUNCUSDT`。其中 `LUNAUSDT` 又被 BHDS 框架拆分为崩盘前的老 LUNA `LUNA1USDT` 和崩盘后的新 LUNA `LUNAUSDT`。

崩盘后的老 LUNA 则被改名为 LUNC。

同时，币安共有三个与 LUNA 相关的合约交易 Symbol：`LUNAUSDT`、`LUNA2USDT` 和 `1000LUNCUSDT`。

根据交易数据，我们可以将以上六个 Symbol 的关系整理成下表：

|                   | 现货       | 永续合约       |
| :---------------- | :---------: | :-------------: |
| 老 LUNA（崩盘前）   | LUNA1USDT | LUNAUSDT       |
| 新 LUNA（崩盘后）   | LUNAUSDT  | LUNA2USDT      |
| LUNC（崩盘后）    | LUNCUSDT  | 1000LUNCUSDT   |

基于以下 Python 代码，我们总结以上六个 Symbol 的 1小时 K 线数据的关键特征：

``` python
def summary(type_, symbol):
    if type_ == 'spot':
        dir_path = spot_dir
    elif type_ == 'usdt_futures':
        dir_path = usdt_futures_dir
    p = os.path.join(dir_path, f'{symbol}.pqt')
    df = pd.read_parquet(p)
    first_begin = df['candle_begin_time'].iloc[0]
    last_begin = df['candle_begin_time'].iloc[-1]
    last_price = df['close'].iloc[-1]
    return {'first_begin': first_begin, 'last_begin': last_begin, 'last_price': last_price}

spot_data = {
    'LUNAUSDT': summary('spot', 'LUNAUSDT'),
    'LUNA1USDT': summary('spot', 'LUNA1USDT'),
    'LUNCUSDT': summary('spot', 'LUNCUSDT'),
}

usdt_futures_data = {
    'LUNAUSDT': summary('usdt_futures', 'LUNAUSDT'),
    'LUNA2USDT': summary('usdt_futures', 'LUNA2USDT'),
    '1000LUNCUSDT': summary('usdt_futures', '1000LUNCUSDT'),
}
print(pd.DataFrame.from_dict(spot_data, orient='index').sort_values('first_begin').to_markdown())
print(pd.DataFrame.from_dict(usdt_futures_data, orient='index').sort_values('first_begin').to_markdown())
```

数据输出如下：

现货
|           | first_begin               | last_begin                |   last_price |
|:----------|:--------------------------|:--------------------------|-------------:|
| LUNA1USDT | 2020-08-21 10:00:00+00:00 | 2022-05-13 00:00:00+00:00 |   0.00005      |
| LUNAUSDT  | 2022-05-31 06:00:00+00:00 | 2024-05-15 23:00:00+00:00 |   0.5885     |
| LUNCUSDT  | 2022-09-09 08:00:00+00:00 | 2024-05-14 23:00:00+00:00 |   0.00010208 |

永续合约
|              | first_begin               | last_begin                |   last_price |
|:-------------|:--------------------------|:--------------------------|-------------:|
| LUNAUSDT     | 2021-01-28 07:00:00+00:00 | 2022-05-12 15:00:00+00:00 |      0.008   |
| 1000LUNCUSDT | 2022-09-09 13:00:00+00:00 | 2024-05-14 23:00:00+00:00 |      0.10201 |
| LUNA2USDT    | 2022-09-10 03:00:00+00:00 | 2024-05-14 23:00:00+00:00 |      0.5546  |

通过以上分析，不难看出，我们只需将现货 Symbol `LUNA1USDT` 和 `LUNAUSDT` 以及合约 Symbol `LUNAUSDT` 和 `LUNA2USDT` 标准化为 `LUNAUSDT`；将现货 Symbol `LUNCUSDT` 与合约 Symbol `1000LUNCUSDT` 标准化为 `LUNCUSDT`。然后，根据 K 线起始和终止时间，我们便可以准确地匹配不同时间段的现货与合约。

## 标准化规则

从以上的例子中，我们可以总结出两条标准化规则：

1. **1000系列**：对于那些价格极低的代币，其合约可能会添加 "1000" 前缀，代表价格乘以 1000。例如，`1000LUNCUSDT` 应该标准化为 `LUNCUSDT`，`1000PEPEUSDT` 标准化为 `PEPEUSDT`。
2. **不规则标准化**：一些没有明显规律的标准化情况，我们可以通过手工输入的规则来指定。例如：

```python
SPOT_SPLIT_MAP = {
    "BCCUSDT": ["BCC1USDT", "BCCUSDT"],
    "BNXUSDT": ["BNX1USDT", "BNXUSDT"],
    "BTCSTUSDT": ["BTCST1USDT", "BTCSTUSDT"],
    "COCOSUSDT": ["COCOS1USDT", "COCOSUSDT"],
    "CVCUSDT": ["CVC1USDT", "CVCUSDT"],
    "DREPUSDT": ["DREP1USDT", "DREPUSDT"],
    "FTTUSDT": ["FTT1USDT", "FTTUSDT"],
    "KEYUSDT": ["KEY1USDT", "KEYUSDT"],
    "LUNAUSDT": ["LUNA1USDT", "LUNAUSDT"],
    "QUICKUSDT": ["QUICK1USDT", "QUICKUSDT"],
    "STRAXUSDT": ["STRAX1USDT", "STRAXUSDT"],
    "SUNUSDT": ["SUN1USDT", "SUNUSDT"],
    "VENUSDT": ["VEN1USDT", "VENUSDT"],
    "VIDTUSDT": ["VIDT1USDT", "VIDTUSDT"]
}

USDT_FUTURES_SPLIT_MAP = {
    'BNXUSDT': ['BNXUSDT', 'BNX1USDT'],
    'ICPUSDT': ['ICPUSDT', 'ICP1USDT'],
    'TLMUSDT': ['TLMUSDT', 'TLM1USDT'],
    'LUNAUSDT': ['LUNAUSDT', 'LUNA2USDT'],
    'DODOUSDT': ['DODOUSDT', 'DODOXUSDT']
}
```

定义标准化 Symbol 函数：

```python
def get_normalized_symbol(s: str, split_map):
    for norm_sym, splits in split_map.items():
        if s in splits:
            s = norm_sym
            break
    if s.startswith('1000'):
        return s[4:]
    return s

sym = 'BTCUSDT'
print(f'Normalize spot {sym} =', get_normalized_symbol(sym, SPOT_SPLIT_MAP))

sym = 'LUNA1USDT'
print(f'Normalize spot {sym} =', get_normalized_symbol(sym, SPOT_SPLIT_MAP))

sym = '1000PEPEUSDT'
print(f'Normalize usdt futures {sym} =', get_normalized_symbol(sym, USDT_FUTURES_SPLIT_MAP))

sym = 'DODOXUSDT'
print(f'Normalize usdt futures {sym} =', get_normalized_symbol(sym, USDT_FUTURES_SPLIT_MAP))
```

输出输出结果如下：

```
Normalize spot BTCUSDT = BTCUSDT
Normalize spot LUNA1USDT = LUNAUSDT
Normalize usdt futures 1000PEPEUSDT = PEPEUSDT
Normalize usdt futures DODOXUSDT = DODOUSDT
```

使用以下代码打印所有非标准匹配：

```python
spot_symbols = sorted(x.split('.')[0] for x in os.listdir(spot_dir))
spot_sym_map = defaultdict(list)
for sym in spot_symbols:
    sym_norm = get_normalized_symbol(sym, SPOT_SPLIT_MAP)
    spot_sym_map[sym_norm].append(sym)
usdt_futures_symbols = sorted(x.split('.')[0] for x in os.listdir(usdt_futures_dir))
usdt_futures_sym_map = defaultdict(list)
for sym in usdt_futures_symbols:
    sym_norm = get_normalized_symbol(sym, USDT_FUTURES_SPLIT_MAP)
    usdt_futures_sym_map[sym_norm].append(sym)

data = dict()
for sym_norm, syms_spot in spot_sym_map.items():
    if sym_norm in usdt_futures_sym_map:
        syms_futures = usdt_futures_sym_map[sym_norm]
        if len(syms_spot) > 1 or len(syms_futures) > 1 or syms_spot[0] != syms_futures[0]:
            data[sym_norm] = {'spot': str(syms_spot), 'usdt_futures': str(syms_futures)}

pd.DataFrame.from_dict(data, orient='index')
```

输出如下，都很符合常识：

|           | spot                        | usdt_futures              |
|:----------|:----------------------------|:--------------------------|
| BNXUSDT   | ['BNX1USDT', 'BNXUSDT']     | ['BNX1USDT', 'BNXUSDT']   |
| BONKUSDT  | ['BONKUSDT']                | ['1000BONKUSDT']          |
| BTCSTUSDT | ['BTCST1USDT', 'BTCSTUSDT'] | ['BTCSTUSDT']             |
| BTTCUSDT  | ['BTTCUSDT']                | ['1000BTTCUSDT']          |
| COCOSUSDT | ['COCOS1USDT', 'COCOSUSDT'] | ['COCOSUSDT']             |
| CVCUSDT   | ['CVC1USDT', 'CVCUSDT']     | ['CVCUSDT']               |
| DODOUSDT  | ['DODOUSDT']                | ['DODOUSDT', 'DODOXUSDT'] |
| FLOKIUSDT | ['FLOKIUSDT']               | ['1000FLOKIUSDT']         |
| FTTUSDT   | ['FTT1USDT', 'FTTUSDT']     | ['FTTUSDT']               |
| ICPUSDT   | ['ICPUSDT']                 | ['ICP1USDT', 'ICPUSDT']   |
| KEYUSDT   | ['KEY1USDT', 'KEYUSDT']     | ['KEYUSDT']               |
| LUNAUSDT  | ['LUNA1USDT', 'LUNAUSDT']   | ['LUNA2USDT', 'LUNAUSDT'] |
| LUNCUSDT  | ['LUNCUSDT']                | ['1000LUNCUSDT']          |
| PEPEUSDT  | ['PEPEUSDT']                | ['1000PEPEUSDT']          |
| SHIBUSDT  | ['SHIBUSDT']                | ['1000SHIBUSDT']          |
| STRAXUSDT | ['STRAX1USDT', 'STRAXUSDT'] | ['STRAXUSDT']             |
| TLMUSDT   | ['TLMUSDT']                 | ['TLM1USDT', 'TLMUSDT']   |
| XECUSDT   | ['XECUSDT']                 | ['1000XECUSDT']           |

## 现货与合约匹配

最后，基于以下代码，我们可以完成现货与永续合约的匹配：

```python
# 计算现货和合约时间戳交集
def intersect(spot_first_ts, spot_last_ts, fut_first_ts, fut_last_ts):
    first_ts = max(spot_first_ts, fut_first_ts)
    last_ts = min(spot_last_ts, fut_last_ts)
    if first_ts > last_ts:
        return None, None
    return first_ts, last_ts


def read_spot_with_match(symbol):
    path = os.path.join(spot_dir, f'{symbol}.pqt')
    df = pd.read_parquet(path)
    sym_norm = get_normalized_symbol(symbol, SPOT_SPLIT_MAP)

    df['swap_symbol'] = ''
    # 如果找不到对应的合约，则对应的合约赋值为空字符串
    if sym_norm not in usdt_futures_sym_map:
        return df

    spot_first_ts = df.index[0]
    spot_last_ts = df.index[-1]
    syms_futures = usdt_futures_sym_map[sym_norm]
    for sym_fut in syms_futures:
        path = os.path.join(usdt_futures_dir, f'{sym_fut}.pqt')
        df_fut = pd.read_parquet(path)
        fut_first_ts = df_fut.index[0]
        fut_last_ts = df_fut.index[-1]
        intersect_first_ts, intersect_last_ts = intersect(spot_first_ts, spot_last_ts, fut_first_ts, fut_last_ts)
        # 如果现货和合约时间戳有交集，则将对应的合约赋值为合约symbol
        if intersect_last_ts is not None and intersect_first_ts is not None:
            print(f'spot={symbol}, usdt_future={sym_fut}, match range {intersect_first_ts} -- {intersect_last_ts}')
            df.loc[intersect_first_ts:intersect_last_ts, 'swap_symbol'] = sym_fut
    return df

df = read_spot_with_match('LUNAUSDT')
cols = ['open', 'close', 'volume', 'swap_symbol']
print(pd.concat([df.loc[df['swap_symbol'] != '', cols].head(3), df.loc[df['swap_symbol'] != '', cols].tail(3)]).to_markdown())

df = read_spot_with_match('LUNA1USDT')
print(pd.concat([df.loc[df['swap_symbol'] != '', cols].head(3), df.loc[df['swap_symbol'] != '', cols].tail(3)]).to_markdown())

df = read_spot_with_match('LUNCUSDT')
print(pd.concat([df.loc[df['swap_symbol'] != '', cols].head(3), df.loc[df['swap_symbol'] != '', cols].tail(3)]).to_markdown())

df = read_spot_with_match('DODOUSDT')
print(pd.concat([df.loc[df['swap_symbol'] == 'DODOUSDT', cols].head(3), df.loc[df['swap_symbol'] == 'DODOUSDT', cols].tail(3)]).to_markdown())

print(pd.concat([df.loc[df['swap_symbol'] == 'DODOXUSDT', cols].head(3), df.loc[df['swap_symbol'] == 'DODOXUSDT', cols].tail(3)]).to_markdown())
```

对于 LUNA 系列，匹配结果如下：

spot=LUNAUSDT, usdt_future=LUNA2USDT, match range 2022-09-10 04:00:00+00:00 -- 2024-05-15 00:00:00+00:00
| candle_end_time           |   open |   close |           volume | swap_symbol   |
|:--------------------------|-------:|--------:|-----------------:|:--------------|
| 2022-09-10 04:00:00+00:00 | 5.2511 |  6.1084 |      8.0773e+06  | LUNA2USDT     |
| 2022-09-10 05:00:00+00:00 | 6.1082 |  6.1223 |      6.8396e+06  | LUNA2USDT     |
| 2022-09-10 06:00:00+00:00 | 6.12   |  5.7931 |      4.41605e+06 | LUNA2USDT     |
| 2024-05-14 22:00:00+00:00 | 0.5583 |  0.5599 | 196318           | LUNA2USDT     |
| 2024-05-14 23:00:00+00:00 | 0.5599 |  0.5569 | 179547           | LUNA2USDT     |
| 2024-05-15 00:00:00+00:00 | 0.5568 |  0.5549 | 208466           | LUNA2USDT     |

spot=LUNA1USDT, usdt_future=LUNAUSDT, match range 2021-01-28 08:00:00+00:00 -- 2022-05-12 16:00:00+00:00
| candle_end_time           |    open |   close |      volume | swap_symbol   |
|:--------------------------|--------:|--------:|------------:|:--------------|
| 2021-01-28 08:00:00+00:00 | 1.2468  | 1.3339  | 3.90032e+06 | LUNAUSDT      |
| 2021-01-28 09:00:00+00:00 | 1.334   | 1.4222  | 1.0363e+07  | LUNAUSDT      |
| 2021-01-28 10:00:00+00:00 | 1.4222  | 1.5627  | 1.27471e+07 | LUNAUSDT      |
| 2022-05-12 14:00:00+00:00 | 0.02018 | 0.01131 | 9.15404e+09 | LUNAUSDT      |
| 2022-05-12 15:00:00+00:00 | 0.0113  | 0.00657 | 1.72897e+10 | LUNAUSDT      |
| 2022-05-12 16:00:00+00:00 | 0.00656 | 0.00887 | 1.66841e+10 | LUNAUSDT      |

spot=LUNCUSDT, usdt_future=1000LUNCUSDT, match range 2022-09-09 14:00:00+00:00 -- 2024-05-15 00:00:00+00:00
| candle_end_time           |       open |      close |      volume | swap_symbol   |
|:--------------------------|-----------:|-----------:|------------:|:--------------|
| 2022-09-09 14:00:00+00:00 | 0.00046777 | 0.00045323 | 5.68616e+10 | 1000LUNCUSDT  |
| 2022-09-09 15:00:00+00:00 | 0.00045324 | 0.00046168 | 2.01121e+10 | 1000LUNCUSDT  |
| 2022-09-09 16:00:00+00:00 | 0.00046196 | 0.000456   | 2.17356e+10 | 1000LUNCUSDT  |
| 2024-05-14 22:00:00+00:00 | 0.00010284 | 0.0001029  | 1.14563e+09 | 1000LUNCUSDT  |
| 2024-05-14 23:00:00+00:00 | 0.00010289 | 0.00010235 | 1.3074e+09  | 1000LUNCUSDT  |
| 2024-05-15 00:00:00+00:00 | 0.00010235 | 0.00010208 | 1.06967e+09 | 1000LUNCUSDT  |

对于 `DODOUSDT`，合约被拆分为 `DODOUSDT` 和 `DODOXUSDT`，匹配结果如下：

spot=DODOUSDT, usdt_future=DODOUSDT, match range 2021-02-19 14:00:00+00:00 -- 2022-05-27 09:00:00+00:00
spot=DODOUSDT, usdt_future=DODOXUSDT, match range 2023-08-08 13:00:00+00:00 -- 2024-05-15 00:00:00+00:00
| candle_end_time           |   open |   close |           volume | swap_symbol   |
|:--------------------------|-------:|--------:|-----------------:|:--------------|
| 2021-02-19 14:00:00+00:00 |  6.785 |   6.302 |      3.22873e+06 | DODOUSDT      |
| 2021-02-19 15:00:00+00:00 |  6.3   |   5.625 |      4.43276e+06 | DODOUSDT      |
| 2021-02-19 16:00:00+00:00 |  5.627 |   5.9   |      3.04779e+06 | DODOUSDT      |
| 2022-05-27 07:00:00+00:00 |  0.135 |   0.14  |      1.12116e+06 | DODOUSDT      |
| 2022-05-27 08:00:00+00:00 |  0.14  |   0.142 |      1.17509e+06 | DODOUSDT      |
| 2022-05-27 09:00:00+00:00 |  0.142 |   0.143 | 439707           | DODOUSDT      |

| candle_end_time           |   open |   close |           volume | swap_symbol   |
|:--------------------------|-------:|--------:|-----------------:|:--------------|
| 2023-08-08 13:00:00+00:00 | 0.1475 |  0.1491 |      3.85968e+07 | DODOXUSDT     |
| 2023-08-08 14:00:00+00:00 | 0.1492 |  0.147  |      2.28892e+07 | DODOXUSDT     |
| 2023-08-08 15:00:00+00:00 | 0.147  |  0.133  |      4.4494e+07  | DODOXUSDT     |
| 2024-05-14 22:00:00+00:00 | 0.1661 |  0.1664 | 177964           | DODOXUSDT     |
| 2024-05-14 23:00:00+00:00 | 0.1664 |  0.1653 | 132254           | DODOXUSDT     |
| 2024-05-15 00:00:00+00:00 | 0.1653 |  0.1649 | 271666           | DODOXUSDT     |

通过上述方法，我们可以实现现货与合约的精确匹配，从而更为准确地研究保温杯策略


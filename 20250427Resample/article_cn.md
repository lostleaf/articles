# 扩展资金费字段：一种分钟偏移小时线资金费率处理方案

此前鳄鱼哥提出了 Bmac 在分钟偏移场景下[1 小时线资金费率问题](https://bbs.quantclass.cn/thread/59509)。

本文提出一种改进的 Resample 方案，用于解决非 0m 偏移情况下资金费率丢失的问题。

## 资金费率相关字段

在原始 0m 偏移小时线中，资金费主要涉及以下 3 个字段：

- funding\_rate：资金费率的主要字段，表示资金费率；
- candle\_begin\_time：K 线开盘时间，同时也是资金费收取时间；
- open：标的开盘价，同时也是收取资金费时的最新标价（实际上更准确应为 Mark Price，但与开盘价的价差通常可忽略）。

可以看到，在非 0m 偏移的小时线中，由于 K 线起始时间并非整点，因此资金费收取时间也不同于开盘时间，同时收取价格也不同于开盘价，因此需要对资金费相关字段进行扩展。

## 扩展方案

在分钟偏移的 K 线数据中，引入以下两个字段：

- funding\_time：资金费收取时间；
- funding\_price：资金费收取价格。

### 改进的 Resample 算法

通过改进 [BHDS 的 Polars Resample 算法](https://github.com/lostleaf/binance_datatool/blob/69aca4e3d4c6cf630e491be43f6b9c2c8aedf85f/generate/resample.py#L18)，实现如下：

```python

"""
将 Polars K 线 DataFrame 重采样到更高的周期，并且支持偏移量。
例如：把 5 分钟 K 线重采样为 1 小时 K 线，并向后偏移 5 分钟。
"""
df = df_btc.clone()

# 原始 K 线数据的时间间隔
time_interval = "1m"

# 目标重采样时间间隔
resample_interval = "4h"

# 对重采样后 K 线施加的偏移量
offset = "15m"

# 将时间间隔转换为 timedelta 对象
interval_delta = convert_interval_to_timedelta(time_interval)
resample_delta = convert_interval_to_timedelta(resample_interval)
offset_delta = convert_interval_to_timedelta(offset)

# 创建 Lazy DataFrame 以提升计算效率
ldf = df.lazy()

# 新增列：每根 K 线的结束时间
ldf = ldf.with_columns((pl.col("candle_begin_time") + interval_delta).alias("candle_end_time"))

# 聚合规则
agg = [
    pl.col("candle_begin_time").first().alias("candle_begin_time_real"),  # 重采样 K 线的真实开始时间
    pl.col("candle_end_time").last(),                                     # 重采样 K 线的结束时间
    pl.col("open").first(),                                               # 重采样 K 线的开盘价
    pl.col("high").max(),                                                 # 重采样区间内的最高价
    pl.col("low").min(),                                                  # 重采样区间内的最低价
    pl.col("close").last(),                                               # 重采样 K 线的收盘价
    pl.col("volume").sum(),                                               # 重采样区间内的成交量
    pl.col("quote_volume").sum(),                                         # 重采样区间内的计价成交量
    pl.col("trade_num").sum(),                                            # 重采样区间内的成交笔数
    pl.col("taker_buy_base_asset_volume").sum(),                          # 重采样区间内的主动买入基础资产成交量
    pl.col("taker_buy_quote_asset_volume").sum(),                         # 重采样区间内的主动买入计价资产成交量
]

if "avg_price_1m" in df.columns:
    # 重采样周期内第一分钟的平均价
    agg.append(pl.col("avg_price_1m").first())

if "funding_rate" in df.columns:
    # 仅当绝对值大于 0.01 bps 时才认为 funding_rate 有效
    has_funding_cond = pl.col("funding_rate").abs() > 1e-6

    # 获取首个有效 funding_rate 及其对应的价格与时间
    agg.extend([
        pl.col("funding_rate").filter(has_funding_cond).first().alias("funding_rate"),
        pl.col("open").filter(has_funding_cond).first().alias("funding_price"),
        pl.col("candle_begin_time").filter(has_funding_cond).first().alias("funding_time")
    ])

# 以 K 线开始时间为键，按指定周期和偏移量进行动态分组重采样
ldf = ldf.group_by_dynamic("candle_begin_time", every=resample_delta, offset=offset_delta).agg(agg)

# 过滤掉长度不足重采样周期的 K 线
ldf = ldf.filter((pl.col("candle_end_time") - pl.col("candle_begin_time_real")) == resample_delta)

# 删除用于中间计算的临时列
ldf = ldf.drop(["candle_begin_time_real"])

print(
    ldf.collect()[
        'candle_begin_time',
        'candle_end_time',
        'open',
        'close',
        'funding_rate',
        'funding_price',
        'funding_time'
    ].tail(10)
)
```

### Resample 结果示例

基于 BHDS 框架生成的 1 分钟线，对 U 本位 BTCUSDT 合约进行 4 小时 K 线重采样，部分关键列结果如下：

```
# shape: (10, 7)
┌─────────────────────────┬─────────────────────────┬─────────┬─────────┬──────────────┬───────────────┬─────────────────────────┐
│ candle_begin_time       ┆ candle_end_time         ┆ open    ┆ close   ┆ funding_rate ┆ funding_price ┆ funding_time            │
│ ---                     ┆ ---                     ┆ ---     ┆ ---     ┆ ---          ┆ ---           ┆ ---                     │
│ datetime[ms, UTC]       ┆ datetime[ms, UTC]       ┆ f64     ┆ f64     ┆ f64          ┆ f64           ┆ datetime[ms, UTC]       │
╞═════════════════════════╪═════════════════════════╪═════════╪═════════╪══════════════╪═══════════════╪═════════════════════════╡
│ 2025-04-20 04:15:00 UTC ┆ 2025-04-20 08:15:00 UTC ┆ 85113.7 ┆ 84730.1 ┆ -0.000011    ┆ 84736.6       ┆ 2025-04-20 08:00:00 UTC │
│ 2025-04-20 08:15:00 UTC ┆ 2025-04-20 12:15:00 UTC ┆ 84730.0 ┆ 84266.8 ┆ null         ┆ null          ┆ null                    │
│ 2025-04-20 12:15:00 UTC ┆ 2025-04-20 16:15:00 UTC ┆ 84266.8 ┆ 84396.4 ┆ -0.000023    ┆ 84304.6       ┆ 2025-04-20 16:00:00 UTC │
│ 2025-04-20 16:15:00 UTC ┆ 2025-04-20 20:15:00 UTC ┆ 84396.5 ┆ 84899.3 ┆ null         ┆ null          ┆ null                    │
│ 2025-04-20 20:15:00 UTC ┆ 2025-04-21 00:15:00 UTC ┆ 84899.3 ┆ 85295.4 ┆ 0.000052     ┆ 85138.9       ┆ 2025-04-21 00:00:00 UTC │
│ 2025-04-21 00:15:00 UTC ┆ 2025-04-21 04:15:00 UTC ┆ 85295.3 ┆ 87214.9 ┆ null         ┆ null          ┆ null                    │
│ 2025-04-21 04:15:00 UTC ┆ 2025-04-21 08:15:00 UTC ┆ 87214.9 ┆ 87489.6 ┆ 0.000054     ┆ 87400.7       ┆ 2025-04-21 08:00:00 UTC │
│ 2025-04-21 08:15:00 UTC ┆ 2025-04-21 12:15:00 UTC ┆ 87489.7 ┆ 87287.3 ┆ null         ┆ null          ┆ null                    │
│ 2025-04-21 12:15:00 UTC ┆ 2025-04-21 16:15:00 UTC ┆ 87287.2 ┆ 88075.6 ┆ 0.000016     ┆ 87997.3       ┆ 2025-04-21 16:00:00 UTC │
│ 2025-04-21 16:15:00 UTC ┆ 2025-04-21 20:15:00 UTC ┆ 88075.5 ┆ 87171.5 ┆ null         ┆ null          ┆ null                    │
└─────────────────────────┴─────────────────────────┴─────────┴─────────┴──────────────┴───────────────┴─────────────────────────┘
```

## 结论

目前分享会官方框架虽然已较为先进，但仍有不少细节值得进一步打磨。

如果大家认为这种 K 线存储方式有意义，可考虑基于此种结构编写回测算法，实盘 BMAC 框架也可以采用该数据结构。


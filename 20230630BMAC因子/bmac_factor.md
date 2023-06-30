# 基于 BMAC 的实盘因子计算框架

BMAC 这个共享K线框架已经发布一段时间，本文主要介绍一个基于 BMAC 的实盘中性因子计算框架

## BMAC v1.1: 支持实盘资金费率获取

为了使 BMAC 支持中性框架 factor 及 filter 计算，我对 BMAC 进行了小升级，使用以下配置即可实盘获取K线的同时获取资金费率

```json
{
    "interval": "1h",
    "http_timeout_sec": 3,
    "candle_close_timeout_sec": 12,
    "trade_type": "usdt_swap",
    "funding_rate": true,
    "dingding": {
        "error": {
            "access_token": "f06e5.....",
            "secret": "SEC439...."        
        }
    }
}
```

或可参考 [github](https://github.com/lostleaf/binance_market_async_crawler/blob/master/usdt_1h_alpha_example/config.json.example)

## 程序设计思路

本程序每小时重复执行以下4步操作：

1. 从 BMAC 加载市场信息，即当前正在交易的合约信息
2. 从 BMAC 加载资金费率
3. 从 BMAC 读取 K线，并计算因子
4. 打 log，记录这一轮计算的因子

在第 4 步后，增加仓位计算模块和下单模块，则构成完整的实盘策略

## 加载市场信息

这一步非常简单，等待 BMAC 存储好市场信息并读取

```python
def load_market(exg_mgr: CandleFeatherManager, run_time, bmac_expire_sec):
    # 从 BMAC 读取合约列表
    expire_time = run_time + timedelta(seconds=bmac_expire_sec)
    is_ready = wait_until_ready(exg_mgr, 'exginfo', run_time, expire_time)

    if not is_ready:
        raise RuntimeError(f'exginfo not ready at {now_time()}')

    df_exg: pd.DataFrame = exg_mgr.read_candle('exginfo')
    symbol_list = list(df_exg['symbol'])
    return symbol_list
```

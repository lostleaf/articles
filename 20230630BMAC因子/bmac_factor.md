# 基于 BMAC 的实盘中性因子计算框架

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

这一步非常简单，等待 BMAC 存储好市场信息并读取，返回当前正在交易的合约列表

```python
# market.py

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

## 加载资金费

这一步同样简单，等待 BMAC 存储好资金费并读取，返回当前资金费 DataFrame

```python
# market.py

def get_fundingrate(exg_mgr: CandleFeatherManager, run_time, expire_sec):
    # 从 BMAC 读取资金费
    expire_time = run_time + timedelta(seconds=expire_sec)
    is_ready = wait_until_ready(exg_mgr, 'funding', run_time, expire_time)

    if not is_ready:
        raise RuntimeError(f'Funding rate not ready')

    return exg_mgr.read_candle('funding')
```

## 计算因子

首先我们定义因子计算器, 由于 factor 和 filter 参数以及命名方式有差异，因此计算器分为 factor 计算器和 filter 计算器

两个工厂函数 `create_XXX_calc_from_alpha_config` 中的参数 `cfg`，均对应中性实盘配置

```python
# calculator.py

import importlib

import pandas as pd

# factor 计算器
class FactorCalculator:

    def __init__(self, factor, back_hour, d_num):
        self.backhour = int(back_hour)
        self.d_num = d_num

        factor_module_name = f'factors.{factor}'
        module = importlib.import_module(factor_module_name)
        self.signal_func = module.signal

        if d_num == 0:
            self.factor_name = f'{factor}_bh_{back_hour}'
        else:
            self.factor_name = f'{factor}_bh_{back_hour}_diff_{d_num}'

    def calc(self, df: pd.DataFrame):
        return self.signal_func(df, self.backhour, self.d_num, self.factor_name)


# filter 计算器
class FilterCalculator:

    def __init__(self, filter, params):
        self.filter_name = f'{filter}_fl_{params}'
        self.params = params

        filter_module_name = f'filters.{filter}'
        module = importlib.import_module(filter_module_name)
        self.signal_func = module.signal

    def calc(self, df: pd.DataFrame):
        return self.signal_func(df, self.params, self.filter_name)


def create_factor_calc_from_alpha_config(cfg):
    factor, if_reverse, back_hour, d_num, weight = cfg
    return FactorCalculator(factor, back_hour, d_num)


def create_filter_calc_from_alpha_config(cfg):
    filter, params = cfg
    return FilterCalculator(filter, params)
```

接着定义 `fetch_swap_candle_data_and_calc_factors_filters` 函数，从 BMAC 读取K线数据并计算因子

其中使用了忙等待这种高频常用技巧，因为 BMAC 为 IO 密集型程序，其中大量时间被用于等待 http 请求，并且各合约K线就绪时间不一，而算因子属于 CPU 密集型程序，使用该技巧能更好地利用 CPU，在等待未就绪K线的同时计算已就绪K线合约因子，降低整体延时

```python
# calc.py

def fetch_swap_candle_data_and_calc_factors_filters(candle_mgr: CandleFeatherManager, symbol_list, run_time, expire_sec,
                                                    factor_calcs, filter_calcs):
    unready_symbols = set(symbol_list)
    expire_time = run_time + timedelta(seconds=expire_sec)
    symbol_data = dict()

    # 算因子（忙等待）
    while True:
        while len(unready_symbols) > 0:
            readys = {s for s in unready_symbols if candle_mgr.check_ready(s, run_time)}
            if len(readys) == 0:
                break
            for sym in readys:
                df = candle_mgr.read_candle(sym)
                df['symbol'] = sym
                for factor_calc in factor_calcs:
                    factor_calc.calc(df)
                for filter_calc in filter_calcs:
                    filter_calc.calc(df)
                symbol_data[sym] = df
            unready_symbols -= readys
            logging.log(MY_DEBUG_LEVEL, 'readys=%d, unready=%d, read=%d', len(readys), len(unready_symbols),
                        len(symbol_data))
        if len(unready_symbols) == 0:
            break
        if now_time() > expire_time:
            break
        time.sleep(0.01)
    return symbol_data

```


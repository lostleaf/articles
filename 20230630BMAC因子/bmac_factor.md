# 基于 BMAC 的中性选币框架（只选币不下单）

BMAC 这个共享K线框架已经发布一段时间，本文基于 BMAC，搭建一个单因子，HH=1，固定资金的中性选币框架，不包含下单模块

## 实盘交易的软件工程

在进入代码之前，我们先来从软件工程的角度，谈谈量化交易实盘系统的软件架构

我认为绝大部分的交易策略，实盘系统可大致分为四个模块：**数据**, **因子**, **仓位**, **执行**

1. 数据：`market.py`，从 BMAC 等待并读取本周期交易的合约基本信息（包括合约名称，最小下单量等），以及资金费率
2. 因子：`factor_calc.py`，等待 BMAC 获取 K线数据，并计算本周期 factors 和 filters
3. 仓位：`position.py`，首先基于上一步算出的 filters，过滤出本周期可用的合约池，再基于 factors 排序，从合约池从选出需要做多和做空的合约，分配固定资金给每个合约
4. 执行：本框架不涉及下单执行，仅做仓位记录

## 代码实现

### 1. 数据

数据包含以下两块：获取合约基本信息，和获取资金费，主要代码在 `market.py`

#### 加载市场信息

这一步非常简单，等待 BMAC 存储好市场信息并读取，返回当前正在交易的合约列表

```python

def load_market(exg_mgr: CandleFeatherManager, run_time, bmac_expire_sec):
    # 从 BMAC 读取合约列表
    expire_time = run_time + timedelta(seconds=bmac_expire_sec)
    is_ready = wait_until_ready(exg_mgr, 'exginfo', run_time, expire_time)

    if not is_ready:
        raise RuntimeError(f'exginfo not ready at {now_time()}')

    df_exg: pd.DataFrame = exg_mgr.read_candle('exginfo')
    return df_exg
```

#### 加载资金费

这一步同样简单，等待 BMAC 存储好资金费并读取，返回当前资金费 DataFrame

```python

def get_fundingrate(exg_mgr: CandleFeatherManager, run_time, expire_sec):
    # 从 BMAC 读取资金费
    expire_time = run_time + timedelta(seconds=expire_sec)
    is_ready = wait_until_ready(exg_mgr, 'funding', run_time, expire_time)

    if not is_ready:
        raise RuntimeError(f'Funding rate not ready')

    return exg_mgr.read_candle('funding')
```

### 2. 因子

因子计算代码主要在 `factor_calc.py`

#### 因子计算器

首先我们定义因子计算器, 由于 factor 和 filter 参数以及命名方式有差异，因此计算器分为 factor 计算器和 filter 计算器

两个工厂函数 `create_XXX_calc_from_alpha_config` 中的参数 `cfg`，均对应中性实盘配置

```python

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

#### 因子计算调用

接着定义 `fetch_swap_candle_data_and_calc_factors_filters` 函数，从 BMAC 读取K线数据并计算因子，并返回当前周期最新的因子

其中使用了忙等待这种高频常用技巧，因为 BMAC 为 IO 密集型程序，其中大量时间被用于等待 http 请求，并且各合约K线就绪时间不一，而算因子属于 CPU 密集型程序，使用该技巧能更好地利用 CPU，在等待未就绪K线的同时计算已就绪K线合约因子，降低整体延时

在计算因子时，会略过上市时间不足的合约（ K线数量 < 999）

```python

def fetch_swap_candle_data_and_calc_factors_filters(candle_mgr: CandleFeatherManager, symbol_list, run_time, expire_sec,
                                                    factor_calcs, filter_calcs, min_candle_num):
    unready_symbols = set(symbol_list)
    expire_time = run_time + timedelta(seconds=expire_sec)
    symbol_data = dict()

    # 算因子（忙等待）
    while True:
        while len(unready_symbols) > 0:
            readies = {s for s in unready_symbols if candle_mgr.check_ready(s, run_time)}
            if len(readies) == 0:
                break
            for sym in readies:
                df = candle_mgr.read_candle(sym)
                if len(df) < min_candle_num:
                    continue
                df['symbol'] = sym
                for factor_calc in factor_calcs:
                    factor_calc.calc(df)
                for filter_calc in filter_calcs:
                    filter_calc.calc(df)
                symbol_data[sym] = df
            unready_symbols -= readies
            logging.log(MY_DEBUG_LEVEL, 'readys=%d, unready=%d, read=%d', len(readies), len(unready_symbols),
                        len(symbol_data))
        if len(unready_symbols) == 0:
            break
        if now_time() > expire_time:
            break
        time.sleep(0.01)
    current_hour_results = [df.iloc[-1] for df in symbol_data.values()]
    df_factor = pd.DataFrame(current_hour_results).reset_index(drop=True)
    return df_factor
```

### 3. 仓位

仓位模块涉及过滤、选币、以及资金分配，代码为 `position.py`

#### 过滤

过滤包含两个部分，一是过滤无效因子(inf 和 nan)，二是根据 filter 因子选出可用合约池，目前直接硬编码了 F 神框架中的过滤渣币和交易量

```python

def filiter_nan_inf(df_factor, factor_filter_cols):
    # 过滤 nan 及 inf
    with pd.option_context('mode.use_inf_as_na', True):  
        return df_factor[~df_factor[factor_filter_cols].isna().any(axis=1)]

def filter_before(df1: pd.DataFrame):
    long_condition1 = df1['ChgPctMax_fl_24'].between(-1e+100, 0.2, inclusive='both')
    df1 = df1.loc[long_condition1]

    df1[f'Volume_fl_24_rank'] = df1.groupby('candle_begin_time')['Volume_fl_24'].rank(method='first',
                                                                                      pct=False,
                                                                                      ascending=False)
    long_condition2 = df1[f'Volume_fl_24_rank'].between(-1e+100, 60, inclusive='both')
    df1 = df1.loc[long_condition2]

    return df1
```

#### 选币

对于过滤过的合约池，直接根据单因子排序，并对首尾合约做空/做多

```python

def select_coin(df_factor: pd.DataFrame, factor_name, ascending, long_num, short_num):
    df_factor = df_factor.sort_values(factor_name, ascending=ascending, ignore_index=True)
    df_short = df_factor.head(short_num).copy()
    df_long = df_factor.tail(long_num).copy()

    return df_long, df_short
```

#### 资金分配

基于配置中的固定资金，一半分配给多头，一半分配给空头，空头/多头均仓分配给选出的合约

每个合约根据收盘价和分配的资金，计算仓位，并根据交易规则舍入仓位至指定精度

```python

def assign_position(df_long, df_short, df_exg, capital_usdt, long_num, short_num):

    def pos(capital, close, face_value):
        return round_to_tick(Decimal(capital / close), face_value)

    long_coin_capital = capital_usdt / 2 / long_num
    df_long['position'] = df_long.apply(
        lambda r: pos(long_coin_capital, r['close'], df_exg.at[r['symbol'], 'face_value']), axis=1)
    df_long['pos_val'] = df_long['position'].astype(float) * df_long['close']

    short_coin_capital = capital_usdt / 2 / short_num
    df_short['position'] = df_short.apply(
        lambda r: pos(short_coin_capital, r['close'], df_exg.at[r['symbol'], 'face_value']), axis=1)
    df_short['pos_val'] = df_short['position'].astype(float) * df_short['close']

    return df_long, df_short
```

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

## 如何使用

本框架与bmac一样，通过读取工作目录下的 json 配置文件初始化程序，并每小时执行因子计算

配置文件样例如下，可参考 `alpha_1h_example` 文件夹下的 json 样例文件

```json
{
    "interval": "1h",  执行周期
    "bmac_dir": "../usdt_1h_alpha",  bmac 文件夹
    "bmac_expire_sec": 30,  bmac 超时时间（秒）
    "factors": [["AdaptBollingv3", true, 120, 0, 1]],  中性 factor 配置
    "filters": [["涨跌幅max", 24], ["Volume", 24]],  中性 filter 配置
    "debug": false  是否启用 debug 模式，debug 模式会立即运行一次并退出，主要用于检测程序及配置的正确性
}
```

然后执行 `python startup.py 配置所在文件夹` 即可

例如可以将 `alpha_1h_example` 文件夹下 `factor_calc.json.example` 文件更名为 `factor_calc.json`，然后执行 `python startup.py alpha_1h_example`

## 程序设计思路

本程序每小时重复执行以下4步操作：

1. 从 BMAC 加载市场信息，即当前正在交易的合约信息
2. 从 BMAC 加载资金费率
3. 从 BMAC 读取 K线，并计算因子
4. 打 log，记录这一轮计算的因子，将本轮计算的最新因子存储在 factor_his 文件夹下 csv 文件中

在第 4 步后，增加仓位计算模块和下单模块，则构成完整的实盘策略





## 主程序

主程序本质上为一个大 loop，每小时运行以上四个步骤，出错重试

```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-

import logging
import os
import sys
import time
import traceback
from datetime import timedelta

import pandas as pd

from config import QuantConfig
from market import get_fundingrate, load_market
from utils.commons import (MY_DEBUG_LEVEL, next_run_time, sleep_until_run_time)
from calc import fetch_swap_candle_data_and_calc_factors_filters

sys.stdout.reconfigure(encoding='utf-8')

# 调试用，实盘可删除参数 level=MY_DEBUG_LEVEL
logging.addLevelName(MY_DEBUG_LEVEL, 'MyDebug')
logging.basicConfig(format='%(asctime)s (%(levelname)s) - %(message)s', level=MY_DEBUG_LEVEL, datefmt='%Y%m%d %H:%M:%S')


def run_loop(Q: QuantConfig):
    run_time = next_run_time('1h')
    if Q.debug:
        run_time -= timedelta(hours=1)

    # sleep 到小时开始
    logging.info(f'Next run time: {run_time}')
    sleep_until_run_time(run_time)

    # 1 加载市场信息
    symbol_list = load_market(Q.exg_mgr, run_time, Q.bmac_expire_sec)
    logging.info('获取当前周期合约完成')
    logging.log(MY_DEBUG_LEVEL, symbol_list[:5])

    # 2 获取当前资金费率
    df_funding = get_fundingrate(Q.exg_mgr, run_time, Q.bmac_expire_sec)
    logging.info('获取资金费数据完成')
    logging.log(MY_DEBUG_LEVEL, '\n' + str(df_funding.head(3)))
    logging.log(MY_DEBUG_LEVEL, '\n' + str(df_funding.tail(3)))

    # 3 算因子
    symbol_data = fetch_swap_candle_data_and_calc_factors_filters(Q.candle_mgr, symbol_list, run_time,
                                                                  Q.bmac_expire_sec, Q.factor_calcs, Q.filter_calcs)
    logging.info('计算所有币种K线因子完成')

    # 4 打 log，将本轮计算的最新因子存储在 factor_his 文件夹下
    output_dir = os.path.join(Q.workdir, 'factor_his')
    if not os.path.exists(output_dir):
        os.makedirs(output_dir)
    output_path = os.path.join(output_dir, run_time.strftime('coin_%Y%m%d_%H%M%S.csv.zip'))

    current_hour_results = [df.iloc[-1] for df in symbol_data.values()]
    df_factor = pd.DataFrame(current_hour_results)
    df_factor.to_csv(output_path, index=False)

    df_factor.drop(columns=[
        'open', 'high', 'low', 'close_time', 'quote_volume', 'trade_num', 'taker_buy_base_asset_volume',
        'taker_buy_quote_asset_volume'
    ],
                   inplace=True)
    logging.log(MY_DEBUG_LEVEL, '\n' + str(df_factor.head(3)))

    if Q.debug:
        exit()


def main(workdir):
    # 初始化配置
    Q = QuantConfig(workdir)

    while True:
        try:
            while True:
                run_loop(Q)
        except Exception as err:
            logging.error('系统出错, 10s之后重新运行, 出错原因: ' + str(err))
            traceback.print_exc()
            time.sleep(10)


if __name__ == '__main__':
    main(sys.argv[1])
```

## 实盘日志

debug 模式，实盘运行日志

```
20230630 17:11:17 (INFO) - Next run time: 2023-06-30 17:00:00+08:00
20230630 17:11:17 (INFO) - 获取当前周期合约完成
20230630 17:11:17 (MyDebug) - ['BTCUSDT', 'ETHUSDT', 'BCHUSDT', 'XRPUSDT', 'EOSUSDT']
20230630 17:11:17 (INFO) - 获取资金费数据完成
20230630 17:11:17 (MyDebug) - 
      symbol  fundingRate                      time
0  SUSHIUSDT     0.000100 2023-04-29 05:00:00+08:00
1    BTSUSDT     0.000100 2023-04-29 05:00:00+08:00
2    INJUSDT    -0.000869 2023-04-29 05:00:00+08:00
20230630 17:11:17 (MyDebug) - 
          symbol  fundingRate                      time
352895  DUSKUSDT       0.0001 2023-06-30 17:00:00+08:00
352896  CTSIUSDT       0.0001 2023-06-30 17:00:00+08:00
352897   ACHUSDT       0.0001 2023-06-30 17:00:00+08:00
20230630 17:11:21 (MyDebug) - readys=187, unready=0, read=187
20230630 17:11:21 (INFO) - 计算所有币种K线因子完成
20230630 17:11:21 (MyDebug) - 
             candle_begin_time     close        volume    symbol  AdaptBollingv3_bh_120  涨跌幅max_fl_24  Volume_fl_24
1499 2023-06-30 16:00:00+08:00   0.23000  3.227302e+06   LRCUSDT               0.571843      0.018544  9.760639e+06
1499 2023-06-30 16:00:00+08:00  97.34000  1.280859e+06   LTCUSDT               0.519714      0.076120  9.781497e+08
1499 2023-06-30 16:00:00+08:00   0.05079  1.484064e+07  COTIUSDT               0.073205      0.015477  9.908935e+06
```


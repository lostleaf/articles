# Neo 框架 V1.1 升级：策略编写规范化，及并行网格搜索初探

## 前言

本文提出 Neo 框架 V1.1 升级：

- 提出策略编写的新规范，所有按照规范编写的策略可无缝接入回测系统
- 重构代码，更加精准地控制 jit 编译，保证每次回测仅 jit 编译一次
- 多进程并行网格搜索，在开启 16 进程进行回测的场景下，相比邢大框架，提速大约 5 倍，误差仍控制在 0.3% 以内

Neo 框架 1.0 版本的技术报告在帖子: [NEO 回测系统：基于 numba 的类 vnpy 高效趋势回测系统
](https://bbs.quantclass.cn/thread/14050)

## 代码目录

本次更新后，NEO 框架包含以下代码

```
.
├── neo_backtesting  回测引擎核心代码
│   ├── __init__.py
│   ├── backtest.py
│   └── simulator.py
├── notebook  回测执行 Jupyter Notebook
│   └── boll_inbar.ipynb
├── strategy  策略代码
│   ├── __init__.py
│   └── boll_inbar.py
├── util  功能函数
│   ├── __init__.py
│   ├── candle.py
│   └── time.py
└── xbx  邢大相关代码
    ├── __init__.py
    ├── backtest.py
    └── strategy.py
```

其中 xbx 文件夹为邢大课程代码，并不是 NEO 框架必要文件

## 策略编写新规范

在 NEO v1.1 版本中，每个策略为 strategy 包中一个单独的 Python 文件，至少包含：

1. 因子计算函数：`factor(candles, params)`，其输入为两个字典，`candles` 和 `params`，输出为包含 OHLCV 和因子的 DataFrame
2. 因子列名常量数组: `FCOLS`
3. 策略 jitclass: `Strategy`，和 v1.0 版本一样，需要包含一个 `on_bar(self, candle, factors, pos, equity)` 函数，然而区别在于 `candle` 和 `factors` 参数的类型变为 `np.void`，这是因为使用了 *numpy structured array* 代替了普通 *numpy array*，好处是每一字段都有名字，减小了出错的概率
4. 函数 `get_default_factor_params_list`: 因子默认参数列表，网格搜索使用
5. 函数 `get_default_strategy_params_list`: 策略默认参数列表，网格搜索使用

那么，v1.0 版本中，基于小时线计算布林轨道，基于分钟线突破的布林带策略，则为

``` python
# strategy/boll_inbar.py

import numba as nb
import numpy as np
import pandas as pd
import talib as ta
from numba.experimental import jitclass

# 因子列名
FCOLS = ['upper', 'median', 'lower']


def factor(candles, params):
    # 取 K 线和参数
    df_1m = candles['1m']
    df_long = candles[params['itl']]
    n = params['n']
    b = params['b']

    # 计算布林带
    upper, median, lower = ta.BBANDS(df_long['close'], timeperiod=n, nbdevup=b, nbdevdn=b, matype=ta.MA_Type.SMA)
    df_fac = pd.DataFrame({
        'upper': upper,
        'median': median,
        'lower': lower,
        'candle_end_time': df_long['candle_end_time']
    })

    # 填充到1分钟
    df_fac.set_index('candle_end_time', inplace=True)
    df_fac = df_1m.join(df_fac, on='candle_end_time')
    for col in FCOLS:
        df_fac[col].ffill(inplace=True)

    # 包含：candle_begin_time, candle_end_time, open, close, high, low, volume, upper, median, lower
    # df_fac 是一个新的 DataFrame，（最好）不要污染原来的 DataFrame
    return df_fac


@jitclass([
    ['leverage', nb.float64],  # 杠杆率 
    ['face_value', nb.float64],  # 合约面值
    ['prev_upper', nb.float64],  # 上根k线上轨
    ['prev_lower', nb.float64],  # 上根k线下轨
    ['prev_median', nb.float64],  # 上根k线均线
    ['prev_close', nb.float64]  # 上根k线收盘价
])
class Strategy:

    def __init__(self, leverage, face_value):
        self.leverage = leverage
        self.face_value = face_value

        self.prev_upper = np.nan
        self.prev_lower = np.nan
        self.prev_median = np.nan
        self.prev_close = np.nan

    def on_bar(self, candle, factors, pos, equity):
        # candle 和 factors 可以当做字典
        cl = candle['close']
        upper = factors['upper']
        lower = factors['lower']
        median = factors['median']

        # 默认保持原有仓位
        target_pos = pos

        if not np.isnan(self.prev_close):
            # 做空或无仓位，上穿上轨，做多
            if pos <= 0 and cl > upper and self.prev_close <= self.prev_upper:
                target_pos = int(equity * self.leverage / cl / self.face_value)

            # 做多或无仓位，下穿下轨，做空
            elif pos >= 0 and cl < lower and self.prev_close >= self.prev_lower:
                target_pos = -int(equity * self.leverage / cl / self.face_value)

            # 做多，下穿中轨，平仓
            elif pos > 0 and cl < median and self.prev_close >= self.prev_median:
                target_pos = 0

            # 做空，上穿中轨，平仓
            elif pos < 0 and cl > median and self.prev_close <= self.prev_median:
                target_pos = 0

        # 更新上根K线数据
        self.prev_upper = upper
        self.prev_lower = lower
        self.prev_close = cl
        self.prev_median = median

        return target_pos


def get_default_factor_params_list():
    params = []
    for interval in ['1h', '30m']:  # 长周期
        for n in range(10, 501, 5):  # 均线周期
            for b in [1.5, 1.8, 2, 2.2, 2.5]:  # 布林带宽度
                params.append({'itl': interval, 'n': n, 'b': b})
    return params


def get_default_strategy_params_list():
    params = []
    for lev in [1, 1.5]:
        params.append({'leverage': lev})

    return params

```

## 新的回测模块

### 回测器 API
v1.1 中，回测模块被重构，其接口为：

```python
# neo_backtesting/backtest.py

class Backtester:

    def __init__(self, candle_paths, contract_type, simulator_params, stra_module):
        """
        candle_paths: K线路径字典，例如 {'1m': 'ETH-USDT_1m.fea'}
        contract_type: 合约类型, 'futures' 正向合约, 'inverse_futures' 反向合约
        simulator_params: 模拟器参数字典，需要与模拟器参数一一对应
        stra_module: 策略模组, 例如 strategy.boll_inbar
        """
        pass

    def run_detailed(self, start_date, end_date, init_capital, face_value, factor_params, strategy_params):
        """
        详细运行一组参数
        start_date: 起始时间, '20180301'
        end_date: 结束时间, '20220901'
        init_capital: 起始资金, 正向合约为 USDT，反向合约为币数 
        face_value: 合约面值, 0.001 等 
        factor_params: 因子参数字典, {'n': 100, 'b': 2.0} 
        strategy_params: 策略参数字典, {'leverage': 1.5}

        返回值：DataFrame, 在 factor 函数返回的 df_fac 基础上增加两列, 仓位 pos 和权益 equity 
        """
        pass

    def run_multi(self, start_date, end_date, init_capital, face_value, fparams_list, sparams_list):
        """
        详细运行多组参数，对 fparams_list, sparams_list 运行网格搜索
        fparams_list: factor_params 数组
        sparams_list: strategy_params 数组

        返回值：DataFrame, 每组参数对应的净值 
        """
        pass
```

其具体实现可参考 `backtest.py`

### 使用 NEO v1.1 回测 boll_inbar 策略
下面通过一个例子调用 `Backtester` 回测 `boll_inbar` 策略, 代码为 notebook/boll_inbar.ipynb

首先定义回测参数

```python
# 回测起始结束日
START_DATE = '20180301'
END_DATE = '20220901'

# 策略参数
N = 100
B = 1.8
LONG_INTERVAL = '1h'

LEVERAGE = 1


# 回测参数
INIT_CAPITAL = 1e5  # 初始资金，10万
FACE_VALUE = 0.001  # 合约面值 0.001

COMM_RATE = 6e-4  # 交易成本万分之 6
LIQUI_RATE = 5e-3  # 爆仓保证金率千分之 5

CONTRACT_TYPE = 'futures'  # 正向合约

# 模拟器参数
SIMULATOR_PARAMS = {
    'init_capital': INIT_CAPITAL, 
    'face_value': FACE_VALUE, 
    'comm_rate': COMM_RATE, 
    'liqui_rate': LIQUI_RATE, 
    'init_pos': 0
}

# from strategy import boll_inbar
STRA = boll_inbar  # 要回测的策略，注意 boll_inbar 是需要被 import 的

ETH_PATHS = {
    '1m': '/home/lostleaf/feather_data/spot/ETH-USDT_1m.fea',
    '30m': '/home/lostleaf/feather_data/spot/ETH-USDT_30m.fea',
    '1h': '/home/lostleaf/feather_data/spot/ETH-USDT_1h.fea'
}
```

以下代码调用回测

```python
%time backtester = Backtester(ETH_PATHS, 'futures', SIMULATOR_PARAMS, boll_inbar)

factor_params = {
    'n': N,
    'b': B,
    'itl': LONG_INTERVAL
}

strategy_params = {
    'leverage': LEVERAGE
}

%time df_ret = backtester.run_detailed(START_DATE, END_DATE, INIT_CAPITAL, FACE_VALUE, factor_params, strategy_params)
df_ret['equity'] /= df_ret['equity'].iat[0]
print(df_ret[['candle_begin_time', 'close', 'pos', 'equity']])
```

运行结果为

```
CPU times: user 575 ms, sys: 610 ms, total: 1.19 s
Wall time: 513 ms
CPU times: user 1.26 s, sys: 694 ms, total: 1.96 s
Wall time: 1.9 s
          candle_begin_time    close       pos      equity
275636  2018-03-01 00:00:00   853.00         0    1.000000
275637  2018-03-01 00:01:00   852.80         0    1.000000
275638  2018-03-01 00:02:00   853.01         0    1.000000
275639  2018-03-01 00:03:00   852.97         0    1.000000
275640  2018-03-01 00:04:00   851.00         0    1.000000
...                     ...      ...       ...         ...
2638630 2022-08-31 23:56:00  1553.80  15620134  242.558917
2638631 2022-08-31 23:57:00  1554.28  15620134  242.633894
2638632 2022-08-31 23:58:00  1555.01  15620134  242.747921
2638633 2022-08-31 23:59:00  1554.10  15620134  242.605778
2638634 2022-09-01 00:00:00  1553.04  15620134  242.440204

[2362999 rows x 4 columns]
```

可以看到，Backtester 初始化，包括载入数据和 jit 编译，耗时 513 毫秒
回测耗时 1.9 秒

### 使用邢大框架回测 boll_inbar 策略

以下都是大家熟悉的课程代码，不多做解释

```python
def load_candles(candle_paths):
    return {itl: read_candle_feather(path) for itl, path in candle_paths.items()}

def backtest(candles, factor_params, leverage):
    df_xbx = STRA.factor(candles, factor_params)
    df_xbx = df_xbx[(df_xbx['candle_begin_time'] >= START_DATE) & (df_xbx['candle_begin_time'] <= END_DATE)]
    df_xbx = signal_simple_bolling(df_xbx)
    df_xbx['pos'] = df_xbx['signal'].shift().ffill().fillna(0)
    df_xbx = equity_curve_for_OKEx_USDT_future_next_open(
        df_xbx,
        slippage=0,
        c_rate=COMM_RATE,
        leverage_rate=leverage,
        face_value=FACE_VALUE,
        min_margin_ratio=LIQUI_RATE)
    return df_xbx

%time candles = load_candles(ETH_PATHS)
%time df_xbx = backtest(candles, factor_params, LEVERAGE)
print(df_xbx[['candle_begin_time', 'close', 'pos', 'equity_curve']])
```

运行结果为

```
CPU times: user 404 ms, sys: 545 ms, total: 949 ms
Wall time: 321 ms
CPU times: user 3.42 s, sys: 2.55 s, total: 5.97 s
Wall time: 5.41 s
          candle_begin_time    close  pos  equity_curve
275636  2018-03-01 00:00:00   853.00  0.0      1.000000
275637  2018-03-01 00:01:00   852.80  0.0      1.000000
275638  2018-03-01 00:02:00   853.01  0.0      1.000000
275639  2018-03-01 00:03:00   852.97  0.0      1.000000
275640  2018-03-01 00:04:00   851.00  0.0      1.000000
...                     ...      ...  ...           ...
2638630 2022-08-31 23:56:00  1553.80  1.0    242.547027
2638631 2022-08-31 23:57:00  1554.28  1.0    242.621999
2638632 2022-08-31 23:58:00  1555.01  1.0    242.736018
2638633 2022-08-31 23:59:00  1554.10  1.0    242.593885
2638634 2022-09-01 00:00:00  1553.04  1.0    242.282780

[2362999 rows x 4 columns]
```

邢大框架由于没有 jit 编译过程，因此初始化阶段快过 NEO 框架，然而回测阶段慢于 NEO 框架

> "Pandas 性能之耻" —— J神

### 正确性

使用以下代码分析相对误差

```python
err = df_ret['equity'] / df_xbx['equity_curve']  - 1
err.describe()
```

输出

```
count    2.362999e+06
mean     2.423074e-04
std      1.199977e-04
min     -1.066092e-03
25%      1.486942e-04
50%      2.772150e-04
75%      3.066479e-04
max      2.940089e-03
dtype: float64
```

相对误差在 -0.1% ~ 0.3% 之间，可以接受

## 参数网格搜索

与普通网格搜索不同，NEO 框架采用了一套自定的任务分片机制，预先将任务划分为 `n_proc` 片，单个进程只需要执行一个任务片，保证了 Backtester 可被重复利用，避免反复读入数据与 jit 编译

```python
def run_gridsearch(stra_module,
                   candle_paths,
                   contract_type,
                   simulator_params,
                   start_date,
                   end_date,
                   init_capital,
                   face_value,
                   n_proc=None):
    if n_proc is None:
        n_proc = max(os.cpu_count() - 1, 1)

    # 获取参数列表
    fparams_list = stra_module.get_default_factor_params_list()
    sparams_list = stra_module.get_default_strategy_params_list()

    # 任务分片，仅对因子参数分片
    fparams_seqs = []
    n = len(fparams_list)
    j = 0
    for i in range(n_proc):
        n_tasks = n // n_proc
        if i < n % n_proc:
            n_tasks += 1
        fparams_seqs.append(fparams_list[j:j + n_tasks])
        j += n_tasks

    # 对因子参数分片网格搜索
    def _search(fl):
        backtester = Backtester(candle_paths=candle_paths,
                                contract_type=contract_type,
                                simulator_params=simulator_params,
                                stra_module=stra_module)
        df = backtester.run_multi(start_date=start_date,
                                  end_date=end_date,
                                  init_capital=init_capital,
                                  face_value=face_value,
                                  fparams_list=fl,
                                  sparams_list=sparams_list)
        return df

    dfs = Parallel(n_jobs=n_proc)(delayed(_search)(fl) for fl in fparams_seqs)
    return pd.concat(dfs, ignore_index=True)
```

### 使用 NEO 框架网格搜索

```python
%%time

df_grid = run_gridsearch(stra_module=STRA, 
                         candle_paths=ETH_PATHS, 
                         contract_type='futures', 
                         simulator_params=SIMULATOR_PARAMS, 
                         start_date=START_DATE, 
                         end_date=END_DATE, 
                         init_capital=INIT_CAPITAL, 
                         face_value=FACE_VALUE, 
                         n_proc=16)

df_grid = df_grid.sort_values('equity', ascending=False, ignore_index=True)
print(df_grid.head())
```

输出

```
        equity  itl    n    b  leverage  face_value
0  2069.809571   1h   75  1.5       1.5       0.001
1  2049.436361  30m  150  1.5       1.5       0.001
2  1891.993335  30m  145  1.5       1.5       0.001
3  1839.323227   1h   70  1.5       1.5       0.001
4  1591.957325   1h  100  1.8       1.5       0.001
CPU times: user 67.3 ms, sys: 589 ms, total: 657 ms
Wall time: 3min 50s
```

### 使用邢大框架网格搜索

```python
%%time

fparams_list = STRA.get_default_factor_params_list()
sparams_list = STRA.get_default_strategy_params_list()

def _run(factor_params, strategy_params):
    leverage = strategy_params['leverage']
    candles = load_candles(ETH_PATHS)
    df_xbx = backtest(candles, factor_params, leverage)
    ret = {'equity': df_xbx['equity_curve'].iat[-1]}
    ret.update(factor_params)
    ret.update(strategy_params)
    return ret

results = Parallel(n_jobs=16)(delayed(_run)(fp, sp) for fp in fparams_list for sp in sparams_list)
df_grid_xbx = pd.DataFrame.from_records(results)
```

输出

```
CPU times: user 4.08 s, sys: 390 ms, total: 4.48 s
Wall time: 23min 4s
```

### 正确性

```python
df_grid_xbx = df_grid_xbx.sort_values('equity', ascending=False, ignore_index=True)
join_idx = ['itl', 'n', 'b', 'leverage']
tmp = df_grid_xbx.join(df_grid.set_index(join_idx), on=join_idx, rsuffix='_neo')

# 不比较归零策略
tmp = tmp[tmp['equity'] > 1]

(tmp['equity_neo'] / tmp['equity'] - 1).describe()
```

对于 1980 组参数中，1931 组赚钱的策略，最终权益相对误差仍在 -0.4% ~ 0.4% 之间

那些会让权益归零的参数，由于机制差距，且权益绝对值太小，难以分析误差，在此忽略

```
count    1931.000000
mean        0.000382
std         0.000753
min        -0.003282
25%        -0.000108
50%         0.000330
75%         0.000816
max         0.003698
dtype: float64
```

## 结论
本文提出了 Neo 框架 V1.1 升级，提出了一种新的策略编写范式，重构并升级了回测模块，并添加了网格搜索功能

在使用 16 进程并行网格搜索的场景下，邢大框架耗时 23分4秒 = 1384秒，NEO 框架耗时 3分50秒 = 230 秒，效率提升大约 5 倍
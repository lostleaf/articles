Numba-based Evaluation Online system, 基于 Numba 的在线回测系统，缩写为 NEO，取救世主尼奥之意

针对邢大离线回测框架无法支持仓位管理和 bar 内交易，以及 VNPY 在线回测框架回测速度缓慢且无法实时结算 PNL 的痛点而设计

# 引言

## 邢大框架和 VNPY 的特性和不足

在学习和研究趋势策略的过程中，邢大的趋势框架和 VNPY 是我使用最多的两个框架，但这两个框架又有很大的不同

邢大框架，使用向量化离线计算，可分为因子计算、交易信号生成、资金曲线生成三个模块。由于使用离线算法，每个模块均需要等待前一模块计算完毕之后才可进行计算。

这种做法很明显的具有速度快的优点——因为向量化相当于将循环语句放到了C语言层面

然而其缺点也同样明显：

1. 交易信号仅支持 1、0、-1 三种开平仓信号，使用固定杠杆率，无法决定具体开仓的合约张数，且一开一平相互对应，无法中途减仓
2. 如要实现止损功能，需要借助 Numba 循环才能保证效率
3. 只能在 K 线结束时刻调仓，无法实现 bar 内交易

相反地，VNPY 基于 python 面向对象设计，使用在线计算：

1. ArrayManager 类负责在线存储近期 K 线行情并计算因子，这种方式时间复杂度高且重复计算多，效率低下
2. 交易信号由用户继承 CtaTemplate 类自行编写策略子类生成，CtaTemplate 类可以与回测引擎 BacktestingEngine 交互，能获取当前持仓的合约张数，可以买卖任意张数的合约，并且可以下条件单执行 bar 内交易
3. 回测引擎 BacktestingEngine 根据 K 线实时模拟撮合交易，但是账户的 PNL 是回测结束后离线计算出的，回测过程中策略无法得知实时权益，因此除非自己计算权益，每次只能开仓固定张数合约，或根据初始权益计算下单张数，无法实现1倍杠杆满仓做多这种操作，即无法根据账户权益使用固定或动态杠杆率开仓

同时，VNPY 使用了 Python 原生循环来遍历 K 线，以支持在线计算，导致计算速度缓慢

以下表格是总结对比两个框架

|           | 邢大框架           | VNPY                             |
| --------- | ------------------ | -------------------------------- |
| 回测速度  | 快（1s以内）       | 慢（数分钟）                     |
| 仓位管理  | 固定杠杆，一开一平 | 可自由加减仓，但无法得知实时权益 |
| bar内交易 | 默认不支持         | 条件单可bar内成交                |

## NEO 系统设计与优点

针对邢大框架和 VNPY 的不足，本文提出一种新的回测范式与在线计算的回测框架，称之为 NEO 回测系统——基于 Numba jit 和 jitclass，仿照 VNPY，并吸取邢大框架的精髓，实现一种新的回测系统：

1. 使用向量化代码事先离线计算好每根 K 线对应的因子，保证因子计算速度
2. 采用 VNPY 的在线回测模式，同时生成交易信号和账户权益，交易信号由目标持仓的合约张数表示，账户权益则由账户 USDT 净权益表示，持仓可由策略任意调整
3. 提出一种1分钟线与大周期 K 线混合回测的模式，可近似实现 bar 内交易，例如对于1小时线，除了可以在整小时处调仓外，也可在小时线内其他 59 个整分钟调仓；这种混合周期的 bar 内交易，邢大框架也可以实现

NEO 系统当前的目的是，对于单标的趋势策略，实现一个类似于 [赵老板的中性回放仿盘](https://bbs.quantclass.cn/thread/13767) 的回测系统，但是效率要达到邢大框架级别

*后续可能的方向：对在线框架横截面扩展，1)将赵老板的回放仿盘效率优化至邢大框架级别，2)同一框架下高效实现趋势、中性、时间策略的仿盘回测*

# 内容与代码结构

本文包含以下内容：

1. 实现一个邢大课程中的1小时线布林带策略，讲解 NEO 系统的核心代码实现，以及策略编写和回测范式，并与邢大框架进行对比，验证其效率和正确性 （✓）
2. 对 1 中的策略，使用 1 分钟线混合回测，在邢大框架和 NEO 系统上分别实现，验证其效率和正确性（✓）
3. 对 2 中的策略，添加仓位管理，这个例子无法在邢大框架实现，将体现 NEO 系统的优越性（✓）

本文主要包含以下代码

```bash
├── case1_boll_online.ipynb        在线化布林策略回测调用代码
├── case2_boll_inbar.ipynb         混合周期bar内交易回测调用代码
├── case3_boll_pos_manage.ipynb    带仓位管理的混合周期回测调用代码
├── neo_backtesting                NEO 回测框架 package
│   ├── backtest.py                    回测辅助函数
│   └── simulator.py                   资金曲线模拟器
├── strategy_example               策略实现 package
│   ├── bolling.py                     策略代码
│   └── factor.py                      因子代码
└── xbx                            从邢大课程抄来的代码
    ├── backtest.py                    资金曲线
    └── strategy.py                    交易开平仓信号
```

# Case1: 布林带策略在邢大框架和 NEO 系统中的实现

我们仍然将布林带策略分解为 3 个模块进行讲解：因子计算、信号生成、资金曲线生成

本小节中所有的回测都基于以下参数，对应代码为 case1_boll_online.ipynb

```python
# case1_boll_online.ipynb

# 回测起始日
START_DATE = '20180301'

# 布林带周期与宽度
N = 100
B = 2

# 回测参数
INIT_CAPITAL = 1e5  # 初始资金，10 万美元
LEVERAGE = 1  # 使用 1 倍杠杆
FACE_VALUE = 0.01  # 合约面值 0.01 ETH
COMM_RATE = 6e-4  # 交易成本万分之 6
LIQUI_RATE = 5e-3  # 爆仓保证金率千分之 5
```

## 因子计算

因子计算部分，NEO 系统与邢大框架完全一致，均采用向量化离线计算的方式提前算好

```python
# strategy_example.factor

def boll_factor(df: pd.DataFrame, n, b):
    """
    单周期的布林带因子
    """
    # 计算均线
    df['median'] = df['close'].rolling(n, min_periods=1).mean()

    # 计算标准差
    std = df['close'].rolling(n, min_periods=1).std(ddof=0)  # ddof代表标准差自由度

    # 计算上轨、下轨道
    df['upper'] = df['median'] + b * std
    df['lower'] = df['median'] - b * std

    return df
```

以上代码邢大课程里面都讲过，我们非常熟悉

接下来，我们读入历史行情，计算因子，并分别为邢大框架和 NEO 系统生成 DataFrame: `df_xbx` 和 `df_online`

```
# case1_boll_online.ipynb

df = pd.read_feather('ETH-USDT_1h.fea')
df = boll_factor(df, 100, 2)

df = df[df['candle_begin_time'] >= START_DATE].reset_index(drop=True)


df_xbx = df.copy()
df_online = df.copy()
```

## 信号生成

### 邢大框架

邢大课程的布林带策略信号生成，放在 `xbx.strategy.signal_simple_bolling` 函数中，与课程代码完全一致，此处不予赘述

调用函数，根据因子生成开平仓信号：

```python
# case1_boll_online.ipynb

df_xbx = signal_simple_bolling(df_xbx)

df_xbx['pos'] = df_xbx['signal'].shift().ffill().fillna(0)
```

此时，df_xbx 中 pos 字段为 K 线开始时刻的目标仓位

### NEO 系统

NEO 系统中，交易信号由策略生成，策略包含最重要的开平仓逻辑，基于 numba jitclass 实现，需要遵守这几个原则：

1. 对策略类使用`@jitclass` 装饰器，需要在装饰器参数中，声明每个成员变量的类型（类似编译型语言）
2. 在`__init__` 函数中，接收必要的初始化参数，对每一个成员变量初始化
3. `on_bar` 函数为每次收到 K 线时的回调函数，是策略对外调用的接口，固定接收 4 个参数：`(candle, factors, pos, equity)`，固定返回一个整数`target_pos`，表示当前期望持有的合约张数（净持仓），正数代表做多，负数代表做空，0代表平仓，将会在下根k线开始时调至目标仓位

`on_bar` 函数中，四个参数的定义分别为

1. candle: K 线，5 维向量，类型为`np.ndarray(5, 'float')`，分别代表 open, high, low, close, volume
2. factors: 因子，n维向量，类型为`np.ndarray(n, 'float')`，有多少个因子就有多少维
3. pos：当前持仓，整数`int64`，目前净持仓
4. equity：账户权益，浮点数`float64`，账户净权益，单位为 USDT

NEO 系统中，由于 numba jitclass 不支持继承，因此策略实现使用 duck-typing 模式，任何满足以上规范的 jitclass 均可被接受

策略实现如下

```

# strategy_example.bolling

@jitclass([
    ['leverage', nb.float64],  # 杠杆率 
    ['face_value', nb.float64],  # 合约面值
    ['prev_upper', nb.float64],  # 上根k线上轨
    ['prev_lower', nb.float64],  # 上根k线下轨
    ['prev_median', nb.float64],  # 上根k线均线
    ['prev_close', nb.float64]  # 上根k线收盘价
])
class BollingStrategy:

    def __init__(self, leverage, face_value):
        self.leverage = leverage
        self.face_value = face_value

        self.prev_upper = np.nan
        self.prev_lower = np.nan
        self.prev_median = np.nan
        self.prev_close = np.nan

    def on_bar(self, candle, factors, pos, equity):
        op, hi, lo, cl, vol = candle
        upper, lower, median = factors

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
```

## **资金曲线生成**

### 邢大框架

函数 `xbx.backtest.equity_curve_for_OKEx_USDT_future_next_open` 为邢大框架中生成资金曲线的代码，同样为课程代码

调用之，即可生成资金曲线完成回测：

```python
# case1_boll_online.ipynb
df_xbx = equity_curve_for_OKEx_USDT_future_next_open(
    df_xbx  , 
    slippage=0,
    c_rate=COMM_RATE,
    leverage_rate=LEVERAGE,
    face_value=FACE_VALUE,
    min_margin_ratio=LIQUI_RATE)
```

运行结束后，在 df_xbx 中添加一列 equity_curve，代表净值

将首尾几行打印出来，则有

```python
# case1_boll_online.ipynb
print(df_xbx[['candle_begin_time', 'pos', 'equity_curve']].head().to_markdown())
print(df_xbx[['candle_begin_time', 'pos', 'equity_curve']].tail().to_markdown())
```

|      | candle_begin_time   |  pos | equity_curve |
| ---: | :------------------ | ---: | -----------: |
|    0 | 2018-03-01 00:00:00 |    0 |            1 |
|    1 | 2018-03-01 01:00:00 |    0 |            1 |
|    2 | 2018-03-01 02:00:00 |    0 |            1 |
|    3 | 2018-03-01 03:00:00 |    0 |            1 |
|    4 | 2018-03-01 04:00:00 |    0 |            1 |

|       | candle_begin_time   |  pos | equity_curve |
| ----: | :------------------ | ---: | -----------: |
| 37527 | 2022-06-13 19:00:00 |   -1 |      68.5849 |
| 37528 | 2022-06-13 20:00:00 |   -1 |       68.152 |
| 37529 | 2022-06-13 21:00:00 |   -1 |      68.5557 |
| 37530 | 2022-06-13 22:00:00 |   -1 |      69.7119 |
| 37531 | 2022-06-13 23:00:00 |   -1 |      69.1675 |

### NEO 系统

NEO 系统中，模拟调仓和资金曲线生成由模拟器负责，模拟器同样用 numba jitclass 实现，记录了账户当前持仓和权益，并根据 k线更新账户权益，检查是否爆仓等

```python
# neo_backtesting.simulator

@jitclass([
    ['equity', nb.float64],  # 账户权益，可以设置为 10万美元
    ['face_value', nb.float64],  # 合约面值，或币安最小下单单位
    ['comm_rate', nb.float64],  # 手续费/交易成本，可设置为万分之6
    ['liqui_rate', nb.float64],  # 爆仓保证金率，千分之5
    ['pre_close', nb.float64],  # 上根k线的收盘价
    ['pos', nb.int64],  # 当前持仓
    ['target_pos', nb.int64]  # 目标持仓
])
class UsdtFutureSimulator:

    def __init__(self, init_capital, face_value, comm_rate, liqui_rate, init_pos=0):
        self.equity = init_capital
        self.face_value = face_value
        self.comm_rate = comm_rate
        self.liqui_rate = liqui_rate

        self.pre_close = np.nan
        self.pos = init_pos
        self.target_pos = init_pos

    def adjust_pos(self, target_pos):
        self.target_pos = target_pos

    def simulate_bar(self, candle):
        op, hi, lo, cl, vol = candle
        if np.isnan(self.pre_close):
            self.pre_close = op

        # K 线开盘时刻
        # 根据开盘价和前收盘价，结算当前账户权益
        self.equity += (op - self.pre_close) * self.face_value * self.pos

        # 当前需要调仓
        if self.target_pos != self.pos:
            delta = self.target_pos - self.pos  # 需要买入或卖出的合约数量
            self.equity -= abs(delta) * self.face_value * op * self.comm_rate  # 扣除手续费
            self.pos = self.target_pos  # 更新持仓

        # K 线当中
        price_min = lo if self.pos > 0 else hi  # 根据持仓方向，找出账户权益最低的价格
        equity_min = self.equity + (price_min - op) * self.pos * self.face_value  #  最低账户权益
        if self.pos == 0:
            margin_ratio_min = 1e8  # 空仓，设置保证金率为极大值
        else:
            margin_ratio_min = equity_min / (self.face_value * abs(self.pos) * price_min)  # 有持仓，计算最低保证金率
        if margin_ratio_min < self.liqui_rate + self.comm_rate:  # 爆仓
            self.equity = 1e-8  # 设置这一刻起的资金为极小值，防止除零错误
            self.pos = 0

        # K线收盘时刻
        self.equity += (cl - op) * self.face_value * self.pos  # 根据收盘价，结算账户权益
        self.pre_close = cl

```

并基于以下回测辅助函数实现回测

```
# neo_backtesting.backtest

@nb.njit
def _run_backtest(candles, factor_mat, simu, stra):
    n = candles.shape[0]  # k 线数量
    pos_open = np.empty(n, dtype=np.int64)  # k 线 open 时刻的目标仓位，当根 K 线始终持有该仓位
    equity = np.empty(n, dtype=np.float64)  # k 线 close 时刻的账户权益

    # 遍历每根 k 线循环
    for i in range(n):
        simu.simulate_bar(candles[i])  # 模拟调仓和 k 线内权益结算
        equity[i] = simu.equity  # 记录权益
        pos_open[i] = simu.pos  # 记录仓位
        target_pos = stra.on_bar(candles[i], factor_mat[i], simu.pos, simu.equity)  # 策略生成目标持仓
        simu.adjust_pos(target_pos)  # 记录目标持仓，下根 k 线 open 时刻调仓
    return pos_open, equity


def backtest_online(
        df: pd.DataFrame,  # 包含 k 线和因子的 dataframe
        simulator: UsdtFutureSimulator,  # 模拟器，目前只支持 UsdtFutureSimulator
        strategy: Any,  # 策略
        factor_columns: List[str]  # 因子的名称
):
    # 将 OHLCV K线转化为 Numpy 矩阵
    candles = df[['open', 'high', 'low', 'close', 'volume']].to_numpy()

    # 将因子转化为 Numpy 矩阵
    factor_mat = df[factor_columns].to_numpy()

    # 运行 jit 回测函数，获得仓位和权益
    pos, equity = _run_backtest(candles, factor_mat, simulator, strategy)
    df['pos'] = pos
    df['equity'] = equity
```

基于以下代码调用：

```
# case1_boll_online.ipynb

simulator = UsdtFutureSimulator(
    init_capital=INIT_CAPITAL, 
    face_value=FACE_VALUE, 
    comm_rate=COMM_RATE, 
    liqui_rate=LIQUI_RATE, 
    init_pos=0)

strategy = BollingStrategy(
    leverage=LEVERAGE, 
    face_value=FACE_VALUE)

backtest_online(
    df, 
    simulator=simulator,
    strategy=strategy,
    factor_columns = ['upper', 'lower', 'median'])
```

以下为回测结果

```python
# case1_boll_online.ipynb

df['equity'] /= INIT_CAPITAL

print(df[['candle_begin_time', 'pos', 'equity']].head().to_markdown(), '\n')
print(df[['candle_begin_time', 'pos', 'equity']].tail().to_markdown(), '\n')
```

|      | candle_begin_time   |  pos | equity |
| ---: | :------------------ | ---: | -----: |
|    0 | 2018-03-01 00:00:00 |    0 |      1 |
|    1 | 2018-03-01 01:00:00 |    0 |      1 |
|    2 | 2018-03-01 02:00:00 |    0 |      1 |
|    3 | 2018-03-01 03:00:00 |    0 |      1 |
|    4 | 2018-03-01 04:00:00 |    0 |      1 |

|       | candle_begin_time   |     pos |  equity |
| ----: | :------------------ | ------: | ------: |
| 37527 | 2022-06-13 19:00:00 | -321335 |   68.59 |
| 37528 | 2022-06-13 20:00:00 | -321335 | 68.1568 |
| 37529 | 2022-06-13 21:00:00 | -321335 | 68.5607 |
| 37530 | 2022-06-13 22:00:00 | -321335 | 69.7178 |
| 37531 | 2022-06-13 23:00:00 | -321335 | 69.1963 |

其中，pos 字段与邢大不同，为理论上回测结束时的真实持仓（做空 3213.35 ETH）

## 正确性

我们打印出 NEO 系统回测资金曲线与邢大框架资金曲线相对误差，统计相对误差取值范围：

```python
print((df['equity'] / df_xbx['equity_curve'] - 1).describe())
```

```
count    37532.000000
mean         0.000539
std          0.000411
min         -0.000478
25%          0.000328
50%          0.000421
75%          0.000514
max          0.002426
dtype: float64
```

与邢大框架基本一致，最大相对误差为 0.24%

## 效率

粗略地对比邢大框架和 NEO 系统运行耗时（case1_boll_online.ipynb相关代码）

|                         | 信号生成 | 资金曲线生成 | 总和    |
| ----------------------- | -------- | ------------ | ------- |
| 邢大框架                | 7.38 ms  | 23.1 ms      | 30.5 ms |
| NEO 系统（未 jit 编译） | -        | 606 ms       | 606 ms  |
| NEO 系统（已 jit 编译） | -        | 16.9 ms      | 16.9 ms |

NEO 系统由于使用了 numba，第一次调用时回触发 jit 编译，耗时 600 多毫秒，如果是第二次及以后运行已经 jit 编译好的代码，则耗时约 17 毫秒，反而快于邢大框架

# Case2: 基于 1 分钟线的混合周期 bar 内交易布林策略

当我们实现一个通道突破策略的时候，很有可能突破的时间点并不是发生在小时线结束的时刻，如果我们在突破后能尽快的追进去，或许可以提高收益

我们将 1 小时线上的布林通道扩展到 1 分钟线上：首先在 1 小时线上计算布林带上中下轨，然后将小时线上的布林通道扩展到每个小时结束后的 59 个非整小时分钟；那么，假如 `12:34:56.789` 这个时刻，突破了上一小时，即 `candle_end_time==12:00:00` 的 1 小时 K 线所计算出来的布林带上下轨，那么我们理论上可以在 `12点35分` 追进去开仓，而不用等到 `13点` 才开仓

## 因子计算

实现这个策略非常简单，只需要在计算因子时将大小周期 DataFrame 相互 join 即可，主要要根据 `candle_end_time` join，否则会引入非常经典的未来函数

```python
# strategy_example.factor

def boll_factor_cross_timeframe(df_1m: pd.DataFrame, df_1h: pd.DataFrame, n, b):
    """
    跨周期的布林带因子
    先在1小时线（大周期）上计算布林带，然后填充到1分钟（小周期）上
    每根1分钟线均使用最近的已闭合的1小时线上中下轨
    """
    df_1h = boll_factor(df_1h, n, b)

    # 根据 k 线结束时间对齐
    df_1h['candle_end_time'] = df_1h['candle_begin_time'] + pd.Timedelta(hours=1)
    df_1m['candle_end_time'] = df_1m['candle_begin_time'] + pd.Timedelta(minutes=1)

    factor_cols = ['upper', 'lower', 'median']
    df = df_1m.join(df_1h.set_index('candle_end_time')[factor_cols], on='candle_end_time')

    # 向后填充分钟线上中下轨
    for col in factor_cols:
        df[col].ffill(inplace=True)

    return df
```

调用以下代码计算因子

```python
# case2_boll_inbar.ipynb

df_1h = pd.read_feather('ETH-USDT_1h.fea')
df_1m = pd.read_feather('ETH-USDT_1m.fea')

df = boll_factor_cross_timeframe(df_1m, df_1h, N, B)

df = df[df['candle_end_time'] >= START_DATE].reset_index(drop=True)

df_xbx = df.copy()
df_online = df.copy()
```

打印出计算结果

```python
# case2_boll_inbar.ipynb

tmp = df[['candle_end_time', 'close', 'upper', 'lower', 'median']]

print(tmp.iloc[58:62].to_markdown(), '\n')
print(tmp.iloc[-63:-58].to_markdown(), '\n')
```

|      | candle_end_time     |  close |   upper |   lower |  median |
| ---: | :------------------ | -----: | ------: | ------: | ------: |
|   58 | 2018-03-01 00:58:00 | 856.24 | 898.871 | 818.384 | 858.627 |
|   59 | 2018-03-01 00:59:00 |  857.7 | 898.871 | 818.384 | 858.627 |
|   60 | 2018-03-01 01:00:00 |    858 | 898.089 | 820.153 | 859.121 |
|   61 | 2018-03-01 01:01:00 | 857.32 | 898.089 | 820.153 | 859.121 |

|         | candle_end_time     |   close |   upper |   lower |  median |
| ------: | :------------------ | ------: | ------: | ------: | ------: |
| 2249210 | 2022-06-13 22:58:00 | 1198.35 | 1678.06 | 1201.53 |  1439.8 |
| 2249211 | 2022-06-13 22:59:00 | 1193.89 | 1678.06 | 1201.53 |  1439.8 |
| 2249212 | 2022-06-13 23:00:00 | 1193.59 | 1673.77 | 1196.31 | 1435.04 |
| 2249213 | 2022-06-13 23:01:00 | 1194.02 | 1673.77 | 1196.31 | 1435.04 |
| 2249214 | 2022-06-13 23:02:00 | 1193.99 | 1673.77 | 1196.31 | 1435.04 |

可以看到，非整小时的 1 分钟线上，布林通道均与前一分钟相同，整小时时刻发生变化

除了因子计算，其他部分不需要修改

## 运行回测

### 邢大框架

基于以下两段代码执行邢大框架回测

生成信号

```python
# case2_boll_inbar.ipynb

df_xbx = signal_simple_bolling(df_xbx)

df_xbx['pos'] = df_xbx['signal'].shift().ffill().fillna(0)
```

生成资金曲线

```python
# case2_boll_inbar.ipynb

df_xbx = equity_curve_for_OKEx_USDT_future_next_open(
    df_xbx, 
    slippage=0,
    c_rate=COMM_RATE,
    leverage_rate=LEVERAGE,
    face_value=FACE_VALUE,
    min_margin_ratio=LIQUI_RATE)
```

回测结果

```python
# case2_boll_inbar.ipynb

df['equity'] /= INIT_CAPITAL

print(df[['candle_begin_time', 'pos', 'equity']].head().to_markdown(), '\n')
print(df[['candle_begin_time', 'pos', 'equity']].tail().to_markdown(), '\n')
```

|      | candle_begin_time   |  pos | equity_curve |
| ---: | :------------------ | ---: | -----------: |
|    0 | 2018-02-28 23:59:00 |    0 |            1 |
|    1 | 2018-03-01 00:00:00 |    0 |            1 |
|    2 | 2018-03-01 00:01:00 |    0 |            1 |
|    3 | 2018-03-01 00:02:00 |    0 |            1 |
|    4 | 2018-03-01 00:03:00 |    0 |            1 |

|         | candle_begin_time   |  pos | equity_curve |
| ------: | :------------------ | ---: | -----------: |
| 2249268 | 2022-06-13 23:55:00 |   -1 |      115.938 |
| 2249269 | 2022-06-13 23:56:00 |   -1 |      115.898 |
| 2249270 | 2022-06-13 23:57:00 |   -1 |      115.956 |
| 2249271 | 2022-06-13 23:58:00 |   -1 |      115.949 |
| 2249272 | 2022-06-13 23:59:00 |   -1 |      115.718 |

## NEO 系统

基于以下代码同时生成信号和资金曲线

```python
#case2_boll_inbar.ipynb

simulator = UsdtFutureSimulator(
    init_capital=INIT_CAPITAL, 
    face_value=FACE_VALUE, 
    comm_rate=COMM_RATE, 
    liqui_rate=LIQUI_RATE, 
    init_pos=0)

strategy = BollingStrategy(leverage=LEVERAGE, face_value=FACE_VALUE)
backtest_online(
    df, 
    simulator=simulator,
    strategy=strategy,
    factor_columns = ['upper', 'lower', 'median'])
```

回测结果

```python
# case2_boll_inbar.ipynb

df['equity'] /= INIT_CAPITAL

print(df[['candle_begin_time', 'pos', 'equity']].head().to_markdown(), '\n')
print(df[['candle_begin_time', 'pos', 'equity']].tail().to_markdown(), '\n')
```

|      | candle_begin_time   |  pos | equity |
| ---: | :------------------ | ---: | -----: |
|    0 | 2018-02-28 23:59:00 |    0 |      1 |
|    1 | 2018-03-01 00:00:00 |    0 |      1 |
|    2 | 2018-03-01 00:01:00 |    0 |      1 |
|    3 | 2018-03-01 00:02:00 |    0 |      1 |
|    4 | 2018-03-01 00:03:00 |    0 |      1 |

|         | candle_begin_time   |     pos |  equity |
| ------: | :------------------ | ------: | ------: |
| 2249268 | 2022-06-13 23:55:00 | -516518 | 115.994 |
| 2249269 | 2022-06-13 23:56:00 | -516518 | 115.954 |
| 2249270 | 2022-06-13 23:57:00 | -516518 | 116.012 |
| 2249271 | 2022-06-13 23:58:00 | -516518 | 116.005 |
| 2249272 | 2022-06-13 23:59:00 | -516518 | 115.811 |

## 正确性

对比和邢大框架的相对误差

```python
# case2_boll_inbar.ipynb

print((df['equity'] / df_xbx['equity_curve'] - 1).describe())
```

```bash
count    2.249273e+06
mean     1.680461e-04
std      2.075052e-04
min     -1.317970e-03
25%      1.464291e-05
50%      1.331632e-04
75%      2.733756e-04
max      2.786497e-03
dtype: float64
```

可以看到最大相对误差为 0.28%

## 效率

|                         | 信号生成 | 资金曲线生成 | 总和   |
| ----------------------- | -------- | ------------ | ------ |
| 邢大框架                | 387 ms   | 578 ms       | 965 ms |
| NEO 系统（未 jit 编译） | -        | 718 ms       | 718 ms |
| NEO 系统（已 jit 编译） | -        | 206 ms       | 206 ms |

可以看到，随着数据量的扩大，即使触发 jit 编译，耗时也小于邢大框架，如果不触发 jit 编译，则大幅度快于邢大框架

# Case3: 带仓位管理的 1 分钟线混合周期布林策略

对于上一节描述的布林策略，考虑以下事实：

1. 有时候布林带很宽，开仓点位离均线很远，假如触发均线止损，亏损会非常巨大，因此开仓点位离均线很远时，应适当降低杠杆，相反离得近时可适当增加杠杆，即通过调整开仓杠杆，归一化开仓风险，但杠杆但不超过某个比例（邢大：最大杠杆不要超过 3 倍）
2. 均线比较迟钝，加入移动止损保护收益，但我们移动止损不完全平仓只按比例减仓

## 策略

则有以下策略

```python
@jitclass([
    ['max_leverage', nb.float64],  # 最大杠杆率 
    ['max_loss', nb.float64],  # 开仓最大亏损
    ['in_stop', nb.boolean],  # 是否在移动止损过程
    ['stop_pct', nb.float64],  # 移动止损比例
    ['stop_close_pct', nb.float64],  # 移动止损平仓比例
    ['face_value', nb.float64],  # 合约面值
    ['highest', nb.float64],  # 开仓后最高价
    ['lowest', nb.float64],  # 开仓后最低价
    ['prev_upper', nb.float64],  # 上根k线上轨
    ['prev_lower', nb.float64],  # 上根k线下轨
    ['prev_median', nb.float64],  # 上根k线均线
    ['prev_close', nb.float64]  # 上根k线收盘价
])
class BollingPosMgtStrategy:

    def __init__(self, max_leverage, max_loss, face_value, stop_pct, stop_close_pct):
        self.max_leverage = max_leverage
        self.max_loss = max_loss
        self.stop_pct = stop_pct
        self.stop_close_pct = stop_close_pct
        self.face_value = face_value

        self.in_stop = False
        self.highest = np.nan
        self.lowest = np.nan

        self.prev_upper = np.nan
        self.prev_lower = np.nan
        self.prev_median = np.nan
        self.prev_close = np.nan

    # 重置止损高低价
    def reset_stop(self, price):
        self.in_stop = True
        self.highest = price
        self.lowest = price

    # 更新止损高低价
    def update_stop_hl(self, price):
        if not self.in_stop:
            return
        self.highest = max(price, self.highest)
        self.lowest = min(price, self.lowest)

    def on_bar(self, candle, factors, pos, equity):
        op, hi, lo, cl, vol = candle
        upper, lower, median = factors

        # 默认保持原有仓位
        target_pos = pos

        # 移动止损中，更新高低价
        self.update_stop_hl(cl)

        if not np.isnan(self.prev_close):
            # 先判断开仓
            # 计算本次使用的杠杆
            risk = abs(cl / median - 1) + 1e-8
            leverage = min(self.max_loss / risk, self.max_leverage)

            # 做空或无仓位，上穿上轨，做多
            if pos <= 0 and cl > upper and self.prev_close <= self.prev_upper:
                target_pos = int(equity * leverage / cl / self.face_value)
                # 用当前价重置止损
                self.reset_stop(cl)

            # 做多或无仓位，下穿下轨，做空
            elif pos >= 0 and cl < lower and self.prev_close >= self.prev_lower:
                target_pos = -int(equity * leverage / cl / self.face_value)
                # 用当前价重置止损
                self.reset_stop(cl)

            # 目前持有做多仓位
            elif pos > 0:
                # 下穿中轨，平仓
                if cl < median and self.prev_close >= self.prev_median:
                    target_pos = 0

                # 移动止损过程中，跌了超过 self.stop_pct
                elif self.in_stop and (1 - cl / self.highest) > self.stop_pct:
                    # 平 self.stop_close_pct 的仓位
                    target_pos = int(pos * (1 - self.stop_close_pct))
                    self.reset_stop(cl)

            # 目前持有做空仓位
            elif pos < 0:
                #上穿中轨，平仓
                if cl > median and self.prev_close <= self.prev_median:
                    target_pos = 0

                # 移动止损过程中，涨了超过 self.stop_pct
                elif self.in_stop and (cl / self.lowest - 1) > self.stop_pct:
                    # 平 self.stop_close_pct 的仓位
                    target_pos = int(pos * (1 - self.stop_close_pct))
                    self.reset_stop(cl)

        # 更新上根K线数据
        self.prev_upper = upper
        self.prev_lower = lower
        self.prev_close = cl
        self.prev_median = median

        return target_pos

```

其他代码不需要更改

## 运行回测

计算因子

```python
# case3_boll_pos_manage.ipynb

df_1h = pd.read_feather('ETH-USDT_1h.fea')
df_1m = pd.read_feather('ETH-USDT_1m.fea')

df = boll_factor_cross_timeframe(df_1m, df_1h, N, B)

df = df[df['candle_end_time'] >= START_DATE].reset_index(drop=True)
```

执行回测

```python
# case3_boll_pos_manage.ipynb

simulator = UsdtFutureSimulator(
    init_capital=INIT_CAPITAL, 
    face_value=FACE_VALUE, 
    comm_rate=COMM_RATE, 
    liqui_rate=LIQUI_RATE, 
    init_pos=0)

strategy = BollingPosMgtStrategy(
    max_leverage=1.5, 
    max_loss=0.05, 
    face_value=FACE_VALUE, 
    stop_pct=0.05, 
    stop_close_pct=0.2)

backtest_online(
    df, 
    simulator=simulator,
    strategy=strategy,
    factor_columns = ['upper', 'lower', 'median'])
```

以上代码中的参数为随手设置，本帖主要关心回测框架而不是策略实现

回测结果

```
# case3_boll_pos_manage.ipynb

df['equity'] /= INIT_CAPITAL

print(df[['candle_begin_time', 'pos', 'equity']].head().to_markdown(), '\n')
print(df[['candle_begin_time', 'pos', 'equity']].tail().to_markdown(), '\n')
```

|      | candle_begin_time   |  pos | equity |
| ---: | :------------------ | ---: | -----: |
|    0 | 2018-02-28 23:59:00 |    0 |      1 |
|    1 | 2018-03-01 00:00:00 |    0 |      1 |
|    2 | 2018-03-01 00:01:00 |    0 |      1 |
|    3 | 2018-03-01 00:02:00 |    0 |      1 |
|    4 | 2018-03-01 00:03:00 |    0 |      1 |

|         | candle_begin_time   |     pos |  equity |
| ------: | :------------------ | ------: | ------: |
| 2249268 | 2022-06-13 23:55:00 | -714816 | 522.373 |
| 2249269 | 2022-06-13 23:56:00 | -714816 | 522.318 |
| 2249270 | 2022-06-13 23:57:00 | -714816 | 522.399 |
| 2249271 | 2022-06-13 23:58:00 | -714816 | 522.389 |
| 2249272 | 2022-06-13 23:59:00 | -714816 | 522.121 |

观察最后一笔交易过程中的仓位变化

```python
# case3_boll_pos_manage.ipynb

mask = df['pos'].diff().fillna(0) != 0
df['value'] = df['pos'].abs() * FACE_VALUE * df['close']
print(df[mask][['candle_end_time', 'open', 'close', 'pos', 'value', 'equity']].tail(8).to_markdown())
```

|         | candle_end_time     |    open |   close |      pos |       value |  equity |
| ------: | :------------------ | ------: | ------: | -------: | ----------: | ------: |
| 2239184 | 2022-06-07 00:26:00 |  1809.5 | 1803.26 |        0 |           0 | 409.232 |
| 2244281 | 2022-06-10 13:23:00 | 1726.51 | 1722.75 | -2726810 | 4.69761e+07 | 409.975 |
| 2247267 | 2022-06-12 14:54:00 | 1507.35 | 1510.87 | -2181448 | 3.29588e+07 | 467.893 |
| 2248078 | 2022-06-13 04:11:00 |  1372.2 | 1374.53 | -1745158 | 2.39877e+07 |   497.7 |
| 2248388 | 2022-06-13 09:16:00 |  1245.9 | 1239.73 | -1396126 | 1.73082e+07 | 520.984 |
| 2248566 | 2022-06-13 12:14:00 |  1248.7 | 1245.01 | -1116900 | 1.39055e+07 | 520.123 |
| 2248777 | 2022-06-13 15:45:00 | 1240.47 | 1242.96 |  -893520 | 1.11061e+07 | 520.391 |
| 2249064 | 2022-06-13 20:32:00 | 1277.95 | 1276.76 |  -714816 | 9.12648e+06 | 517.336 |

可以看到，在暴跌的过程中，每次反弹都平掉一部分仓位，兑现利润，随着行情发展，持有的仓位和名义价值越来越低，因此即使在 1200 左右强烈超跌反弹，也不会回吐大量浮盈

## 效率

邢大框架根本无法实现，NEO 系统依然高效

|                         | 资金曲线生成 | 总和         |
| ----------------------- | ------------ | ------------ |
| 邢大框架                | -            | （无法实现） |
| NEO 系统（未 jit 编译） | 894 ms       | 894 ms       |
| NEO 系统（已 jit 编译） | 214 ms       | 214 ms       |

# 总结

本文提出了 NEO 趋势回测系统，一种基于 Numba jit 和 jitclass 实现的在线单合约趋势回测框架及其策略编程范式，允许实现仓位管理和基于 1 分钟线的 bar 内交易，同时保持和邢大框架同样高效回测效率

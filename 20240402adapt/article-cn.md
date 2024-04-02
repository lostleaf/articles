# 自适应布林标准化：Z-Score、Max-Abs 标准化与J神自适应布林

J神自适应布林的详细讨论可参考我的帖子：[「lostleaf」带bias优化的自适应布林+移动止损 | BTC: 5.84（达标）、ETH: 18.13（达标）、参数: 3](https://bbs.quantclass.cn/thread/1724)。

本文将通过数学变换，进一步挖掘自适应策略的潜力。

## Z-Score 标准化

在统计学中，Z-Score 表示随机变量与其均值之间的**差值**，除以标准差的倍数。

在自然界中，由于正态分布的 `68–95–99.7法则`，绝大多数情况下，Z-Score 的取值位于 -2 到 2 之间。因此，布林带的默认宽度倍数定为 2。

在正态分布中，Z-Score 大于 2 或小于 -2 的情况通常表明异常性。例如：
- J神的身高为一米九，则可以说 “J神身高的 Z-Score 大于 2”，意味着 J神的身高高于 97.5% 的中国男性（97.5% = 95% + (1 - 95%) / 2）。
- 如果某标的的价格突破了 2 倍布林带，即其滚动 Z-Score 大于 2，则可以认为该标的呈现出上涨趋势。

以下是 Z-Score 标准化的代码实现：

``` python
def zscore_normalize(df: pd.DataFrame, indicator: str, n: int):
    df['median'] = df[indicator].rolling(n, min_periods=1).mean()
    df['std'] = df[indicator].rolling(n, min_periods=1).std(ddof=0)
    df[f'{indicator}_zscore'] = (df[indicator] - df['median']) / df['std']
```

通过 Z-Score 标准化，`indicator_zscore` 在时间序列上将表现出一定的稳定性：其均值大致为 0，取值范围从负无穷到正无穷，但通常位于 [-2, 2] 范围内。

因此，布林带的默认开多仓条件可替换为：

```python
df['close_zscore'] > 2
```

通过对常数 `2` 进行 J 神自适应处理，可得到：

```python
df['close_zscore'] > df['close_zscore'].abs().rolling(n, min_periods=1).max().shift(1)
```

即：

```python
df['close_zscore'] / df['close_zscore'].abs().rolling(n, min_periods=1).max().shift(1) > 1
```

## Max-Abs 标准化

对于 `df['close_zscore'] / df['close_zscore'].abs().rolling(n, min_periods=1).max().shift(1)`，我将其命名为 Max-Abs 标准化，这可以视作经典 Min-Max 标准化的一个变种：

```python
def max_abs_normalize(df: pd.DataFrame, indicator: str, n: int):
    df['max_abs'] = df[indicator].abs().rolling(n, min_periods=1).max()
    df[f'{indicator}_max_abs_norm'] = df[indicator] / df['max_abs']
```

经 Max-Abs 标准化处理后，随机变量的取值将被限制在 `[-1, 1]` 的范围内。

## 自适应布林标准化

将 Z-Score 标准化与 Max-Abs 标准化结合起来，我将其称为自适应布林标准化。这一方法的原始创意源自 J神自适应布林：

```python
def adapt_bolling_normalize(df: pd.DataFrame, indicator: str, n: int):
    zscore_normalize(df, indicator, n)

    # 使用 n + 1 以保证与 J神方法的一致性
    max_abs_normalize(df, f'{indicator}_zscore', n + 1)

    df.drop(columns=['median', 'std', f'{indicator}_zscore', 'max_abs'], inplace=True)
```

任何时间序列，经过自适应布林标准化后，均会表现出一定的平稳性，并拥有固定的 `[-1, 1]` 取值范围。因此，我们可以利用此性质实现自适应策略（尽管这可能导致某些信息的丢失）。

例如，尽管收盘价 `close` 通常既不平稳也没有固定的取值范围，使用 `close` 本身往往不能直接指导交易。但是，通过对其进行自适应标准化，我们可以构建一个以 `1` 为上轨，`0` 为中轨，`-1` 为下轨的通道突破策略，从而得到自适应布林策略。

## 实验验证

使用币安 ETHUSDT 永续数据，验证自适应布林标准化后的收盘价构造的通道突破策略，是否与原始策略等价

首先 J 神原始自适应布林策略

```python
def adapt_bolling_J(df, n):
    # ===计算指标
    # 计算均线
    df['median'] = df['close'].rolling(n, min_periods=1).mean()
    # 计算上轨、下轨道
    df['std'] = df['close'].rolling(n, min_periods=1).std(ddof=0)
    df['zscore'] = (df['close'] - df['median']) / df['std']
    df['zscore_abs_max'] = df['zscore'].abs().rolling(n, min_periods=1).max().shift(1)
    df['up'] = df['median'] + df['zscore_abs_max'] * df['std']
    df['dn'] = df['median'] - df['zscore_abs_max'] * df['std']
    
    # 突破上轨做多
    condition1 = df['close'] > df['up']
    condition2 = df['close'].shift(1) <= df['up'].shift(1)
    condition = condition1 & condition2
    df.loc[condition, 'signal_long'] = 1

    # 突破下轨做空
    condition1 = df['close'] < df['dn']
    condition2 = df['close'].shift(1) >= df['dn'].shift(1)
    condition = condition1 & condition2
    df.loc[condition, 'signal_short'] = -1

    # 均线平仓(多头持仓)
    condition1 = df['close'] < df['median']
    condition2 = df['close'].shift(1) >= df['median'].shift(1)
    condition = condition1 & condition2
    df.loc[condition, 'signal_long'] = 0

    # 均线平仓(空头持仓)
    condition1 = df['close'] > df['median']
    condition2 = df['close'].shift(1) <= df['median'].shift(1)
    condition = condition1 & condition2
    df.loc[condition, 'signal_short'] = 0
    
    # 合并去重
    df['signal_long'] = df['signal_long'].ffill().fillna(0)
    df['signal_short'] = df['signal_short'].ffill().fillna(0)
    df['signal'] = df[['signal_long', 'signal_short']].sum(axis=1)
    temp = df[['signal']]
    temp = temp[temp['signal'] != temp['signal'].shift(1)]
    df['signal'] = temp['signal']
    
    df.drop(columns=['signal_long', 'signal_short', 'median', 'std', 'zscore', 'zscore_abs_max', 'up', 'dn'], 
            inplace=True)
    return df
```

首先，定义自适应通道突破（上轨`1`,中轨`0`,下轨`-1`）

``` python
def adapt_channel_break(df: pd.DataFrame, indicator: str):
    
    # 突破上轨做多
    condition1 = df[indicator] == 1
    condition2 = df[indicator].shift(1) < 1
    condition = condition1 & condition2
    df.loc[condition, 'signal_long'] = 1

    # 突破下轨做空
    condition1 = df[indicator] == -1
    condition2 = df[indicator].shift(1) > -1
    condition = condition1 & condition2
    df.loc[condition, 'signal_short'] = -1

    # 均线平仓(多头持仓)
    condition1 = df[indicator] < 0
    condition2 = df[indicator].shift(1) >= 0
    condition = condition1 & condition2
    df.loc[condition, 'signal_long'] = 0

    # 均线平仓(空头持仓)
    condition1 = df[indicator] > 0
    condition2 = df[indicator].shift(1) <= 0
    condition = condition1 & condition2
    df.loc[condition, 'signal_short'] = 0
    
    # 合并去重
    df['signal_long'] = df['signal_long'].ffill().fillna(0)
    df['signal_short'] = df['signal_short'].ffill().fillna(0)
    df['signal'] = df[['signal_long', 'signal_short']].sum(axis=1)
    temp = df[['signal']]
    temp = temp[temp['signal'] != temp['signal'].shift(1)]
    df['signal'] = temp['signal']    
    df.drop(columns=['signal_long', 'signal_short', indicator], inplace=True)
```

基于自适应布林标准化的通道突破策略则非常简单：

```python
def adapt_bolling_new(df, n):
    # ===计算指标
    adapt_bolling_normalize(df, 'close', n)
    adapt_channel_break(df, 'close_zscore_max_abs_norm')
    return df
```

基于以下代码验证开平仓信号正确性: 
```python

def read_data():
    df = pd.read_feather('ETHUSDT.fea')
    return df

df_orig = read_data()
df_orig = adapt_bolling_J(df_orig, 150)
df_orig = df_orig[df_orig['candle_begin_time'] >= '20200101']

df_new = read_data()
df_new = adapt_bolling_new(df_new, 150)
df_new = df_new[df_new['candle_begin_time'] >= '20200101']
print('NAN all correct:', (df_orig['signal'].isna() == df_new['signal'].isna()).all())

print('Signals all correct:',
  (df_orig.loc[df_orig['signal'].notnull(), 'signal'] == df_new.loc[df_new['signal'].notnull(), 'signal']).all())
```

输出

```
NAN all correct: True
Signals all correct: True
```

信号一致

## 扩展

Z-Score 标准化，Max-Abs 标准化，自适应布林标准化，三种标准化方法，结合自适应通道突破，可以构造趋势策略；也可以用于其他策略

一下是两个例子，用于扩展自适应趋势策略，不一定赚钱，但可能有些参考价值

例如，对于动量因子，`df['mtm'] = df['close'].pct_change(n)`

由于动量的本质是历史收益率，而收益率本身具有平稳性，因此可以仅使用 Max-Abs 标准化构造，构造自适应动量策略

```python
def adapt_mtm(df, n):
    df['mtm'] = df['close'].pct_change(n)
    max_abs_normalize(df, 'mtm', n)
    adapt_channel_break(df, 'mtm_max_abs_norm')
    df.drop(columns='mtm', inplace=True)
    return df
```

也可以对 mtm 使用自适应布林标准化，构造自适应动量布林策略

```python
def adapt_mtm_bolling(df, n):
    df['mtm'] = df['close'].pct_change(n)
    adapt_bolling_normalize(df, 'mtm', n)
    adapt_channel_break(df, 'mtm_zscore_max_abs_norm')
    df.drop(columns='mtm', inplace=True)
    return df
```
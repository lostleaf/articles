# AROON Technical Indicator Implementation and Optimization: Accelerating the rolling argmax Operator by 1500x using Numba and Monotonic Queue

AROON is a common technical indicator, whose definition can be found at [Investopedia](https://www.investopedia.com/terms/a/aroon.asp#toc-formulas-for-the-aroon-indicator).

The challenge in implementing this indicator lies in the rolling argmax operator, for which pandas does not provide an official implementation. A naive implementation based on Series results in considerably slow computations.

Optimizing the computation of this indicator presents an interesting but not easy problem.

Code reference is available at [Github](https://github.com/lostleaf/articles/blob/master/20221013numba_aroon/optimize_aroon.ipynb).

## Data and Parameters

First, we generate a HLC DataFrame of length 50,000 through a random walk to simulate the trend of a certain commodity.

We set the lookback window `n=200`: 

``` python
length = 50000
n = 200

np.random.seed(2022)
cl = 10000 + np.cumsum(np.random.normal(scale=20, size=length))
hi = cl + np.random.rand() * 100
lo = cl - np.random.rand() * 100

df = pd.DataFrame({'high': hi, 'low': lo, 'close': cl})

print(df.head()[['high', 'low', 'close']].to_markdown(), '\n')

print(df.tail()[['high', 'low', 'close']].to_markdown(), '\n')

print(df.describe().to_markdown())
```

Samples and statistics of the generated data:


|      |    high |     low |   close |
| ---: | ------: | ------: | ------: |
|    0 | 10017.9 | 9965.82 | 9999.99 |
|    1 | 10012.4 | 9960.32 | 9994.49 |
|    2 | 10009.6 | 9957.53 | 9991.71 |
|    3 | 10049.3 | 9997.23 | 10031.4 |
|    4 |   10055 | 10002.9 |   10037 |

|       |    high |     low |   close |
| ----: | ------: | ------: | ------: |
| 49995 | 13187.1 |   13135 | 13169.2 |
| 49996 | 13209.7 | 13157.6 | 13191.8 |
| 49997 | 13223.1 |   13171 | 13205.2 |
| 49998 | 13221.6 | 13169.5 | 13203.7 |
| 49999 | 13242.1 |   13190 | 13224.2 |

|       |    high |     low |   close |
| :---- | ------: | ------: | ------: |
| count |   50000 |   50000 |   50000 |
| mean  | 11801.7 | 11749.6 | 11783.7 |
| std   | 943.887 | 943.887 | 943.887 |
| min   | 9543.01 | 9490.91 | 9525.09 |
| 25%   | 11007.4 | 10955.3 | 10989.5 |
| 50%   | 11986.2 | 11934.1 | 11968.3 |
| 75%   | 12524.3 | 12472.2 | 12506.4 |
| max   | 14159.4 | 14107.3 | 14141.5 |

## Naive Implementation Based on Pandas Series

Initially, we implement a naive version of the Aroon indicator using Series as a baseline.

```python
def aroon_naive(df, n):
    # 求列的 rolling 窗口内的最大值对于的 index
    high_len = df['high'].rolling(n, min_periods=1).apply(lambda x: pd.Series(x).idxmax())

    # 当前日距离过去N天最高价的天数
    high_len = df.index - high_len
    aroon_up = 100 * (n - high_len) / n

    low_len = df['low'].rolling(n, min_periods=1).apply(lambda x: pd.Series(x).idxmin())
    low_len = df.index - low_len
    aroon_down = 100 * (n - low_len) / n

    return aroon_up, aroon_down

%time up_naive, down_naive = aroon_naive(df, n)
```

This version utilizes DataFrame.rolling and creates a new Series in each lambda function to compute the rolling argmax.

Due to the high cost of creating Series and suboptimal complexity, where complexity = O(number of candles * backhour), this code is exceedingly slow.

Test results:

```
CPU times: user 2.82 s, sys: 134 ms, total: 2.95 s
Wall time: 2.83 s
```

## Testing Function

We verify the correctness of each implementation using the following function.

This function will print out how many `nan` values are in the the tested signal, how many non-`nan` values are correct or incorrect, and the accuracy rate.

``` python
def check_signal(sig_original, sig_check):
    print('Num of nan:', sig_check.isna().sum())
    mask = sig_original.notnull() & sig_check.notnull()
    n_eq = ((sig_original[mask] - sig_check[mask]).abs() < 1e-8).sum()
    l = mask.sum()
    print(f'Num of equal: {n_eq}, Num of not equal: {l - n_eq}, Ratio good: {n_eq / l * 100.0}%')
```

## Numpy Implementation

A simple optimization method is to use numpy functions, avoiding the repeated creation of Series and thus improving efficiency to some extent.

``` python
def high_len_cal(x):
    return (np.maximum.accumulate(x) == x.max()).sum()
  
def low_len_cal(x):
    return (np.minimum.accumulate(x) == x.min()).sum()

def aroon_numpy(df, n):
    high_len = df['high'].rolling(n).apply(high_len_cal, raw=True, engine='cython') - 1
    aroon_up = 100 * (n - high_len) / n

    low_len = df['low'].rolling(n).apply(low_len_cal, raw=True, engine='cython') - 1
    aroon_down = 100 * (n - low_len) / n
    return aroon_up, aroon_down

%time up_numpy, down_numpy = aroon_numpy(df, n)
print('Check up')
check_signal(up_naive, up_numpy)

print('Check down')
check_signal(down_naive, down_numpy)
```

Test results are as follows:

```
CPU times: user 369 ms, sys: 1.77 ms, total: 370 ms
Wall time: 370 ms
Check up
Num of nan: 199
Num of equal: 49801, Num of not equal: 0, Ratio good: 100.0%
Check down
Num of nan: 199
Num of equal: 49801, Num of not equal: 0, Ratio good: 100.0%
```

## Naive Numba Implementation

To avoid using python functions during the rolling process, we consider abandoning pandas and using numba njit + numpy .argmax to implement the rolling argmax.

``` python
@nb.njit(nb.int32[:](nb.float64[:], nb.int32), cache=True)
def rolling_argmin(arr, n):
    idx_min = 0
    results = np.empty(len(arr), dtype=np.int32)
    for i, x in enumerate(arr):
        if i < n:
            results[i] = np.argmin(arr[: i + 1])
        else:
            results[i] = np.argmin(arr[i - n + 1: i + 1]) + i - n + 1
    return results

def aroon_numba(df, n):
    low_len = pd.Series(rolling_argmin(df['low'].values, n))

    high_len = pd.Series(rolling_argmin(-df['high'].values, n))

    high_len = df.index - high_len
    low_len = df.index - low_len

    aroon_up = 100 * (n - high_len) / n
    aroon_down = 100 * (n - low_len) / n
    return pd.Series(aroon_up), pd.Series(aroon_down)

%time up_numba, down_numba = aroon_numba(df, n)
print('Check up')
check_signal(up_naive, up_numba)

print('Check down')
check_signal(down_naive, down_numba)
```

Test results are as follows, with no `nan` values in the signal and all results being correct.

```
CPU times: user 27.8 ms, sys: 189 µs, total: 28 ms
Wall time: 28 ms
Check up
Num of nan: 0
Num of equal: 50000, Num of not equal: 0, Ratio good: 100.0%
Check down
Num of nan: 0
Num of equal: 50000, Num of not equal: 0, Ratio good: 100.0%
```

## Numba + Monotonic Queue Implementation

In the naive numba implementation, although time efficiency was improved by avoiding frequent python function calls, its complexity still remains at O(number of candles * backhour), which is not ideal.

In the field of algorithms, there is a standard optimal solution for rolling argmax, which requires the use of a monotonic queue to optimize, bringing the complexity down to O(number of candles + backhour), a significant reduction.

The specific principle of this algorithm is more complex and can be referred to in the [LeetCode 239 solution](https://leetcode.cn/problems/sliding-window-maximum/solution/hua-dong-chuang-kou-zui-da-zhi-by-leetco-ki6m/), which is considered hard difficulty on LeetCode.

``` python
@nb.njit(nb.int32[:](nb.float64[:], nb.int32), cache=True)
def rolling_argmin_queue(arr, n):
    results = np.empty(len(arr), dtype=np.int32)
    
    head = 0
    tail = 0
    que_idx = np.empty(len(arr), dtype=np.int32)
    for i, x in enumerate(arr[:n]):
        while tail > 0 and arr[que_idx[tail - 1]] > x:
            tail -= 1
        que_idx[tail] = i
        tail += 1
        results[i] = que_idx[0]
    
    for i, x in enumerate(arr[n:], n):
        if que_idx[head] <= i - n:
            head += 1
        while tail > head and arr[que_idx[tail - 1]] > x:
            tail -= 1
        que_idx[tail] = i
        tail += 1
        results[i] = que_idx[head]
    return results
            
def aroon_numba_queue(df, n):
    low_len = pd.Series(rolling_argmin_queue(df['low'].values, n))
    high_len = pd.Series(rolling_argmin_queue(-df['high'].values, n))
    
    high_len = df.index - high_len
    low_len = df.index - low_len

    aroon_up = 100 * (n - high_len) / n
    aroon_down = 100 * (n - low_len) / n
    return pd.Series(aroon_up), pd.Series(aroon_down)

%time up_nbque, down_nbque = aroon_numba_queue(df, n)
print('Check up')
check_signal(up_naive, up_nbque)

print('Check down')
check_signal(down_naive, down_nbque)
```

Test results are as follows, with no `nan` values in the signal and all results being correct.

```
CPU times: user 1.79 ms, sys: 67 µs, total: 1.86 ms
Wall time: 1.87 ms
Check up
Num of nan: 0
Num of equal: 50000, Num of not equal: 0, Ratio good: 100.0%
Check down
Num of nan: 0
Num of equal: 50000, Num of not equal: 0, Ratio good: 100.0%
```

# Conclusion

Comparing the time consumption of various methods, the summary is as follows:

|               | Time(ms) | Speedup |
| ------------- | -------- | ------- |
| Naive         | 2830     | 1       |
| Numpy         | 370      | 7.65    |
| Naive Numba   | 28       | 101     |
| Numba + Queue | 1.87     | 1513    |

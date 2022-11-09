# AROON 技术指标实现与优化：使用 Numba 和单调队列将 rolling argmax 算子提速1500倍

AROON 是一个常见的技术指标，其定义可以参考 [Investopedia](https://www.investopedia.com/terms/a/aroon.asp#toc-formulas-for-the-aroon-indicator)

该因子实现难点在于 rolling argmax 算子，pandas 并没有给出该算子的官方实现，基于 Series 的 naive 实现会导致计算相当慢

优化该因子的计算是一个有趣的问题

代码可以参考 [Github](https://github.com/lostleaf/articles/blob/master/20221013%E4%BD%BF%E7%94%A8numba%E5%92%8C%E5%8D%95%E8%B0%83%E9%98%9F%E5%88%97%E5%B0%86AROON%E5%9B%A0%E5%AD%90%E6%8F%90%E9%80%9F1500%E5%80%8D/optimize_aroon.ipynb)

## 数据与参数

首先，我们通过随机游走，生成一个长度为 50000 的 HLC DataFrame，用于模拟某种大宗商品的走势

取回看窗口 `n=200`

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

## 基于 Series 的 Naive 实现

首先，我们基于 Series 实现一个 naive 版本的 Aroon 因子，并以此作为基准

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

该版本使用了 DataFrame.rolling，并且每次都在 lambda 函数中创建了一个 Series，用来计算 rolling argmax

因为创建 Series 成本昂贵，且复杂度不优，复杂度 = O(K线数量 * backhour)，所以这段代码非常慢

测试结果:

```
CPU times: user 2.82 s, sys: 134 ms, total: 2.95 s
Wall time: 2.83 s
```


## 测试函数

我们基于以下函数验证每种实现的正确性

``` python
def check_signal(sig_original, sig_check):
    print('Num of nan:', sig_check.isna().sum())
    mask = sig_original.notnull() & sig_check.notnull()
    n_eq = ((sig_original[mask] - sig_check[mask]).abs() < 1e-8).sum()
    l = mask.sum()
    print(f'Num of equal: {n_eq}, Num of not equal: {l - n_eq}, Ratio good: {n_eq / l * 100.0}%')
```

该函数会打印出，被测试的 signal，有多少个 nan，非 nan 值中有多少正确多少错误，以及正确率

## numpy 实现

一种简单的优化方法是，使用 numpy 函数，避免了反复创建 Series，一定程度上提升了效率

``` python
def high_len_cal(x):
    return (np.maximum.accumulate(x) == x.max()).sum()
  
def low_len_cal(x):
    return (np.minimum.accumulate(x) == x.min()).sum()

def aroon_numpy(df, n):
    high_len = df['high'].rolling(n).apply(high_len_cal, raw=True, engine='cython')
    aroon_up = 100 * (n - high_len) / n

    low_len = df['low'].rolling(n).apply(low_len_cal, raw=True, engine='cython')
    aroon_down = 100 * (n - low_len) / n
    return aroon_up, aroon_down

%time up_numpy, down_numpy = aroon_ysx(df, n)
print('Check up')
check_signal(up_naive, up_numpy)

print('Check down')
check_signal(down_naive, down_numpy)
```

测试结果如下

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

## 朴素的 Numba 实现

为了避免在 rolling 的过程中使用 python 函数，我们考虑抛弃 pandas，使用 numba njit + numpy .argmax 来实现 rolling argmax

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

测试结果如下，信号中没有 nan 出现，结果全对

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

## Numba + 单调队列实现

以上朴素 numba 实现中，避免了频繁 python 函数调用，时间上得到了优化，然而其复杂度仍然为 O(K线数量 * backhour)，导致性能仍然不够完美

在算法领域，rolling argmax 有个标准最优解，需要使用单调队列来优化，其复杂度为 O(K线数量 + backhour)，复杂度降低一个数量级

该算法具体原理较复杂，具体可参考 [leetcode239 题解](https://leetcode.cn/problems/sliding-window-maximum/solution/hua-dong-chuang-kou-zui-da-zhi-by-leetco-ki6m/) ，为 leetcode hard 难度

```python
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

测试结果如下，信号中没有 nan 出现，结果全对

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

# 结论

对比几种方法耗时，总结如下

|               | Time(ms) | Speedup |
| ------------- | -------- | ------- |
| Naive         | 2830     | 1       |
| Numpy         | 370      | 7.65    |
| Naive Numba   | 28       | 101     |
| Numba + Queue | 1.87     | 1513    |

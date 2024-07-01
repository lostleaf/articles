# 【BMAC2.0-前传】利用 asyncio 和 Websocket 获取并录制币安 K 线行情

众所周知，币安有两种获取 K 线数据的方式 —— REST API 和 Websocket

BMAC1 通过使用反复调用 REST API 的方式获取闭合 K 线，这样做虽然可以稳定地获取 K 线数据，但速度相对较慢且会消耗大量的 API 权重

BMAC2 则会使用 REST API 和 Websocket 混合驱动的方式，更加高效地获取 K 线数据

本文为 BMAC2 技术报告第一篇，将介绍其中的核心技术：使用 Python asyncio，通过订阅 Websocket K 线数据频道，异步获取并录制币安 K 线行情

## 连接币安行情推送 Websocket 

要通过 Websocket 获取行情，首先需要实现一个稳健的 Websocket 客户端

由于本人并不是 HTTP 专家，这里复制并精简了 [python-binance 的 `ReconnectingWebsocket`](https://github.com/sammchardy/python-binance/blob/master/binance/streams.py)，封装为 `ws_basics.py`，直接调用即可

在 `binance_market_ws.py` 中，我们定义生成币安 K 线 Websocket 连接的函数如下

``` python
from ws_basics import ReconnectingWebsocket

# 现货 WS Base Url
SPOT_STREAM_URL = 'wss://stream.binance.com:9443/'

# U 本位合约 WS Base Url
USDT_FUTURES_FSTREAM_URL = 'wss://fstream.binance.com/'

# 币本位合约 WS Base Url
COIN_FUTURES_DSTREAM_URL = 'wss://dstream.binance.com/'


def get_coin_futures_multi_candlesticks_socket(symbols, time_inteval):
    """
    返回币本位合约单周期多个 symbol K 线 websocket 连接
    """
    channels = [f'{s.lower()}@kline_{time_inteval}' for s in symbols]
    return ReconnectingWebsocket(
        path='/'.join(channels),
        url=COIN_FUTURES_DSTREAM_URL,
        prefix='stream?streams=',
    )


def get_usdt_futures_multi_candlesticks_socket(symbols, time_inteval):
    """
    返回 U 本位合约单周期多个 symbol K 线 websocket 连接
    """
    channels = [f'{s.lower()}@kline_{time_inteval}' for s in symbols]
    return ReconnectingWebsocket(
        path='/'.join(channels),
        url=USDT_FUTURES_FSTREAM_URL,
        prefix='stream?streams=',
    )

def get_spot_multi_candlesticks_socket(symbols, time_inteval):
    """
    返回现货单周期多个 symbol K 线 websocket 连接
    """
    channels = [f'{s.lower()}@kline_{time_inteval}' for s in symbols]
    return ReconnectingWebsocket(
        path='/'.join(channels),
        url=SPOT_STREAM_URL,
        prefix='stream?streams=',
    )

```

通过以下示例代码 `ex1_recv_single.py`，我们尝试通过 Websocket 连接收取 BTCUSDT 永续合约 1 分钟 K 线数据并打印至屏幕

```python
import asyncio
import logging

from binance_market_ws import get_usdt_futures_multi_candlesticks_socket


async def main():
    socket = get_usdt_futures_multi_candlesticks_socket(['BTCUSDT'], '1m')
    async with socket as socket_conn:
        while True:
            try:
                res = await socket_conn.recv()
                print(res)
            except asyncio.TimeoutError:
                logging.error('Recv candle ws timeout')
                break


if __name__ == '__main__':
    asyncio.run(main())
```

运行以上代码，选取其中较为有代表性的几条数据如下

```python
{'stream': 'btcusdt@kline_1m', 'data': {'e': 'kline', 'E': 1719765539838, 's': 'BTCUSDT', 'k': {'t': 1719765480000, 'T': 1719765539999, 's': 'BTCUSDT', 'i': '1m', 'f': 5122041311, 'L': 5122041720, 'o': '61607.90', 'c': '61623.30', 'h': '61623.30', 'l': '61605.30', 'v': '16.692', 'n': 410, 'x': False, 'q': '1028411.77850', 'V': '12.553', 'Q': '773414.33780', 'B': '0'}}}
{'stream': 'btcusdt@kline_1m', 'data': {'e': 'kline', 'E': 1719765540037, 's': 'BTCUSDT', 'k': {'t': 1719765480000, 'T': 1719765539999, 's': 'BTCUSDT', 'i': '1m', 'f': 5122041311, 'L': 5122041728, 'o': '61607.90', 'c': '61624.90', 'h': '61624.90', 'l': '61605.30', 'v': '16.710', 'n': 418, 'x': True, 'q': '1029521.00470', 'V': '12.571', 'Q': '774523.56400', 'B': '0'}}}
{'stream': 'btcusdt@kline_1m', 'data': {'e': 'kline', 'E': 1719765540545, 's': 'BTCUSDT', 'k': {'t': 1719765540000, 'T': 1719765599999, 's': 'BTCUSDT', 'i': '1m', 'f': 5122041729, 'L': 5122041730, 'o': '61624.90', 'c': '61625.00', 'h': '61625.00', 'l': '61624.90', 'v': '0.026', 'n': 2, 'x': False, 'q': '1602.24770', 'V': '0.003', 'Q': '184.87500', 'B': '0'}}}
```

可以看到，每条数据被解析为一个 Python 字典，接下来我们需要解析该数据字典，将其转换为我们熟悉的 DataFrame

## 解析币安行情推送数据

根据[币安文档](https://developers.binance.com/docs/zh-CN/derivatives/usds-margined-futures/websocket-market-streams/Kline-Candlestick-Streams)，我们可以将数据字典中的 k 字段与常用的 K 线 DataFrame 列名一一对应

```python
def convert_to_dataframe(x, interval_delta):
    """
    解析 WS 返回的数据字典，返回 DataFrame
    """
    columns = [
        'candle_begin_time', 'open', 'high', 'low', 'close', 'volume', 'quote_volume', 'trade_num',
        'taker_buy_base_asset_volume', 'taker_buy_quote_asset_volume'
    ]
    candle_data = [
        pd.to_datetime(int(x['t']), unit='ms', utc=True),
        float(x['o']),
        float(x['h']),
        float(x['l']),
        float(x['c']),
        float(x['v']),
        float(x['q']),
        float(x['n']),
        float(x['V']),
        float(x['Q'])
    ]

    # 以 K 线结束时间为时间戳
    return pd.DataFrame(data=[candle_data], columns=columns, index=[candle_data[0] + interval_delta])
```

为了数据解析的稳健性，我们还需要更进一步，采用防御性编程，严格检查数据有效性，并判断 K 线是否闭合，仅接收闭合 K 线

```python
def handle_candle_data(res, interval_delta):
    """
    处理 WS 返回数据
    """

    # 防御性编程，如果币安出现错误未返回 data 字段，则抛弃
    if 'data' not in res:
        return

    # 取出 data 字段
    data = res['data']

    # 防御性编程，如果 data 中不包含 e 字段或 e 字段（数据类型）不为 kline 或 data 中没有 k 字段（K 线数据），则抛弃
    if data.get('e', None) != 'kline' or 'k' not in data:
        return

    # 取出 k 字段，即 K 线数据
    candle = data['k']

    # 判断 K 线是否闭合，如未闭合则抛弃
    is_closed = candle.get('x', False)
    if not is_closed:
        return

    # 将 K 线转换为 DataFrame
    df_candle = convert_to_dataframe(candle, interval_delta)
    return df_candle
```

基于以下示例代码 `ex2_parse_data.py`，我们尝试解析上一节中收取的 K 线数据（K 线数据保存为 `ex2_ws_candle.json`）

```python
def main():
    # 载入 JSON 数据
    data = json.load(open('ex2_ws_candle.json'))

    # K 线周期为 1m
    interval_delta = pd.Timedelta(minutes=1)

    # 尝试解析每条 WS 数据
    for idx, row in enumerate(data, 1):
        row_parsed = handle_candle_data(row, interval_delta)
        if row_parsed is None:
            print(f'Row{idx} is None')
        else:
            print(f'Row{idx} candlestick\n' + str(row_parsed))


if __name__ == '__main__':
    main()
```

输出如下

```
Row1 is None
Row2 candlestick
                                  candle_begin_time     open     high      low    close  volume  quote_volume  trade_num  taker_buy_base_asset_volume  taker_buy_quote_asset_volume
2024-06-30 16:39:00+00:00 2024-06-30 16:38:00+00:00  61607.9  61624.9  61605.3  61624.9   16.71  1.029521e+06      418.0                       12.571                    774523.564
Row3 is None
```

其中第一条和第三条由于 K 线不闭合，因此被抛弃，输出为 None

第二条为闭合 K 线，被解析为 DataFrame

## 单周期多标的 Websocket K 线数据接收器 CandleListener

结合以上两节，可定义 `CandleListener` (`candle_listener.py`)

其中主函数为 `start_listen`，负责建立 Websocket 连接并收取 K 线数据

`handle_candle_data` 函数则负责解析接收到的 K 线数据，将有效且闭合的 K 线打入消息队列（下一节中简介其作用） `self.que` 中

```python
import asyncio
import logging
from datetime import datetime, timedelta

import pandas as pd
import pytz

from binance_market_ws import (get_coin_futures_multi_candlesticks_socket, get_spot_multi_candlesticks_socket,
                               get_usdt_futures_multi_candlesticks_socket)


def convert_to_dataframe(x, interval_delta):
    """
    解析 WS 返回的数据字典，返回 DataFrame
    """
    columns = [
        'candle_begin_time', 'open', 'high', 'low', 'close', 'volume', 'quote_volume', 'trade_num',
        'taker_buy_base_asset_volume', 'taker_buy_quote_asset_volume'
    ]
    candle_data = [
        pd.to_datetime(int(x['t']), unit='ms', utc=True),
        float(x['o']),
        float(x['h']),
        float(x['l']),
        float(x['c']),
        float(x['v']),
        float(x['q']),
        float(x['n']),
        float(x['V']),
        float(x['Q'])
    ]

    # 以 K 线结束时间为时间戳
    return pd.DataFrame(data=[candle_data], columns=columns, index=[candle_data[0] + interval_delta])


class CandleListener:

    # 交易类型到 ws 函数映射
    TRADE_TYPE_MAP = {
        'usdt_futures': get_usdt_futures_multi_candlesticks_socket,
        'coin_futures': get_coin_futures_multi_candlesticks_socket,
        'spot': get_spot_multi_candlesticks_socket
    }

    def __init__(self, type_, symbols, time_interval, que):
        # 交易类型
        self.trade_type = type_
        # 交易标的
        self.symbols = set(symbols)
        # K 线周期
        self.time_interval = time_interval
        self.interval_delta = convert_interval_to_timedelta(time_interval)
        # 消息队列
        self.que: asyncio.Queue = que
        # 重链接 flag
        self.req_reconnect = False

    async def start_listen(self):
        """
        WS 监听主函数
        """

        if not self.symbols:
            return
        
        socket_func = self.TRADE_TYPE_MAP[self.trade_type]
        while True:
            # 创建 WS
            socket = socket_func(self.symbols, self.time_interval)
            async with socket as socket_conn:
                # WS 连接成功后，获取并解析数据
                while True:
                    if self.req_reconnect: # 如果需要重连，则退出重新连接
                        self.req_reconnect = False
                        break
                    try:
                        res = await socket_conn.recv()
                        self.handle_candle_data(res)
                    except asyncio.TimeoutError: # 如果长时间未收到数据（默认60秒，正常情况K线每1-2秒推送一次），则退出重新连接
                        logging.error('Recv candle ws timeout, reconnecting')
                        break

    def handle_candle_data(self, res):
        """
        处理 WS 返回数据
        """

        # 防御性编程，如果币安出现错误未返回 data 字段，则抛弃
        if 'data' not in res:
            return

        # 取出 data 字段
        data = res['data']

        # 防御性编程，如果 data 中不包含 e 字段或 e 字段（数据类型）不为 kline 或 data 中没有 k 字段（K 线数据），则抛弃
        if data.get('e', None) != 'kline' or 'k' not in data:
            return

        # 取出 k 字段，即 K 线数据
        candle = data['k']

        # 判断 K 线是否闭合，如未闭合则抛弃
        is_closed = candle.get('x', False)
        if not is_closed:
            return

        # 将 K 线转换为 DataFrame
        df_candle = convert_to_dataframe(candle, self.interval_delta)

        # 将 K 线 DataFrame 放入通信队列
        self.que.put_nowait({
            'type': 'candle_data',
            'data': df_candle,
            'closed': is_closed,
            'run_time': df_candle.index[0],
            'symbol': data['s'],
            'time_interval': self.time_interval,
            'trade_type': self.trade_type,
            'recv_time': now_time()
        })

    def add_symbols(self, *symbols):
        for symbol in symbols:
            self.symbols.add(symbol)

    def remove_symbols(self, *symbols):
        for symbol in symbols:
            if symbol in self.symbols:
                self.symbols.remove(symbol)

    def reconnect(self):
        self.req_reconnect = True
```

## 录制币安 K 线行情

在这一节中，我们通过多个 Websocket 连接，异步接收现货、U 本位合约、币本位合约的 K 线数据，并以 parquet 格式将 K 线数据 DataFrame 存储在硬盘上

为了达到这一目的，我们首先回顾生产者-消费者架构：生产者-消费者架构是一种并发场景下常见的软件设计模式，其中生产者提供数据，消费者处理数据，生产者和消费者之间通常使用消息队列传递数据

这种模式将数据产生和数据处理进行了解耦，在我们的业务场景中，通过使用多生产者和单一消费者，可以保证硬盘写入的正确性

示例代码`ex3_record_multi.py`中，我们定义3个生产者，均为 `CandleListener` 实例：

- `listener_usdt_perp_1m`: 收取 U 本位 BTCUSDT 和 ETHUSDT 合约 1 分钟线数据
- `listener_coin_perp_3m`: 收取币本位 BTCUSD_PERP 和 ETHUSD_PERP 合约 3 分钟线数据
- `listener_spot_1m`: 收取现货 BTCUSDT 和 BNBUSDT 1 分钟线数据

定义一个消费者，用于更新 K 线数据 

```python
def update_candle_data(df_new: pd.DataFrame, symbol, time_interval, trade_type):
    """
    将接收到的 K 线数据 DataFrame 以 parquet 格式写入硬盘
    """
    output_path = f'{trade_type}_{symbol}_{time_interval}.pqt'

    if not os.path.exists(output_path):
        df_new.to_parquet(output_path, compression='zstd')
        return

    df = pd.read_parquet(output_path)
    df = pd.concat([df, df_new])
    df.sort_index()
    df.drop_duplicates('candle_begin_time')
    df.to_parquet(output_path)


async def dispatcher(main_que: asyncio.Queue):
    """
    用于处理接收到的 K 线数据的消费者
    """
    while True:
        # 从主队列取出数据
        req = await main_que.get()
        run_time = req['run_time']
        req_type = req['type']

        # 根据数据类型调用相应的处理函数
        if req_type == 'candle_data':  # K 线数据更新
            symbol = req['symbol']
            time_interval = req['time_interval']
            trade_type = req['trade_type']
            update_candle_data(req['data'], symbol, time_interval, trade_type)
            logging.info('Record %s %s-%s at %s', trade_type, symbol, time_interval, run_time)
        else:
            logging.warning('Unknown request %s %s', req_type, run_time)
```

核心调用代码如下

```python
# ex3_record_multi.py

async def main():
    logging.info('Start record candlestick data')
    # 主队列
    main_que = asyncio.Queue()

    # 生产者
    listener_usdt_perp_1m = CandleListener('usdt_futures', ['BTCUSDT', 'ETHUSDT'], '1m', main_que)
    listener_coin_perp_3m = CandleListener('coin_futures', ['BTCUSD_PERP', 'ETHUSD_PERP'], '3m', main_que)
    listener_spot_1m = CandleListener('spot', ['BTCUSDT', 'BNBUSDT'], '1m', main_que)

    # 消费者
    dispatcher_task = dispatcher(main_que)

    await asyncio.gather(listener_usdt_perp_1m.start_listen(), listener_coin_perp_3m.start_listen(),
                         listener_spot_1m.start_listen(), dispatcher_task)
```

运行时输出如下

```bash
20240630 22:59:36 (INFO) - Start record candlestick data
20240630 23:00:00 (INFO) - Record usdt_futures ETHUSDT-1m at 2024-06-30 15:00:00+00:00
20240630 23:00:00 (INFO) - Record spot BNBUSDT-1m at 2024-06-30 15:00:00+00:00
20240630 23:00:00 (INFO) - Record spot BTCUSDT-1m at 2024-06-30 15:00:00+00:00
20240630 23:00:00 (INFO) - Record usdt_futures BTCUSDT-1m at 2024-06-30 15:00:00+00:00
20240630 23:00:01 (INFO) - Record coin_futures ETHUSD_PERP-3m at 2024-06-30 15:00:00+00:00
20240630 23:00:02 (INFO) - Record coin_futures BTCUSD_PERP-3m at 2024-06-30 15:00:00+00:00
20240630 23:01:00 (INFO) - Record spot BNBUSDT-1m at 2024-06-30 15:01:00+00:00
20240630 23:01:00 (INFO) - Record spot BTCUSDT-1m at 2024-06-30 15:01:00+00:00
```

硬盘写入如下 parquet 文件

```
-rw-rw-r-- 1 admin admin 8.8K Jun 30 23:09 coin_futures_BTCUSD_PERP_3m.pqt
-rw-rw-r-- 1 admin admin 8.8K Jun 30 23:09 coin_futures_ETHUSD_PERP_3m.pqt
-rw-rw-r-- 1 admin admin 9.3K Jun 30 23:10 spot_BNBUSDT_1m.pqt
-rw-rw-r-- 1 admin admin 9.3K Jun 30 23:10 spot_BTCUSDT_1m.pqt
-rw-rw-r-- 1 admin admin 9.4K Jun 30 23:10 usdt_futures_BTCUSDT_1m.pqt
-rw-rw-r-- 1 admin admin 9.4K Jun 30 23:10 usdt_futures_ETHUSDT_1m.pqt
```

录制数据如

```python
In [2]: pd.read_parquet('usdt_futures_BTCUSDT_1m.pqt')
Out[2]:
                                  candle_begin_time     open     high      low    close   volume  quote_volume  trade_num  taker_buy_base_asset_volume  taker_buy_quote_asset_volume
2024-06-30 15:00:00+00:00 2024-06-30 14:59:00+00:00  61580.2  61586.1  61580.2  61581.7   18.203  1.121016e+06      340.0                        6.931                  4.268353e+05
2024-06-30 15:01:00+00:00 2024-06-30 15:00:00+00:00  61581.8  61612.8  61581.7  61612.7   79.385  4.890301e+06     1015.0                       62.865                  3.872662e+06
... 中间省略 ...
2024-06-30 15:10:00+00:00 2024-06-30 15:09:00+00:00  61643.5  61643.5  61633.5  61636.9   35.319  2.176951e+06      530.0                       11.421                  7.039525e+05

In [3]: pd.read_parquet('coin_futures_ETHUSD_PERP_3m.pqt')
Out[3]:
                                  candle_begin_time     open     high      low    close   volume  quote_volume  trade_num  taker_buy_base_asset_volume  taker_buy_quote_asset_volume
2024-06-30 15:00:00+00:00 2024-06-30 14:57:00+00:00  3386.98  3387.00  3386.48  3386.98  54646.0    161.348842      259.0                      31434.0                     92.810019
2024-06-30 15:03:00+00:00 2024-06-30 15:00:00+00:00  3386.98  3389.36  3386.98  3389.35  29754.0     87.806150      275.0                      27826.0                     82.116929
2024-06-30 15:06:00+00:00 2024-06-30 15:03:00+00:00  3389.36  3389.79  3387.27  3388.39  39127.0    115.465249      363.0                       9255.0                     27.315708
2024-06-30 15:09:00+00:00 2024-06-30 15:06:00+00:00  3388.39  3390.59  3388.39  3390.22  15443.0     45.562138      220.0                       8133.0                     23.994293
```

自此，我们基本已经实现了一个简易的 BMAC

当然，要实现一个稳健的币安实盘数据客户端并不简单，还需要添加更多的逻辑来保证数据的完整性和正确性

BMAC2.0 本传 ——《BMAC 2.0: REST 和 Websocket 混合驱动的异步币安行情数据客户端》，敬请期待
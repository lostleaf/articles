# Using asyncio and WebSocket to Retrieve and Record Binance K-Line Market Data

As we know, Binance offers two methods to obtain K-line data: REST API and WebSocket. Among these, WebSocket is the preferred method recommended by Binance for obtaining real-time data.

This guide will show you how to use Python asyncio to subscribe to Binance K-line data via WebSocket, and asynchronously record Binance K-line market data as Pandas DataFrame in Parquet format.

## Connecting to Binance Market Data WebSocket

To get market data via WebSocket, we first need to implement a robust WebSocket client.

Here, we will use a simplified version of `ReconnectingWebsocket` from [python-binance's `streams.py`](https://github.com/sammchardy/python-binance/blob/master/binance/streams.py), which can be directly called from `ws_basics.py`.

In `binance_market_ws.py`, we define the functions to generate Binance K-line WebSocket connections as follows:

```python
from ws_basics import ReconnectingWebsocket

# Spot WS Base URL
SPOT_STREAM_URL = 'wss://stream.binance.com:9443/'

# USDT Futures WS Base URL
USDT_FUTURES_FSTREAM_URL = 'wss://fstream.binance.com/'

# Coin Futures WS Base URL
COIN_FUTURES_DSTREAM_URL = 'wss://dstream.binance.com/'


def get_coin_futures_multi_candlesticks_socket(symbols, time_inteval):
    """
    Returns a WebSocket connection for multiple symbols' K-line data for coin-margined futures.
    """
    channels = [f'{s.lower()}@kline_{time_inteval}' for s in symbols]
    return ReconnectingWebsocket(
        path='/'.join(channels),
        url=COIN_FUTURES_DSTREAM_URL,
        prefix='stream?streams=',
    )


def get_usdt_futures_multi_candlesticks_socket(symbols, time_inteval):
    """
    Returns a WebSocket connection for multiple symbols' K-line data for USDT-margined futures.
    """
    channels = [f'{s.lower()}@kline_{time_inteval}' for s in symbols]
    return ReconnectingWebsocket(
        path='/'.join(channels),
        url=USDT_FUTURES_FSTREAM_URL,
        prefix='stream?streams=',
    )

def get_spot_multi_candlesticks_socket(symbols, time_inteval):
    """
    Returns a WebSocket connection for multiple symbols' K-line data for spot trading.
    """
    channels = [f'{s.lower()}@kline_{time_inteval}' for s in symbols]
    return ReconnectingWebsocket(
        path='/'.join(channels),
        url=SPOT_STREAM_URL,
        prefix='stream?streams=',
    )
```

Using the example code `ex1_recv_single.py`, we try to connect to the WebSocket and receive BTCUSDT perpetual contract 1-minute K-line data and print it to the screen:

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

Running the above code, we get representative data similar to the following:

```python
{'stream': 'btcusdt@kline_1m', 'data': {'e': 'kline', 'E': 1719765539838, 's': 'BTCUSDT', 'k': {'t': 1719765480000, 'T': 1719765539999, 's': 'BTCUSDT', 'i': '1m', 'f': 5122041311, 'L': 5122041720, 'o': '61607.90', 'c': '61623.30', 'h': '61623.30', 'l': '61605.30', 'v': '16.692', 'n': 410, 'x': False, 'q': '1028411.77850', 'V': '12.553', 'Q': '773414.33780', 'B': '0'}}}
{'stream': 'btcusdt@kline_1m', 'data': {'e': 'kline', 'E': 1719765540037, 's': 'BTCUSDT', 'k': {'t': 1719765480000, 'T': 1719765539999, 's': 'BTCUSDT', 'i': '1m', 'f': 5122041311, 'L': 5122041728, 'o': '61607.90', 'c': '61624.90', 'h': '61624.90', 'l': '61605.30', 'v': '16.710', 'n': 418, 'x': True, 'q': '1029521.00470', 'V': '12.571', 'Q': '774523.56400', 'B': '0'}}}
{'stream': 'btcusdt@kline_1m', 'data': {'e': 'kline', 'E': 1719765540545, 's': 'BTCUSDT', 'k': {'t': 1719765540000, 'T': 1719765599999, 's': 'BTCUSDT', 'i': '1m', 'f': 5122041729, 'L': 5122041730, 'o': '61624.90', 'c': '61625.00', 'h': '61625.00', 'l': '61624.90', 'v': '0.026', 'n': 2, 'x': False, 'q': '1602.24770', 'V': '0.003', 'Q': '184.87500', 'B': '0'}}}
```

Each piece of data is parsed into a Python dictionary, and we need to convert this data dictionary into a DataFrame.

## Parsing Binance Market Data

According to the [Binance documentation](https://developers.binance.com/docs/derivatives/usds-margined-futures/websocket-market-streams/Kline-Candlestick-Streams), we can map the `k` field in the data dictionary to common K-line DataFrame column names.

```python
import pandas as pd

def convert_to_dataframe(x, interval_delta):
    """
    Parse WS returned data dictionary, return as DataFrame
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

    # Use K-line end time as the timestamp
    return pd.DataFrame(data=[candle_data], columns=columns, index=[candle_data[0] + interval_delta])
```

For robustness, we need to further adopt defensive programming, strictly checking data validity and determining if the K-line is closed, only accepting closed K-lines.

```python
def handle_candle_data(res, interval_delta):
    """
    Handle WS returned data
    """

    # Defensive programming, discard if Binance returns no data field
    if 'data' not in res:
        return

    # Extract data field
    data = res['data']

    # Defensive programming, discard if data does not contain e field or e field (data type) is not kline or data does not contain k field (K-line data)
    if data.get('e', None) != 'kline' or 'k' not in data:
        return

    # Extract k field, i.e., K-line data
    candle = data['k']

    # Determine if K-line is closed, discard if not closed
    is_closed = candle.get('x', False)
    if not is_closed:
        return

    # Convert K-line to DataFrame
    df_candle = convert_to_dataframe(candle, interval_delta)
    return df_candle
```

Based on the example code `ex2_parse_data.py`, we attempt to parse the K-line data received in the previous section (K-line data saved as `ex2_ws_candle.json`).

```python
import json
import pandas as pd

def main():
    # Load JSON data
    data = json.load(open('ex2_ws_candle.json'))

    # K-line interval is 1m
    interval_delta = pd.Timedelta(minutes=1)

    # Try to parse each WS data
    for idx, row in enumerate(data, 1):
        row_parsed = handle_candle_data(row, interval_delta)
        if row_parsed is None:
            print(f'Row{idx} is None')
        else:
            print(f'Row{idx} candlestick\n' + str(row_parsed))


if __name__ == '__main__':
    main()
```

The output is as follows:

```
Row1 is None
Row2 candlestick
                                  candle_begin_time     open     high      low    close  volume  quote_volume  trade_num  taker_buy_base_asset_volume  taker_buy_quote_asset_volume
2024-06-30 16:39:00+00:00 2024-06-30 16:38:00+00:00  61607.9  61624.9  61605.3  61624.9   16.71  1.029521e+06      418.0                       12.571                    774523.564
Row3 is None
```

The first and third records are discarded because the K-line is not closed, resulting in an output of None. 

The second record is a closed K-line and is parsed into a DataFrame.

## Single Time Interval Multiple Symbol WebSocket K-Line Data Receiver `CandleListener`

Combining the previous two sections, we can define `CandleListener` (`candle_listener.py`).

The main function is `start_listen`, which is responsible for establishing the WebSocket connection and receiving K-line data.

The `handle_candle_data` function is responsible for parsing the received K-line data, pushing valid and closed K-lines into the message queue (`self.que`) introduced in the next section.

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
    Parse the dictionary returned by WS and return a DataFrame
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

    # Use the K-line end time as the timestamp
    return pd.DataFrame(data=[candle_data], columns=columns, index=[candle_data[0] + interval_delta])


class CandleListener:

    # Mapping of trade types to ws functions
    TRADE_TYPE_MAP = {
        'usdt_futures': get_usdt_futures_multi_candlesticks_socket,
        'coin_futures': get_coin_futures_multi_candlesticks_socket,
        'spot': get_spot_multi_candlesticks_socket
    }

    def __init__(self, type_, symbols, time_interval, que):
        # Trade type
        self.trade_type = type_
        # Trading symbols
        self.symbols = set(symbols)
        # K-line period
        self.time_interval = time_interval
        self.interval_delta = convert_interval_to_timedelta(time_interval)
        # Message queue
        self.que: asyncio.Queue = que
        # Reconnection flag
        self.req_reconnect = False

    async def start_listen(self):
        """
        Main function for WS listening
        """

        if not self.symbols:
            return
        
        socket_func = self.TRADE_TYPE_MAP[self.trade_type]
        while True:
            # Create WS
            socket = socket_func(self.symbols, self.time_interval)
            async with socket as socket_conn:
                # After the WS connection is successful, receive and parse the data
                while True:
                    if self.req_reconnect:
                        # Reconnect if reconnection is required
                        self.req_reconnect = False
                        break
                    try:
                        res = await socket_conn.recv()
                        self.handle_candle_data(res)
                    except asyncio.TimeoutError: 
                        # Reconnect if no data is received for long (default 60 seconds)
                        # Normally K-line is pushed every 1-2 seconds                        
                        logging.error('Recv candle ws timeout, reconnecting')
                        break

    def handle_candle_data(self, res):
        """
        Handle data returned by WS
        """

        # Defensive programming, discard if Binance returns an error without the data field
        if 'data' not in res:
            return

        # Extract the data field
        data = res['data']

        # Defensive programming, discard if the data does not contain the e field or the e field (data type) is not kline or the data does not contain the k field (K-line data)
        if data.get('e', None) != 'kline' or 'k' not in data:
            return

        # Extract the k field, which is the K-line data
        candle = data['k']

        # Check if the K-line is closed, discard if not closed
        is_closed = candle.get('x', False)
        if not is_closed:
            return

        # Convert the K-line to a DataFrame
        df_candle = convert_to_dataframe(candle, self.interval_delta)

        # Put the K-line DataFrame into the communication queue
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

## Recording Binance K-Line Market Data

In this section, we asynchronously receive K-line data for spot, USDT-margined contracts, and coin-margined contracts through multiple WebSocket connections and store the K-line data DataFrame on the hard disk in Parquet format.

To achieve this, we first review the producer-consumer architecture: The producer-consumer architecture is a common software design pattern in concurrent scenarios, where producers provide data, consumers process data, and data is typically passed between producers and consumers through a message queue.

This pattern decouples data production from data processing. In our business scenario, using multiple producers and a single consumer ensures the correctness of hard disk writes.

In the example code `ex3_record_multi.py`, we define three producers, all of which are instances of `CandleListener`:

- `listener_usdt_perp_1m`: Receives 1-minute K-line data for the BTCUSDT and ETHUSDT USDT-margined contracts.
- `listener_coin_perp_3m`: Receives 3-minute K-line data for the BTCUSD_PERP and ETHUSD_PERP coin-margined contracts.
- `listener_spot_1m`: Receives 1-minute K-line data for the BTCUSDT and BNBUSDT spot pairs.

A consumer is defined to update the K-line data.

```python
def update_candle_data(df_new: pd.DataFrame, symbol, time_interval, trade_type):
    """
    Writes the received K-line data DataFrame to the hard disk in Parquet format.
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
    Consumer that processes the received K-line data.
    """
    while True:
        # Get data from the main queue
        req = await main_que.get()
        run_time = req['run_time']
        req_type = req['type']

        # Call the appropriate processing function based on the data type
        if req_type == 'candle_data':  # K-line data update
            symbol = req['symbol']
            time_interval = req['time_interval']
            trade_type = req['trade_type']
            update_candle_data(req['data'], symbol, time_interval, trade_type)
            logging.info('Record %s %s-%s at %s', trade_type, symbol, time_interval, run_time)
        else:
            logging.warning('Unknown request %s %s', req_type, run_time)
```

The core calling code is as follows:

```python
# ex3_record_multi.py

async def main():
    logging.info('Start recording candlestick data')
    # Main queue
    main_que = asyncio.Queue()

    # Producers
    listener_usdt_perp_1m = CandleListener('usdt_futures', ['BTCUSDT', 'ETHUSDT'], '1m', main_que)
    listener_coin_perp_3m = CandleListener('coin_futures', ['BTCUSD_PERP', 'ETHUSD_PERP'], '3m', main_que)
    listener_spot_1m = CandleListener('spot', ['BTCUSDT', 'BNBUSDT'], '1m', main_que)

    # Consumer
    dispatcher_task = dispatcher(main_que)

    await asyncio.gather(listener_usdt_perp_1m.start_listen(), listener_coin_perp_3m.start_listen(),
                         listener_spot_1m.start_listen(), dispatcher_task)
```

The runtime output is as follows:

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

The Parquet files written to the hard disk are as follows:

```
-rw-rw-r-- 1 admin admin 8.8K Jun 30 23:09 coin_futures_BTCUSD_PERP_3m.pqt
-rw-rw-r-- 1 admin admin 8.8K Jun 30 23:09 coin_futures_ETHUSD_PERP_3m.pqt
-rw-rw-r-- 1 admin admin 9.3K Jun 30 23:10 spot_BNBUSDT_1m.pqt
-rw-rw-r-- 1 admin admin 9.3K Jun 30 23:10 spot_BTCUSDT_1m.pqt
-rw-rw-r-- 1 admin admin 9.4K Jun 30 23:10 usdt_futures_BTCUSDT_1m.pqt
-rw-rw-r-- 1 admin admin 9.4K Jun 30 23:10 usdt_futures_ETHUSDT_1m.pqt
```

The recorded data examples are as follows:

```python
In [2]: pd.read_parquet('usdt_futures_BTCUSDT_1m.pqt')
Out[2]:
                                  candle_begin_time     open     high      low    close   volume  quote_volume  trade_num  taker_buy_base_asset_volume  taker_buy_quote_asset_volume
2024-06-30 15:00:00+00:00 2024-06-30 14:59:00+00:00  61580.2  61586.1  61580.2  61581.7   18.203  1.121016e+06      340.0                        6.931                  4.268353e+05
2024-06-30 15:01:00+00:00 2024-06-30 15:00:00+00:00  61581.8  61612.8  61581.7  61612.7   79.385  4.890301e+06     1015.0                       62.865                  3.872662e+06
......
2024-06-30 15:10:00+00:00 2024-06-30 15:09:00+00:00  61643.5  61643.5  61633.5  61636.9   35.319  2.176951e+06      530.0                       11.421                  7.039525e+05

In [3]: pd.read_parquet('coin_futures_ETHUSD_PERP_3m.pqt')
Out[3]:
                                  candle_begin_time     open     high      low    close   volume  quote_volume  trade_num  taker_buy_base_asset_volume  taker_buy_quote_asset_volume
2024-06-30 15:00:00+00:00 2024-06-30 14:57:00+00:00  3386.98  3387.00  3386.48  3386.98  54646.0    161.348842      259.0                      31434.0                     92.810019
2024-06-30 15:03:00+00:00 2024-06-30 15:00:00+00:00  3386.98  3389.36  3386.98  3389.35  29754.0     87.806150      275.0                      27826.0                     82.116929
2024-06-30 15:06:00+00:00 2024-06-30 15:03:00+00:00  3389.36  3389.79  3387.27  3388.39  39127.0    115.465249      363.0                       9255.0                     27.315708
2024-06-30 15:09:00+00:00 2024-06-30 15:06:00+00:00  3388.39  3390.59  3388.39  3390.22  15443.0     45.562138      220.0                       8133.0                     23.994293
```

At this point, we have essentially implemented a simple asynchronous Binance K-line data client.

However, creating a robust Binance real-time data client is not simple; more logic needs to be added to ensure data integrity and accuracy.

Stay tuned for subsequent technical reports.
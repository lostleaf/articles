# [共享K线] BMAC v1.0：基于 Asyncio 的币安全市场单周期实时 K线爬虫

Github 项目：[传送门](https://github.com/lostleaf/binance_market_async_crawler)

Binance Market Async Crawler, 简称 BMAC，（又）一个币安全市场实时 K线爬虫

原理为使用 Python Asyncio 轮询币安 rest api 获取实时K线，并存储为 pandas feather 文件格式

目前v1.0版本同时支持币U本位合约(即 dapi 和 fapi )

## 使用方法

入口文件为 `crawl_binance_swap`，使用方法为 `python crawl_binance_swap.py 运行目录`

目前项目中附带了两个使用案例，`usdt_5m_example` USDT 本位永续合约5分钟线案例，`coin_5m_example` 币本位永续合约5分钟线案例

将其中 `config.json.example` 重命名为 `config.json` 并填写 dingding token 即可使用

以 `usdt_5m_example` 为例，`config.json` 字段含义为

``` json
{
    // 运行周期，5分钟
    "interval": "5m", 

    // http 请求超时时间，3秒
    "http_timeout_sec": 3,
    
    // K 线闭合超时时间，12 秒，即周期开始 12 秒内会反复请求 rest api 直到 K 线闭合
    // 如果想速度快且不在意 K 线闭合可设置为 0，但钉钉会经常警告 K线未闭合，可将 crawler.py 中 msg['not_closed'] = list(may_not_closed) 删去
    "candle_close_timeout_sec": 12,
    
    // K 线类型，USDT 本位永续，目前支持 usdt_swap 和 coin_swap, coin_swap 为币本位永续
    "trade_type": "usdt_swap",
    
    // symbol 白名单，null 代表全市场
    "keep_symbols": null,
    
    // 不为空则只抓取给定的 symbol
    // "keep_symbols": ["BTCUSDT", "ETHUSDT"],

    // dingding 配置，不写则不发送 dingding 消息
    "dingding": {
        "error": {
            "access_token": "f06e5.....",
            "secret": "SEC439...."        
        }
    }
}
```

使用 `python crawl_binance_swap.py usdt_5m_example` 即可运行

运行时文件目录类似
```
usdt_5m_example
├── config.json 设置文件
├── config.json.example 非必要
├── exginfo_5m 本周期正在交易的 symbol 元数据，包括最小下单单位等
│   ├── exginfo_20230407_001500.ready
│   └── exginfo.fea
└── usdt_swap_5m USDT本位永续5分钟K线
    ├── 1000LUNCUSDT_20230407_001500.ready 
    ├── 1000LUNCUSDT.fea
    ├── 1000SHIBUSDT_20230407_001500.ready
    ├── 1000SHIBUSDT.fea
    ├── 1000XECUSDT_20230407_001500.ready
    ├── 1000XECUSDT.fea
    ├── 1INCHUSDT_20230407_001500.ready
    ├── 1INCHUSDT.fea
    ├── AAVEUSDT_20230407_001500.ready
    ├── AAVEUSDT.fea
    ├── ACHUSDT_20230407_001500.ready
    ├── ACHUSDT.fea
    ├── ADAUSDT_20230407_001500.ready
    ├── ...
```

## 代码结构

```
├── README.md 
├── candle_manager.py 存放 CandleFeatherManager 类代码
├── checker.py 一个简单的检查器，也可以参考编写策略
├── coin_5m_example 币本位 5 分钟案例
│   └── config.json.example 币本位配置文件案例
├── crawl_binance_swap.py 入口主程序
├── crawler.py 存放 Crawler 类代码
├── dingding.py 钉钉相关代码
├── market_api.py BinanceMarketApi及其子类代码
├── test_market.py BinanceMarketApi测试代码
├── usdt_5m_example USDT本位 5 分钟案例
│   └── config.json.example USDT本位配置文件案例
└── util.py 一些辅助函数
```

## 项目原理

BMAC 基本原理如下：

1. 封装币安公有 Rest api 为 `BinanceMarketApi` 类，基于 aiohttp 调用
2. 封装 Feather 文件读写及管理为 `CandleFeatherManager` 类
3. 基于以上两个类，将 K线实时获取封装为 `Crawler` 类
4. 将钉钉相关发消息接口调用封装为 `DingDingSender` 类
5. 入口文件 `crawl_binance_swap.py`，主要负责读取配置文件，初始化并调用 `Crawler`，出错将错误信息通过 `DingDingSender` 发送到钉钉

### BinanceMarketApi: 币安公有接口抽象

BinanceMarketApi 类为抽象类，主要对币安以下这三个接口进行封装，以及对api返回的数据进行解析：

1. 获取服务器时间及已消耗权重(`/time`)
2. 获取交易规则和交易对(`/exchangeInfo`)
3. 获取K线(`/klines`)

BinanceMarketApi 类定义如下：

``` python

class BinanceMarketApi(ABC):
    '''
    BinanceMarketApi 类为抽象类
    '''

    # 每次最多获取的K线数量
    MAX_ONCE_CANDLES = 1500

    # 每分钟权重上限
    MAX_MINUTE_WEIGHT = 2400

    def __init__(self, aiohttp_session, candle_close_timeout_sec):
        '''
        构造函数，接收 aiohttp Session 和K线闭合超时时间 candle_close_timeout_sec
        '''
        self.session = aiohttp_session
        self.candle_close_timeout_sec = candle_close_timeout_sec

    @abstractclassmethod
    def parse_syminfo(cls, info):
        '''
        抽象函数，解析 exchange info 中每个 symbol 交易规则，币U本位有所不同
        '''
        pass

    @abstractmethod
    async def aioreq_timestamp_and_weight(self) -> Tuple[int, int]:
        '''
        抽象函数, /time 接口具体 http 调用
        '''
        pass

    @abstractmethod
    async def aioreq_candle(self, symbol, interval, **kwargs) -> list:
        '''
        抽象函数, /klines 接口具体 http 调用
        '''
        pass

    @abstractmethod
    async def aioreq_exchange_info(self) -> dict:
        '''
        抽象函数, /exchangeInfo 接口具体 http 调用
        '''
        pass

    async def get_timestamp_and_weight(self) -> Tuple[int, int]:
        '''
        从 /time 接口的返回值中，解析出当前服务器时间和已消耗权重
        '''
        ...

    async def get_candle(self, symbol, interval, **kwargs) -> pd.DataFrame:
        '''
        从 /klines 接口返回值中，解析出K线数据并转换为 dataframe
        '''
        ...

    async def fetch_recent_closed_candle(self, symbol, interval, run_time, limit=5) -> Tuple[pd.DataFrame, bool]:
        '''
        获取 run_time 周期闭合K线，原理为反复获取K线，直到K线闭合或超时
        返回值为 tuple(K线df, 是否闭合布尔值)
        '''
        ...

    async def get_syminfo(self):
        '''
        从 /exchangeinfo 接口的返回值中，解析出当前每个symbol交易规则
        '''
        ...
```

`BinanceMarketApi` 派生出 USDT 本位 (fapi)类`BinanceUsdtFutureMarketApi` 和币本位 (dapi)类 `BinanceCoinFutureMarketApi`

由于币安交易所文档字段说明比较模糊，为了保证理解正确，我为 `BinanceMarketApi` 专门编写了简单的单元测试，`test_market.py`

### CandleFeatherManager: 文件系统及 DataFrame 读写抽象

`CandleFeatherManager` 类实现了 Feather 文件的管理，包括从 Feather 读取 K线，将 K线写入 Feather，以及相关文件锁的管理

`CandleFeatherManager` 类定义如下

``` python
class CandleFeatherManager:

    def __init__(self, base_dir):
        '''
        初始化，设定读写根目录
        '''
        self.base_dir = base_dir

    def clear_all(self):
        '''
        清空历史文件（如有），并创建根目录
        '''
        ...

    def format_ready_file_path(self, symbol, run_time):
        '''
        获取 ready file 文件路径, ready file 为每周期 K线文件锁
        ready file 文件名形如 {symbol}_{runtime年月日}_{runtime_时分秒}.ready
        '''
        ...

    def set_candle(self, symbol, run_time, df: pd.DataFrame):
        '''
        设置K线，首先将新的K线 DataFrame 写入 Feather，然后删除旧 ready file，并生成新 ready file
        '''
        ...

    def update_candle(self, symbol, run_time, df_new: pd.DataFrame):
        '''
        使用新获取的K线，更新 symbol 对应K线 Feather，主要用于每周期K线更新
        '''
        ...

    def check_ready(self, symbol, run_time):
        '''
        检查 symbol 对应的 ready file 是否存在，如存在，则表示 run_time 周期 K线已获取并写入 Feather
        '''
        ...

    def read_candle(self, symbol) -> pd.DataFrame:
        '''
        读取 symbol 对应的 K线
        '''
        ...

    def has_symbol(self, symbol) -> bool:
        '''
        检查某 symbol Feather 文件是否存在
        '''
        ...

    def remove_symbol(self, symbol):
        '''
        移除 symbol，包括删除对应的 Feather 文件和 ready file
        '''
        ...

    def get_all_symbols(self):
        '''
        获取当前所有 symbol
        '''
        ...
```

### Crawler: K线爬虫业务逻辑

`Crawler` 类为 K线爬虫主要业务逻辑实现，其构造函数如下

``` python
class Crawler:

    def __init__(self, interval, exginfo_mgr, candle_mgr, market_api, symbol_filter):
        '''
        interval: K线周期
        exginfo_mgr: 用于管理 exchange info（合约交易规则）的 CandleFeatherManager
        candle_mgr: 用于管理 K线的 CandleFeatherManager
        market_api: BinanceMarketApi 的子类，用于请求币本位或 USDT 本位合约公有 API
        symbol_filter: 仿函数，用于过滤出 symbol

        初始化阶段，exginfo_mgr 和 candle_mgr，会清空历史数据并建立数据目录
        '''
        self.interval = interval
        self.market_api: BinanceMarketApi = market_api
        self.candle_mgr: CandleFeatherManager = candle_mgr
        self.exginfo_mgr: CandleFeatherManager = exginfo_mgr
        self.symbol_filter = symbol_filter
        self.exginfo_mgr.clear_all()
        self.candle_mgr.clear_all()
```

其中 symbol_filter 为仿函数 (Functor)，以 USDT本位永续合约举例，其对应的 Filter 为

``` python
class TradingUsdtSwapFilter:

    def __init__(self, keep_symbols=None):
        self.keep_symbols = set(keep_symbols) if keep_symbols else None

    @classmethod
    def is_trading_usdt_swap(cls, x):
        '''
        筛选出所有正在被交易的(TRADING)，USDT本位的，永续合约（PERPETUAL）
        '''
        return x['quote_asset'] == 'USDT' and x['status'] == 'TRADING' and x['contract_type'] == 'PERPETUAL'

    def __call__(self, syminfo: dict) -> list:
        symbols = [info['symbol'] for info in syminfo.values() if self.is_trading_usdt_swap(info)]
        if self.keep_symbols is not None:  # 如有白名单，则只保留白名单内的
            symbols = [sym for sym in symbols if sym in self.keep_symbols]
        return symbols
```

`TradingUsdtSwapFilter` 会筛选出所有正在交易的U本位永续合约，如有白名单 `keep_symbols`，则只保留白名单内的 symbol 

通过扩展 BinanceMarketApi 和 Filter，理论上可以组合出不同本位不同类型合约的过滤器，例如U本位永续，U本位交割，币本位永续，币本位交割等

`Crawler` 又可分为两个阶段

#### 初始化历史阶段 init_history

``` python

async def init_history(self):
    '''
    初始化历史阶段 init_history
    1. 通过调用 self.market_api.get_syminfo 获取所有交易的 symbol, 并根据 symbol_filter 过滤出我们想要的 symbol
    2. 通过调用 self.market_api.get_candle，请求每个 symbol 最近 1500 根K线（币安最大值）
    3. 将每个 symbol 获取的 1500 根近期的 K线通过 self.candle_mgr.set_candle 写入文件
    '''
    ...
```

#### 定时获取 K线阶段 run_loop
``` python
async def run_loop(self):
    '''
    定时获取 K线 run_loop
    1. 计算出 self.interval 周期下次运行时间 run_time, 并 sleep 到 run_time
    2. 通过调用 self.market_api.get_syminfo 获取所有交易的 symbol 及交易规则, 并根据 symbol_filter 过滤出我们想要的 symbol
    3. 删除之前有交易，但目前没有交易的 symbol（可能可以防止 BNXUSDT 拆分之类的事件），这些停止交易的 symbol 会发送钉钉警告
    4. 对所有这在交易的 symbol, 调用 self.market_api.fetch_recent_closed_candle 获取最近 5 根 K线
    5. 将获取的 K线通过 self.candle_mgr.update_candle 写入 feather，并更新 ready file，未闭合 K线也会被写入，并发送钉钉警告
    '''
    ...
```

### crawl_binance_swap 入口函数

入口函数主要负责解析配置文件，实例化以上涉及的类，并完成函数调用，核心伪代码如下

``` python

MARKET_API_DICT = {'usdt_swap': BinanceUsdtFutureMarketApi, 'coin_swap': BinanceCoinFutureMarketApi}
SYMBOL_FILTER_DICT = {'usdt_swap': TradingUsdtSwapFilter, 'coin_swap': TradingCoinSwapFilter}

async def main(argv):
    #从 argv 中获取根目录
    base_dir = argv[1]

    # 读取 config.json，获取配置
    cfg = json.load(open(os.path.join(base_dir, 'config.json')))

    interval = cfg['interval']
    ...

    while True:
        try:
            async with create_aiohttp_session(http_timeout_sec) as session:
                # 实例化所有涉及的类
                market_api = market_api_cls(session, candle_close_timeout_sec)
                ...
                crawler = Crawler(interval, exginfo_mgr, candle_mgr, market_api, symbol_filter)

                # 首先获取历史数据
                await crawler.init_history()

                # 无限循环，每周期获取最新K线
                while True:
                    msg = await crawler.run_loop()
                    if msg and msg_sender: # 如有合约停止交易或有合约当周期K线没有闭合，则报错
                        msg['localtime'] = str(now_time())
                        await msg_sender.send_message(json.dumps(msg, indent=1), 'error')
        except Exception as e:
            # 出错则通过钉钉报错
            ...
```

## 运行示例

参照 usdt_5m_example/config.json.example 编写 config.json，并运行 `python crawl_binance_swap.py usdt_5m_example`

会输出类似如下 log
```
> python crawl_binance_swap.py usdt_5m_example
20230409 21:59:08 (INFO) - Saved symbols: 0, Server time:, 2023-04-09 21:59:08.672000+08:00, Used weight: 1
20230409 21:59:09 (INFO) - Saved symbols: 20, Server time:, 2023-04-09 21:59:09.120000+08:00, Used weight: 202
20230409 21:59:09 (INFO) - Saved symbols: 40, Server time:, 2023-04-09 21:59:09.582000+08:00, Used weight: 403
20230409 21:59:10 (INFO) - Saved symbols: 60, Server time:, 2023-04-09 21:59:09.982000+08:00, Used weight: 604
20230409 21:59:10 (INFO) - Saved symbols: 80, Server time:, 2023-04-09 21:59:10.466000+08:00, Used weight: 805
20230409 21:59:10 (INFO) - Saved symbols: 100, Server time:, 2023-04-09 21:59:10.961000+08:00, Used weight: 1006
20230409 21:59:11 (INFO) - Saved symbols: 120, Server time:, 2023-04-09 21:59:11.383000+08:00, Used weight: 1207
20230409 21:59:11 (INFO) - Saved symbols: 140, Server time:, 2023-04-09 21:59:11.806000+08:00, Used weight: 1408
20230409 21:59:12 (INFO) - Saved symbols: 160, Server time:, 2023-04-09 21:59:12.249000+08:00, Used weight: 1609
20230409 21:59:12 (INFO) - Saved symbols: 175, Server time:, 2023-04-09 21:59:12.602000+08:00, Used weight: 1760
20230409 21:59:12 (INFO) - Next candle crawler run at 2023-04-09 22:00:00+08:00
20230409 22:00:10 (INFO) - Saved symbols: 175. Server time: 2023-04-09 22:00:10.042000+08:00, used weight: 590
20230409 22:00:10 (INFO) - Next candle crawler run at 2023-04-09 22:05:00+08:00
20230409 22:05:08 (INFO) - Saved symbols: 175. Server time: 2023-04-09 22:05:08.015000+08:00, used weight: 563
20230409 22:05:08 (INFO) - Next candle crawler run at 2023-04-09 22:10:00+08:00
20230409 22:10:07 (INFO) - Saved symbols: 175. Server time: 2023-04-09 22:10:07.671000+08:00, used weight: 542
20230409 22:10:07 (INFO) - Next candle crawler run at 2023-04-09 22:15:00+08:00
20230409 22:15:07 (INFO) - Saved symbols: 175. Server time: 2023-04-09 22:15:07.927000+08:00, used weight: 579
20230409 22:15:07 (INFO) - Next candle crawler run at 2023-04-09 22:20:00+08:00
```

首先在初始化历史数据阶段，会分批获取历史数据，并消耗较多权重，如果在此过程中消耗权重接近2400，则会等到下一分钟再继续初始化

然后每次K线更新循环，会请求每个 symbol 闭合 K线，耗时 7-10 秒，消耗不到 600 权重；如果为了速度不要求 K线闭合，则这一步仅需 2 秒左右

## 检查器 checker.py

为了检验数据正确性我还额外编写了检查器 `checker.py`

1. 检查每周期 exginfo 及各 symbol K线是否及时就绪
2. 检查是不是 exginfo 中包含的每一个 symbol 都存储了 K线
3. 检查每个 symbol K线的正确性

我运行了一段时间没发现什么问题，欢迎各位老板帮忙 review 及提交 pull request

## 总结

BMAC 是一个基于 python asyncio 和 pandas dataframe 的币安共享K线解决方案，主要通过 aiohttp 来并发请求币安 K线 api，并通过将 K线数据存储为 Pandas Feather 格式，供其他策略读取

目前 v1.0 版本，BMAC 同时支持币本位和U本位合约，可用于支持中性策略全市场U本位合约选币，以及U本位/币本位单币择时策略
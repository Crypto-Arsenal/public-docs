## Developing My Strategy in Python

# 基本結構

- 使用 python 3 撰寫 Strategy Class
- 系統透過固定介面呼叫，傳入市場資料，策略須回傳是否執行訂單
- 策略須自行維護訂單生命週期
- 可以使用 [TA-LIB](https://github.com/acrazing/talib-binding-node) 計算常見技術指標，使用 TA 存取
- 可以使用 Log(str) 紀錄運行資訊

# 策略

```python
# Class name must be Strategy
class Strategy():
    # option setting needed
    def __setitem__(self, key, value):
        self.options[key] = value

    # option setting needed
    def __getitem__(self, key):
        return self.options.get(key, '')

    def __init__(self):
        # strategy property needed
        self.subscribedBooks = {
            'Bitfinex': {
                'pairs': ['ETH-USDT'],
            },
        }

        # seconds for broker to call trade()
        # do not set the frequency below 60 sec.
        # 10 * 60 for 10 mins
        self.period = 10 * 60
        self.options = {}

        # user defined class attribute
        self.last_type = 'sell'

	
    # called every self.period
    def trade(self, information):
        # for single pair strategy, user can choose which exchange/pair to use when launch, get current exchange/pair from information
        exchange = list(information['candles'])[0]
        pair = list(information['candles'][exchange])[0]
        if self.last_type == 'sell':
            self.last_type = 'buy'
            return [
                {
                    'exchange': exchange,
                    'amount': 1,
                    'price': -1,
                    'type': 'MARKET',
                    'pair': pair,
                },
            ]
        else:
            self.last_type = 'sell'
            return [
                {
                    'exchange': exchange,
                    'amount': -1,
                    'price': -1,
                    'type': 'MARKET',
                    'pair': pair,
                },
            ]
        return []
	

    def on_order_state_change(self, order):
        Log("on order state change message: " + str(order) + " order price: " + str(order["price"]))

```

## 架構

### Candle

系統呼叫 strategy 時，共會傳入一個參數並放次於第一個參數，此參數後續以 information 命名解釋． 傳入的資訊分為3大類，分別為 candles(K線圖資訊)， orders(訂單），以及 orderBooks(訂單簿)． 上述資訊皆放置於information物件中，取用方法為:

```python
information['candles']
information['orders']
information['orderBooks']
```

Candle 會以 array of object 的形式傳入上次呼叫到此次呼叫的 K 線圖資訊，共含有6項資訊 open、close、high、low、volume和time，取用方式如下:

```python
information['candles'][exchange][pair][0]['open']
information['candles'][exchange][pair][0]['close']
information['candles'][exchange][pair][0]['high']
information['candles'][exchange][pair][0]['low']
information['candles'][exchange][pair][0]['volume']
information['candles'][exchange][pair][0]['time']
```

### 交易所 Exchange

策略使用者可於使用策略時決定使用的 exchange 交易所 (例如 Bitfinex), 這時此設定會被忽略，以使用者執行策略時選擇為主,程式內請使用 information 取得當前使用的 exchange 進行交易

```python
exchange = list(information['candles'])[0]
```

### 交易對 Pair

Crypto-Arsenal 支援單一 Strategy 同時註冊多組交易 pair，因此 candle 與 orderBooks 會同時傳入多組 pair 的資訊，策略使用者可於使用策略時決定使用的 pair(例如 BTC-USDT), 這時程式內設定的 pair 會被忽略，以使用者執行策略時選擇為主,程式內請使用 information 取得當前使用的 pair 進行交易

```python
pair = list(information['candles'][exchange])[0]
```

### Assets

策略會擁有隨時更新的資產 (assets) 資訊，即當前可進行操作的貨幣，取用方式如下:

```python
  pair = list(information['candles'][exchange])[0] #BTC-USDT
  target_currency = pair.split('-')[0]  #BTC
  base_currency = pair.split('-')[1]  #USDT

  base_currency_amount = self['assets'][exchange][base_currency]
  target_currency_amount = self['assets'][exchange][target_currency]
```

### Orders

使用information['orders']取得訂單資訊，以 object of array of object 的形式儲存，第一筆訂單為information['orders'][0]，訂單內的資料包含 price, amount, status和orderId.
其中status可能為FILLED或NEW，分別代表成交或尚未成交．
取用範例如下:

```python
  for i in range(len(information['orders'])):
    Log('order price: ' + str(information['orders'][i]['price']) +
    ' ,order amount: ' + str(information['orders'][i]['amount']) + 
    ' ,status: ' + str(information['orders'][i]['status']) + 
    ' ,orderId: ' + str(information['orders'][i]['orderId']))   

```
刪除訂單的範例如下:
```python
  CancelOrder(information['orders'][0])
```
### Order Books

Order Books 是在實際交易時才會使用，在回測和競技場比賽都用不到．
在執行 Backtest 模式中，不會傳入 Order books 資訊．
orderBooks 會以 object of array of object 的形式傳入，將傳入目前交易所中此 pair 的訂單資訊，分成兩大類 asks 與 bids，兩類中皆有相同的三項資料格式 price、count、amount，取用方式如下:

```python
  order_books = information['orderBooks']
  one_order_book = order_books[exchange][pair]

  ask_order_book = one_order_book['asks']
  last_ask_price = ask_order_book[-1]['price']
  last_ask_count = ask_order_book[-1]['count']
  last_ask_amount = ask_order_book[-1]['amount']

  bid_order_book = one_order_book['bids']
  last_bid_price = bid_order_book[-1]['price']
  last_bid_count = bid_order_book[-1]['count']
  last_bid_amount = bid_order_book[-1]['amount']

```

## 回傳值 (trade)

在 Crypto-Arsenal 的策略中，要向交易所下訂單的方式採用系統呼叫`trade`的回傳值進行交易。

```python
def trade(self, information):
	[
	  {
		'exchange': string
		'pair': string
		'type': string
		'amount': float
		'price': float
	  }
	]
```

### 欄位說明

- exchange: 此筆訂單要向哪個交易所進行交易
- pair: 交易貨幣對
- type: 此筆訂單採用何種交易方式 (限價: `LIMIT` 市價: `MARKET`)
- amount: 要交易的數位貨幣數量，大於 0 代表購買，小於 0 代表販賣
- price: 以多少價格進行交易，若採用市價，此欄位請填入隨意值

### 範例

#### 限價

此訂單為欲向`Binance`交易所，以每單位`212.42`購買`1`個`ETH-USDT`的交易配對

```python
[
  {
    'exchange': 'Binance',
    'pair': 'ETH-USDT',
    'type': 'LIMIT',
    'amount': 1,
    'price': 212.42
  }
]
```

#### 市價

此訂單為欲向`Binance`交易所，以`市價`購買`1`個`ETH-USDT`的交易配對

**雖然交易採用市價，price 仍須存在，值為隨意數值**

```python
[
  {
    'exchange': 'Binance',
    'pair': 'ETH-USDT',
    'type': 'MARKET',
    'amount': 1,
    'price': -1
  }
]
```

# Helper Functions

## Log

用來印出指定訊息

```python
Log(str);
```

最大長度為 200 個字串

### 範例

印出收盤價、開盤價與交易量

```python
exchange = list(information['candles'])[0] //Bitfinex
pair = list(information['candles'][exchange])[0] //BTC-USDT
Log(information['candles'][exchange][pair][0]['close'])
Log(information['candles'][exchange][pair][0]['open'])
Log(information['candles'][exchange][pair][0]['volume'])
```

### 範例

印出擁有的資產'USDT'和'BTC'

```python
Log('assest usdt: ' + str(self['assets'][exchange]['USDT']))
Log('assest btc: ' + str(self['assets'][exchange]['BTC']))
```

## on_order_state_change()

回傳每一筆訂單紀錄，結構如下

```python
{'exchange': 'Bitfinex', 'pair': 'ETH-USDT', 'amount': 1.0, 'amountFilled': 1.0, 'price': 1849.7, 'type': 'MARKET', 'status': 'FILLED'}
```

### 範例

取回訂單資訊，並印出成交價資訊
需 define 一個函數 on_order_state_change，平台會依照使用者定義之週期呼叫此函數，並印出相關資訊。

```python
    def on_order_state_change(self,  order):
        Log("on order state change message: " + str(order) + " order price: " + str(order["price"]))
```

## GetLastOrderSnapshot

回傳最後一次成交的訂單資訊，結構如下

```python
{'exchange': 'Bitfinex', 'pair': 'ETH-USDT', 'amount': 1.0, 'price': 182.39, 'type': 'MARKET'}
```

### 範例

取回最後一次掛單紀錄，並印出價量資訊。與 on_order_state_change 不同，GetLastOrderSnapshot 函數是在 trade 函數當中呼叫。

```python
snap = GetLastOrderSnapshot()
if snap != {}:
    Log( 'snap_amount: ' + str(snap['amount']) + ' ,snap_price: ' + str(snap['price']))
```

## 存取策略參數

透過 `self['OPTION_NAME']` 存取策略參數

### 範例

當使用者自定義參數 R1，使用以下方法可在程式內取回所定義的 R1 數值
![image](https://drive.google.com/uc?export=view&id=16-bHa0jOZvPerOqEdIKRwDXlH50MHplr)

```python
def trade(self, information):
    R1 = float(self['R1'])
```

## 空單交易、手續費與滑價調整

新增策略參數 is_shorting, exchange_fee, spread

### 範例

使用者透過將 is_shorting 參數設定為 true，可開啟空單交易模式  
另可調整 exchange_fee 以及 spread，以設定更嚴格的回測條件
![image2](https://drive.google.com/uc?export=view&id=1IWJoekgYPgQWfxfLv_DZ3KjtTkuIeA4L)
上圖所示，exchange_fee 設定為 0.01，代表每一筆交易會加收總額 1%費用
（若沒有設定，預設是 0.1%的手續費）

spread 代表滑點，0.05 代表每筆交易價格會有 5%差異

## 進階用法

可使用 np 存取 [numpy](http://www.numpy.org/)  
可使用 talib 存取 [talib](https://github.com/mrjbq7/ta-lib)

### 範例

使用 np.append，使用 talib 計算 RSI

```python
self.close_price_trace = np.append(self.close_price_trace, [float( information['candles'][exchange][pair][0]['close'])])
rsi = talib.RSI(self.close_price_trace, Len)[-1]
```

### 黃金交叉策略範例

```python
class Strategy():
    # option setting needed
    def __setitem__(self, key, value):
        self.options[key] = value

    # option setting needed
    def __getitem__(self, key):
        return self.options.get(key, '')

    def __init__(self):
        # strategy property
        self.subscribedBooks = {
            'Bitfinex': {
                'pairs': ['ETH-USDT'],
            },
        }
        self.period = 10 * 60
        self.options = {}

        # user defined class attribute
        self.last_type = 'sell'
        self.last_cross_status = None
        self.close_price_trace = np.array([])
        self.ma_long = 10
        self.ma_short = 5
        self.UP = 1
        self.DOWN = 2

    def on_order_state_change(self,  order):
        Log("on order state change message: " + str(order) + " order price: " + str(order["price"]))

    def get_current_ma_cross(self):
        s_ma = talib.SMA(self.close_price_trace, self.ma_short)[-1]
        l_ma = talib.SMA(self.close_price_trace, self.ma_long)[-1]
        if np.isnan(s_ma) or np.isnan(l_ma):
            return None
        if s_ma > l_ma:
            return self.UP
        return self.DOWN


    # called every self.period
    def trade(self, information):
        exchange = list(information['candles'])[0]
        pair = list(information['candles'][exchange])[0]
        target_currency = pair.split('-')[0]  #ETH
        base_currency = pair.split('-')[1]  #USDT
        base_currency_amount = self['assets'][exchange][base_currency] 
        target_currency_amount = self['assets'][exchange][target_currency] 
        # add latest price into trace
        close_price = information['candles'][exchange][pair][0]['close']
        self.close_price_trace = np.append(self.close_price_trace, [float(close_price)])
        # only keep max length of ma_long count elements
        self.close_price_trace = self.close_price_trace[-self.ma_long:]
        # calculate current ma cross status
        cur_cross = self.get_current_ma_cross()
        if cur_cross is None:
            return []
        if self.last_cross_status is None:
            self.last_cross_status = cur_cross
            return []
        # cross up
        if self.last_type == 'sell' and cur_cross == self.UP and self.last_cross_status == self.DOWN:
            Log('buying 1 unit of ' + str(target_currency))
            self.last_type = 'buy'
            self.last_cross_status = cur_cross
            return [
                {
                    'exchange': exchange,
                    'amount': 1,
                    'price': -1,
                    'type': 'MARKET',
                    'pair': pair,
                }
            ]
        # cross down
        elif self.last_type == 'buy' and cur_cross == self.DOWN and self.last_cross_status == self.UP:
            Log('assets before selling: ' + str(self['assets'][exchange][base_currency]))
            self.last_type = 'sell'
            self.last_cross_status = cur_cross
            return [
                {
                    'exchange': exchange,
                    'amount': -target_currency_amount,
                    'price': -1,
                    'type': 'MARKET',
                    'pair': pair,
                }
            ]
        self.last_cross_status = cur_cross
        return []

```

## Binary Code

使用 python 開發時，可以選擇上傳 binary code 保護原始碼
請務必使用 python 3.6 版本進行編譯

1. 請先以 `python3 -m py_compile strategy.py` 將原始碼編譯
2. 至 `./__pycache__ ` 中找到 `strategy.cpython-36.pyc`
3. 到 My Strategy 的 Edit Strategy 頁面選擇 Use Binary Code 後上傳。

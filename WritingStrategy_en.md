# Developing Your Strategy with JS

## What You Need To Know...
* Get started to develop your Strategy Class by using Nodejs in compliance with ES6.
* There are dedicated interfaces to be coded along, please refer to 'Basic Coding Structure' as below; note that your strategy takes in current market trading data and must return results whether your trades are executed or not.
* The life cycle of your trades/orders are maintained by your own strategy.
* You can leverage off-the-shelf technical analysis indicators of [TA-LIB](https://github.com/acrazing/talib-binding-node) for quickly develop your strategies.
* You can record your trading data with `Log(str)`.

## Create Your Strategy
### Class Structure
``` javascript
class BuyOneSellOneMarket {

  trade(information) {
    Log("Start strategy!");
    const exchange = Object.keys(information.candles)[0];
    const pair = Object.keys(information.candles[exchange])[0];

    if (this.lastOrderType === 'sell') {
      this.lastOrderType = 'buy';
      return [
        {
          exchange: exchange,
          pair: pair,
          type: 'MARKET',
          amount: 1,
          price: -1
        }
      ]
    } else {
      this.lastOrderType = 'sell';
      return [
        {
          exchange: exchange,
          pair: pair,
          type: 'MARKET',
          amount: -1,
          price: -1
        }
      ]
    }
  }

  // must have, monitoring life cycle of order you made
  onOrderStateChanged(state) {
    Log('onOrderStateChanged');
  }

  constructor() {
    // must have for developer
    this.subscribedBooks = {
      'Bitfinex': {
        pairs: [ 'BTC-USDT' ]
      },
    };

    // seconds for broker to call trade()
    // do not set the frequency below 60 sec.
    // 60 * 30 for 30 mins
    this.period = 60 * 30;

    // must have
    // assets should be set by broker
    this.assets = undefined;

    // customizable properties
    // sell price - buy price
    this.minEarnPerOp = 1;

    // other properties, your code
    this.lastOrderType = 'sell';
  }
}

```

## 傳入資料結構 ( trade() )
系統呼叫strategy時，共會傳入一個參數並放次於第一個參數，此參數後續以`information`命名解釋．
傳入的資訊分為兩大項，其一為`candle`(`K線圖資訊`)，另一項為`orderBooks`(`訂單狀態`)．
上述兩項資訊將放置於一個object中，取用方法為:

``` javascript
  const candle = information.candle;
  const orderBooks = information.orderBooks;
```

Crypto-Arsenal支援單一Strategy同時註冊多組交易pair，因此`candle`與`orderBooks`會同時傳入多組pair的資訊，如需取用請以下列方式取用，
其中`exchange`與`pair`為在strategy constructor中的`this.subscribedBooks`向系統註冊的pair
``` javascript
  const candle = information.candle;
  const oneCandle = candle[exchange][pair];

  const orderBooks = information.orderBooks;
  const oneOrderBook = orderBooks[exchange][pair];
```

Single Pair 策略使用者可於使用策略時決定使用的 pair, 這時此設定會被忽略，以使用者執行策略時選擇為主，下單時請從 information 取得當前使用的 pair
``` javascript
  // Correct way to get exchange / pair in single pair strategy
  const exchange = Object.keys(information.candles)[0];
  const pair = Object.keys(information.candles[exchange])[0];
  const candleData = information.candles[exchange][pair][0];
```

### Candle
Candle會以array of object的形式傳入上次呼叫到此次呼叫的K線圖資訊，共含有五項資訊 `open`、`close`、`high`、`low`、`volume`，取用方式如下:
``` javascript
  const candles = information.candles;
  const oneCandle = candles[exchange][pair][0];

  const open = oneCandle.open;
  const close = oneCandle.close;
  const high = oneCandle.high;
  const low = oneCandle.low;
  const volume = oneCandle.volume;
```

### Order Books
Order Books會以object of array of object的形式傳入，將傳入目前交易所中此pair的訂單資訊，分成兩大項`asks`與`bids`，兩項中皆有相同的三項資料格式`price`、`count`、`amount`，取用方式如下:
**在執行Backtest模式中，不會傳入Order books資訊**
``` javascript
  const orderBooks = information.orderBooks;
  const oneOrderBook = orderBooks[exchange][pair];

  const askOrderBook = oneOrderBook.asks;
  for (const askOrder of askOrderBook) {
    const price = askOrder.price;
    const count = askOrder.count;
    const amount = askOrder.amount;
  }

  const bidOrderBook = oneOrderBook.bids;
    for (const bidOrder of bidOrderBook) {
      const price = bidOrder.price;
      const count = bidOrder.count;
      const amount = bidOrder.amount;
    }
```

### Assets
策略會擁有隨時更新的資產 (assets) 資訊，即當前可進行操作的貨幣，取用方式如下:
```javascript
  const usd = this.assets[exchange]['USDT'];
  const eth = this.assets[exchange]['ETH'];
```

## 回傳值 (trade)
在Crypto-Arsenal的策略中，要向交易所下訂單的方式採用系統呼叫`trade`的回傳值進行交易。
### 基本格式
``` javascript
[
  {
    exchange: string
    pair: string
    type: string
    amount: float
    price: float
  },
  {
    exchange: string
    pair: string
    type: string
    amount: float
    price: float
  } ....
]
```
回傳值為一個含有0-n個object的array，

### 欄位說明

* exchange: 此筆訂單要向哪個交易所進行交易
* pair: 交易pair為何
* type: 此筆訂單採用何種交易方式 (限價: `LIMIT` 市價: `MARKET`)
* amount: 要交易的數位貨幣數量，大於0代表購買，小於0代表販賣
* price: 以多少價格進行交易，若採用市價，此欄位請填入隨意值

### 範例
#### 限價
此訂單為欲向`Binance`交易所，以每單位`214.42`購買`1`個`ETH-USDT`的交易配對

``` javascript
[
  {
    exchange: 'Binance',
    pair: 'ETH-USDT',
    type: 'LIMIT',
    amount: 1,
    price: 212.42
  }
]
```

#### 市價
此訂單為欲向`Binance`交易所，以`市價`購買`1`個`ETH-USDT`的交易配對

**雖然交易採用市價，price仍須存在，值為隨意數值**
``` javascript
[
  {
    exchange: 'Binance',
    pair: 'ETH-USDT',
    type: 'MARKET',
    amount: 1,
    price: -1
  }
]
```

# Helper Functions
## Log
Used to log the message.
``` javascript
Log(str);
```
Must pass in javascript's `string` type.
Maximum length of string is 200 characters

# 存取策略參數
使用 ```this.xxx``` 或是 ```this['OPTION_NAME']``` 存取策略參數

# 更多範例

此策略使用[TA-LIB](https://github.com/acrazing/talib-binding-node)計算兩條指數平均線(快線與慢線)，並當發生黃金交叉(快線往上穿過慢線)時買入，死亡交叉(快線往下穿破慢線)時賣出。
``` javascript
class EMACross {
  // must have
  // function for trading strategy
  // return order object for trading or null for not trading
  trade(information) {
    // define your own trading strategy here

    // exchange may offline
    if (!information.candles) return [];


    const exchange = Object.keys(information.candles)[0];
    const pair = Object.keys(information.candles[exchange])[0];
    const baseCurrency = pair.split('-')[1]; // pair must in format '{TARGET}-{BASE}', eg. BTC-USDT, ETH-BTC
    const currency = pair.split('-')[0]; // pair must in format '{TARGET}-{BASE}', eg. BTC-USDT, ETH-BTC

    if (!information.candles[exchange][pair]) return [];

    // information like
    const candleData = information.candles[exchange][pair][0];

    // keep track history data
    this.history.push({
      Time: candleData.time,
      Open: candleData.open,
      Close: candleData.close,
      High: candleData.high,
      Low: candleData.low,
      Volumn: candleData.volumn,
    });

    let lastPrice = (information.candles[exchange][pair][0]['close']);
    if (!lastPrice) return [];

    // release old data
    if (this.history.length > this.long) {
      this.history.shift();
    } else {
      return [];
    }

    const marketData = this.history;
    // Calculate EMA with TA-LIB, return type is [double], get last MA value by pop()
    const MAShort = TA.EMA(marketData, 'Close', this.short).pop();
    const MALong = TA.EMA(marketData, 'Close', this.long).pop();

    // Track cross
    const curSide = (MAShort > MALong) ? ('UP') : ('DOWN');
    Log("MA side: " + curSide);
    if(!this.preSide) {
      this.preSide = curSide;
      return [];
    }

    // When up cross happend
    if (this.phase == this.PHASES.waitBuy && this.preSide == 'DOWN' && curSide == 'UP') {
      // Not enough assets, we can't buy
      if (this.assets[exchange][baseCurrency] < lastPrice) {
        return [];
      }
      this.preSide = curSide;
      this.phase = this.PHASES.waitSell;
      // Buy 1 coin
      return [
        {
          exchange: exchange,
          pair: pair,
          type: 'LIMIT',
          amount: 1, // [CHANGE THIS] 一次買入多少
          price: lastPrice
        }
      ];
    }
    if (this.phase == this.PHASES.waitSell && this.preSide == 'UP' && curSide == 'DOWN') {
      this.preSide = curSide;
      this.phase = this.PHASES.waitBuy;
      // Sell all remaining coin
      return [
        {
          exchange: exchange,
          pair: pair,
          type: 'LIMIT',
          amount: -this.assets[exchange][currency], // 一次賣出所有擁有的
          price: lastPrice
        }
      ];
    }

    this.preSide = curSide;
    return [];
  }

  // must have
  onOrderStateChanged(state) {
    Log('order change' + JSON.stringify(state));
  }

  constructor() {
    // must have for developer
    this.subscribedBooks = {
      'Binance': {
        pairs: ['ETH-USDT']
      },
    };

    // seconds for broker to call trade()
    // do not set the frequency below 60 sec.
    this.period = 60 * 60; // [CHANGE THIS] 設定均線資料點的時間間隔，即均線週期
    // must have
    // assets should be set by broker
    this.assets = undefined;

    // customizable properties
    this.long = 10; // [CHANGE THIS] 設定均線交叉的慢線週期
    this.short = 5; // [CHANGE THIS] 設定均線交叉的快線週期
    this.PHASES = {
      init: 0,
      waitBuy: 2,
      waitSell: 4
    };

    this.preSide = undefined;

    this.phase = this.PHASES.waitBuy;

    this.history = [];
  }
}
```

* 調整參數

![](https://i.imgur.com/y82aDps.jpg)

![](https://i.imgur.com/VGIhd4z.png)

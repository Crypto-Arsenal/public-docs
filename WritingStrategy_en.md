# Developing Your Strategy with JS

## What You Need To Know...
* Get started to develop your Strategy Class by using Nodejs in compliance with ES6.
* There are dedicated interfaces to be coded along, please refer to 'Basic Coding Structure' as below; note that your strategy takes in current market trading data and must return results whether your trades are executed or not.
* The life cycle of your trades/orders are maintained by your own strategy.
* You can leverage off-the-shelf technical analysis indicators of [TA-LIB](https://github.com/acrazing/talib-binding-node) for quickly develop your strategies.
* You can record your trading data with `Log(str)`.

## Create Your Strategy
### Basic Coding Structure
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
### Carried-in Data in `trade()`
When a strategy is called, `information` as a parameter is carried into `trade()`, containing two categories of objects, `candle` and `orderBooks`. The following code snippet shows you how to retrieve `candle` and `orderBooks` info,

``` javascript
  const candle = information.candle;
  const orderBooks = information.orderBooks;
```

Crypto-Arsenal supports multi-pair trading, `candle` and `orderBooks` could carry-in data for multiple pairs. The following code snippet shows you how to acquaire one specific candle and orderbook info among multiple pairs in an exchange.
Please note that `exchange` and `pair` are registered by developers through `this.subscribedBooks` in `strategy constructor`.

``` javascript
  const candle = information.candle;
  const oneCandle = candle[exchange][pair];

  const orderBooks = information.orderBooks;
  const oneOrderBook = orderBooks[exchange][pair];
```

If a strategy is used for single-pair trading, the single trading pair and exchange can be assigned on the fly. The following code snippet shows you how to get current `exchange` and `pair` info.

``` javascript
  // Correct way to get exchange / pair in single pair strategy
  const exchange = Object.keys(information.candles)[0];
  const pair = Object.keys(information.candles[exchange])[0];
  const candleData = information.candles[exchange][pair][0];
```

### Candle Info
Each `candle` contains five different info, `open`, `close`, `high`, `low` and `volume` with respect to a trading pair under an exchange. The following code snippet shows you how to acquire these info.

``` javascript
  const candles = information.candles;
  const oneCandle = candles[exchange][pair][0];

  const open = oneCandle.open;
  const close = oneCandle.close;
  const high = oneCandle.high;
  const low = oneCandle.low;
  const volume = oneCandle.volume;
```

### OrderBooks Info
Each `orderBook` contains two different info, `asks` and `bids`, with respect to a trading pair under an exchange. Each of `asks` or `bids` contains three different info, `price`, `count` and `amount`. The following code snippet shows you how to acquire these info. **Please note that there shows NO `orderBooks` info under Backtesting Mode.**

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
The following code snippet shows you how to get your up-to-date assets in an exchange.

```javascript
  const usd = this.assets[exchange]['USDT'];
  const eth = this.assets[exchange]['ETH'];
```

### Return Value from `trade()`
All strategies in Crypto-Arsenal, the return value from `trade()` contains 0 to N objects in an array, shown as below code snippet. You make orders to an exchange by this return value.

### Format of Return Value of `trade()`
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

### Detail for Each Info
* exchange: The exchage you would like to put orders to.
* pair: Your assigned trading pair.
* type: The type of trading, `LIMIT`: order is made by assigned price; `MARKET`: order is made by market price.
* amount: The amount of crypto for trading. '>= 0' means 'buy' (order taker). '< 0' means 'sell' (order maker).
* price: The price you wish for trading. If 'type' is assigned to `MARKET`, then price can be any value.

### LIMIT
In the following code snippet, it shows that this order is made to 'Binance' with assigned price of 212.42 USDT to purpose 1 ETH.

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

### MARKET
In the following code snippet, it shows that this order is made to 'Binance' with current market price (USDT) to purpose 1 ETH.
**Note: It is mandatory that price shalle be assigned to any value, even the type of trading is based on market price!**

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

### Log
Used to log the message. 'str' must be javescript string tyep and maximum length of 'str' is 200 characters.

``` javascript
  Log(str);
```

### Access to Parameter of a Strategy
Use ```this.xxx``` or ```this['OPTION_NAME']``` to access parameter of a strategy.

## Example
The following example is a MA Cross Strategy, which uses [TA-LIB](https://github.com/acrazing/talib-binding-node) to calculate two moving averages, short and long lines. When a signal ('short' breaks through 'long' line) appears, then it shows a 'buying point'; otherwise, 'selling point'.

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

## Parameters You Can Adjust

![](https://i.imgur.com/y82aDps.jpg)

![](https://i.imgur.com/VGIhd4z.png)

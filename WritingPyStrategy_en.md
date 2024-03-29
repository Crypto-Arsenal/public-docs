# Developing My Strategy in Python
## What You Need To Know...
Here are hints for you to quickly develop your trading strategies:
* Get started to develop your Strategy Class in compliance with Python 3.
* There are dedicated interfaces to be coded along, please refer to 'Basic Coding Structure' as below; note that your strategy takes in current market trading data and must return results whether your trades are executed or not.
* The life cycle of your trades/orders are maintained by your own strategy.
* You can leverage off-the-shelf technical analysis indicators of [TA-LIB](https://github.com/acrazing/talib-binding-node) for quickly develop your strategies.
* You can record your trading data with `Log(str)`.

## Create My Cloud Strategy
### Basic Coding Structure
``` python
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

* `information` above, returns the basic trading structure, `__init__` returns all necessary parameters, for more details please refer to [Nodejs](WritingStrategy.md).

### Advanced
* You can access to [numpy](http://www.numpy.org/) by using np.
* You can access to [talib](https://github.com/mrjbq7/ta-lib) by using talib.
* You can access to strategy parameters via ```self['OPTION_NAME']```.

### A Simple Example: MA Cross Strategy
``` python
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
## Upload .pyc to the Cloud
When using Python to develop you trading strategies, you can choose to upload your binary codes (.pyc) to our platform if you want to keep your source codes in private.
Make sure using python3.6 to compile

1. Compile your source codes with ```python3 -m py_compile strategy.py```.
2. Find ```strategy.cpython-36.pyc``` in ```./__pycache__```
3. Go to My Strategy, under `Edit Strategy` tab, check `Use Binary Code` to upload your binary codes (.pyc).




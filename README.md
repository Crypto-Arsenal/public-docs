# Crypto Arsenal Documentation

This repository contains document files for [https://www.crypto-arsenal.io/](https://www.crypto-arsenal.io/)

| File Name  | Description | Language |
| ---------- | ----------- | -------- |
| [WritingStrategy.md](WritingStrategy.md) | Develop My Strategy with JS | 中文 |
| [WritingStrategy_en.md](WritingStrategy_en.md) | Develop My Strategy with JS | English |
| [WritingPyStrategy.md](WritingPyStrategy.md) | Develop My Strategy with Python | 中文 |
| [WritingPyStrategy_en.md](WritingPyStrategy_en.md) | Develop My Strategy with Python | English |
| [RemoteStrategy.md](RemoteStrategy.md) | Develop My Remote Strategy | 中文 |
| [RemoteStrategy_en.md](RemoteStrategy_en.md) | Develop My Remote Strategy | English |


# Strategy Argument Configuration

You can add some extra arguments for the strategy and change these arguments for every live trade, backtest or simulation.
For example, if you have the following configuration for a strategy

## Warning

All arguments are actually stored and parsed as `String` type in your strategy code. **So you have to parse it by your self!**

## Type

- Boolean

  A **STRING** that indicates either `true` or `false`

- Number

  A **STRING** that represents a `float` value.

- String

  A **STRING** that holds a `ASCII` string

- Select

  A **STRING** that holds a list of `ASCII` string. Separated by the symbol vertical bar `|`.
  

## Example

Suppose you use the following configuration

| Variable | Description  | Type    | Default Value             |
| -------- | ------------ | ------- | ------------------------- |
| MyArgA   | true/false   | Boolean | true                      |
| MyArgB   | a float num  | Number  | 10.9                      |
| MyArgC   | a ASCII Str  | String  | Hello World!              |
| MyArgD   | a ASCII list | Select  | OptionA\|OptionB\|OptionC |

**Note: Only ASCII string is supported currently.**

Then you're allowed to tweak these variables to be used in your strategy before trading. `MyArgD` will appear as a list consisting of `OptionA`, `OptionB` and `OptionC`.

![image](https://user-images.githubusercontent.com/5862369/56237500-b3511900-60be-11e9-878d-3e5c2cff4991.png)


### JavaScript Example

You can access these arguments by `this.argName` or `this['argName']`.

```js
Log(this['MyArgA']);        // true
Log(typeof this['MyArgA']); // string

Log(this['MyArgB']);        // 10.9
Log(typeof this['MyArgB']); // string

Log(this['MyArgC']);        // Hello World!
Log(typeof this['MyArgC']); // string

Log(this['MyArgD']);        // OptionC
Log(typeof this['MyArgD']); // string
```

#### All of the types are string in javascript

So if you want to check  MyArgA is true or not, you have to use it like `if(this['MyArgA'] === 'true') {...}` instead of ~~`if(this['MyArgA]) { ... }`~~.

### Python Example

You can access these arguments by `self['argName']`.

```py
Log(self['MyArgA']) # true
Log(str(self['MyArgA'].__class__)) # <class 'str'>

Log(self['MyArgB']) # 10.9
Log(str(self['MyArgB'].__class__)) # <class 'str'>

Log(self['MyArgC']) # Hello World!
Log(str(self['MyArgC'].__class__)) # <class 'str'>

Log(self['MyArgD']) # OptionC
Log(str(self['MyArgD'].__class__)) # <class 'str'>
```

#### All of the types are string in python

So if you want to check  MyArgA is true or not, you have to use it like `if self['MyArgA'] == 'true'` instead of ~~`if self['MyArgA]`~~.


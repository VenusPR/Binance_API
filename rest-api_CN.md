# REST行情与交易接口 (2022-06-15)
## API 基本信息
* 本篇列出接口的baseurl: **https://api.binance.com**
* 如果上面的baseURL访问有性能问题，请访问下面的API集群:
  * **https://api1.binance.com**
  * **https://api2.binance.com**
  * **https://api3.binance.com**
* 所有接口的响应都是 JSON 格式。
* 响应中如有数组，数组元素以时间**升序**排列，越早的数据越提前。  
* 所有时间、时间戳均为UNIX时间，单位为**毫秒**。

## HTTP 返回代码

* HTTP `4XX` 错误码用于指示错误的请求内容、行为、格式。问题在于请求者。
* HTTP `403` 错误码表示违反WAF限制(Web应用程序防火墙)。
* HTTP `429` 错误码表示警告访问频次超限，即将被封IP。
* HTTP `418` 表示收到429后继续访问，于是被封了。
* HTTP `5XX` 错误码用于指示Binance服务侧的问题。    


## 接口错误代码

* 每个接口都有可能抛出异常，异常响应格式如下：

```javascript
{
  "code": -1121,
  "msg": "Invalid symbol."
}
```

* 具体的错误码及其解释在[错误代码汇总](./errors_CN.md)

## 接口的基本信息
* `GET` 方法的接口, 参数必须在 `query string`中发送。
* `POST`, `PUT`, 和 `DELETE` 方法的接口,参数可以在内容形式为`application/x-www-form-urlencoded`的 `query string` 中发送，也可以在 `request body` 中发送。 如果你喜欢，也可以混合这两种方式发送参数。
* 对参数的顺序不做要求。
* 但如果同一个参数名在`query string`和`request body`中都有，`query string`中的会被优先采用。

# 访问限制
## 访问限制基本信息
* 以下是`intervalLetter` 作为头部值:
  * SECOND => S
  * MINUTE => M
  * HOUR => H
  * DAY => D
* 在 `/api/v3/exchangeInfo`接口中`rateLimits`数组里包含有REST接口(不限于本篇的REST接口)的访问限制。包括带权重的访问频次限制、下单速率限制。本篇`枚举定义`章节有限制类型的进一步说明。
* 违反任何一个速率限制时，将返回429。

## IP 访问限制
* 每个请求将包含一个`X-MBX-USED-WEIGHT-(intervalNum)(intervalLetter)`的头，其中包含当前IP所有请求的已使用权重。
* 每一个接口均有一个相应的权重(weight)，有的接口根据参数不同可能拥有不同的权重。越消耗资源的接口权重就会越大。
* 收到429时，您有责任停止发送请求，不得滥用API。
* **收到429后仍然继续违反访问限制，会被封禁IP，并收到418错误码**
* 频繁违反限制，封禁时间会逐渐延长，**从最短2分钟到最长3天**.
* `Retry-After`的头会与带有418或429的响应发送，并且会给出**以秒为单位**的等待时长(如果是429)以防止禁令，或者如果是418，直到禁令结束。
* **访问限制是基于IP的，而不是API Key**

## 下单频率限制
* 每个成功的下单回报将包含一个`X-MBX-ORDER-COUNT-(intervalNum)(intervalLetter)`的头，其中包含当前账户已用的下单限制数量。
* 当下单数超过限制时，会收到带有429但不含`Retry-After`头的响应。请检查 `GET api/v3/exchangeInfo` 的下单频率限制 (rateLimitType = ORDERS) 并等待封禁时间结束。
* 被拒绝或不成功的下单并不保证回报中包含以上头内容。
* **下单频率限制是基于每个账户计数的。**
* 用户可以通过接口 `GET api/v3/rateLimit/order` 来查询当前的下单量.

# 数据来源
* 因为API系统是异步的, 所以返回的数据有延时很正常, 也在预期之中。
* 在每个接口中，都列出了其数据的来源，可以用于理解数据的时效性。

系统一共有3个数据来源，按照更新速度的先后排序。排在前面的数据最新，在后面就有可能存在延迟。
  * **撮合引擎** - 表示数据来源于撮合引擎
  * **缓存** - 表示数据来源于内部或者外部的缓存
  * **数据库** - 表示数据直接来源于数据库

有些接口有不止一个数据源, 比如 `缓存 => 数据库`, 这表示接口会先从第一个数据源检查，如果没有数据，则检查下一个数据源。

# 接口鉴权类型
* 每个接口都有自己的鉴权类型，鉴权类型决定了访问时应当进行何种鉴权。
* 鉴权类型会在本文档中各个接口名称旁声明，如果没有特殊声明即默认为 `NONE`。
* 如果需要 API-keys，应当在HTTP头中以 `X-MBX-APIKEY`字段传递。
* API-keys 与 secret-keys **是大小写敏感的**。
* API-keys可以被配置为只拥有访问一些接口的权限。
 例如, 一个 API-key 仅可用于发送交易指令, 而另一个 API-key 则可访问除交易指令外的所有路径。
* 默认 API-keys 可访问所有鉴权路径.

鉴权类型 | 描述
------------ | ------------
NONE | 不需要鉴权的接口
TRADE | 需要有效的API-KEY和签名
USER_DATA | 需要有效的API-KEY和签名
USER_STREAM | 需要有效的API-KEY
MARKET_DATA | 需要有效的API-KEY

* `TRADE` 和 `USER_DATA` 接口是 签名(SIGNED)接口.

# 需要签名的接口 (TRADE 与 USER_DATA)
* 调用`SIGNED` 接口时，除了接口本身所需的参数外，还需要在`query string` 或 `request body`中传递 `signature`, 即签名参数。
* 签名使用`HMAC SHA256`算法. API-KEY所对应的API-Secret作为 `HMAC SHA256` 的密钥，其他所有参数作为`HMAC SHA256`的操作对象，得到的输出即为签名。
* `签名` **大小写不敏感**.
* `totalParams`定义为与`request body`串联的`query string`。

## 时间同步安全
* 签名接口均需要传递 `timestamp`参数，其值应当是请求发送时刻的unix时间戳(毫秒)。
* 服务器收到请求时会判断请求中的时间戳，如果是5000毫秒之前发出的，则请求会被认为无效。这个时间空窗值可以通过发送可选参数 `recvWindow`来定义。
* 逻辑伪代码：

  ```javascript
  if (timestamp < (serverTime + 1000) && (serverTime - timestamp) <= recvWindow) {
    // process request
  } else {
    // reject request
  }
  ```

**关于交易时效性** 
互联网状况并不100%可靠，不可完全依赖,因此你的程序本地到币安服务器的时延会有抖动.
这是我们设置`recvWindow`的目的所在，如果你从事高频交易，对交易时效性有较高的要求，可以灵活设置`recvWindow`以达到你的要求。
**不推荐使用5秒以上的recvWindow**


## POST /api/v3/order 的示例

以下是在linux bash环境下使用 `echo`,`openssl`和`curl`工具实现的一个调用接口下单的示例
apikey、secret仅供示范

Key | Value
------------ | ------------
apiKey | vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A
secretKey | NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j


参数 | 取值
------------ | ------------
symbol | LTCBTC
side | BUY
type | LIMIT
timeInForce | GTC
quantity | 1
price | 0.1
recvWindow | 5000
timestamp | 1499827319559


### 示例 1: 所有参数通过 query string 发送
* **queryString:** symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559
* **HMAC SHA256 签名:**

    ```
    [linux]$ echo -n "symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71
    ```


* **curl 调用:**

    ```
    (HMAC SHA256)
    [linux]$ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://api.binance.com/api/v3/order?symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71'
    ```

### 示例 2: 所有参数通过 request body 发送
* **requestBody:** symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559
* **HMAC SHA256 签名:**

    ```
    [linux]$ echo -n "symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71
    ```


* **curl 调用:**

    ```
    (HMAC SHA256)
    [linux]$ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://api.binance.com/api/v3/order' -d 'symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC&quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=c8db56825ae71d6d79447849e617115f4a920fa2acdcab2b053c4b2838bd6b71'
    ```

### 示例 3: 混合使用 query string 与 request body
* **queryString:** symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC
* **requestBody:** quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559
* **HMAC SHA256 签名:**

    ```
    [linux]$ echo -n "symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTCquantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559" | openssl dgst -sha256 -hmac "NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j"
    (stdin)= 0fd168b8ddb4876a0358a8d14d0c9f3da0e9b20c5d52b2a00fcf7d1c602f9a77
    ```


* **curl 调用:**

    ```
    (HMAC SHA256)
    [linux]$ curl -H "X-MBX-APIKEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A" -X POST 'https://api.binance.com/api/v3/order?symbol=LTCBTC&side=BUY&type=LIMIT&timeInForce=GTC' -d 'quantity=1&price=0.1&recvWindow=5000&timestamp=1499827319559&signature=0fd168b8ddb4876a0358a8d14d0c9f3da0e9b20c5d52b2a00fcf7d1c602f9a77'
    ```

Note that the signature is different in example 3.
There is no & between "GTC" and "quantity=1".

# 公开API接口
## 术语解释
这里的术语适用于全部文档，建议特别是新手熟读，也便于理解。

* `base asset` 指一个交易对的交易对象，即写在靠前部分的资产名, 比如`BTCUSDT`, `BTC`是`base asset`。
* `quote asset` 指一个交易对的定价资产，即写在靠后部分的资产名, 比如`BTCUSDT`, `USDT`是`quote asset`。


## 枚举定义
**交易对状态 (status):**

* PRE_TRADING 盘前交易
* TRADING 正常交易中
* POST_TRADING 盘后交易
* END_OF_DAY 收盘
* HALT 交易终止(该交易对已下线)
* AUCTION_MATCH 集合竞价
* BREAK 交易暂停

**账户与交易对权限(权限):**

* SPOT 现货
* MARGIN 杠杆
* LEVERAGED 杠杆代币
* TRD_GRP_002 交易组 002
* TRD_GRP_003 交易组 003
* TRD_GRP_004 交易组 004

**订单状态 (status):**

状态 | 描述
-----------| --------------
`NEW` | 订单被交易引擎接受
`PARTIALLY_FILLED`| 部分订单被成交
`FILLED` | 订单完全成交
`CANCELED` | 用户撤销了订单
`PENDING_CANCEL` | 撤销中(目前并未使用)
`REJECTED`       | 订单没有被交易引擎接受，也没被处理
`EXPIRED` | 订单被交易引擎取消, 比如 <br/>LIMIT FOK 订单没有成交<br/>市价单没有完全成交<br/>强平期间被取消的订单<br/>交易所维护期间被取消的订单

**OCO 状态 (状态类型集 listStatusType):**

状态 | 描述
-----------| --------------
`RESPONSE`     | 当ListStatus响应失败的操作时使用。 (订单完成或取消订单)
`EXEC_STARTED` | 当已经下单或者订单有更新时
`ALL_DONE`     | 当订单执行结束或者不在激活状态


**OCO 订单状态 (订单状态集 listOrderStatus):**

状态 | 描述
-----------| --------------
`EXECUTING` | 当已经下单或者订单有更新时
`ALL_DONE`| 当订单执行结束或者不在激活状态
`REJECT` | 当订单状态响应失败(订单完成或取消订单)


**指定订单的类型**

* OCO 选择性委托订单

**订单种类 (orderTypes, type):**

* LIMIT 限价单
* MARKET  市价单
* STOP_LOSS 止损单
* STOP_LOSS_LIMIT 限价止损单
* TAKE_PROFIT 止盈单
* TAKE_PROFIT_LIMIT 限价止盈单
* LIMIT_MAKER 限价做市单

**订单返回类型 (newOrderRespType):**

* ACK
* RESULT
* FULL

**订单方向 (side):**

* BUY - 买入
* SELL - 卖出

**Time in force (timeInForce):**

这里定义了订单多久能够失效

Status | Description
-----------| --------------
`GTC` | 成交为止 <br> 订单会一直有效，直到被成交或者取消。
`IOC` | 无法立即成交的部分就撤销 <br> 订单在失效前会尽量多的成交。
`FOK` | 无法全部立即成交就撤销 <br> 如果无法全部成交，订单会失效。

**K线间隔 (interval):**

m -> 分钟; h -> 小时; d -> 天; w -> 周; M -> 月

* 1m
* 3m
* 5m
* 15m
* 30m
* 1h
* 2h
* 4h
* 6h
* 8h
* 12h
* 1d
* 3d
* 1w
* 1M

**限制种类 (rateLimitType):**

* REQUESTS_WEIGHT - 单位时间请求权重之和上限

    ```json
    {
      "rateLimitType": "REQUEST_WEIGHT",
      "interval": "MINUTE",
      "intervalNum": 1,
      "limit": 1200
    }
    ```

* ORDERS - 单位时间下单(撤单)次数上限

    ```json
    {
      "rateLimitType": "ORDERS",
      "interval": "SECOND",
      "intervalNum": 1,
      "limit": 10
    }

* RAW_REQUESTS - 单位时间请求次数上限

    ```json
    {
      "rateLimitType": "RAW_REQUESTS",
      "interval": "MINUTE",
      "intervalNum": 5,
      "limit": 5000
    }
    ```

**限制间隔 (interval):**

* SECOND
* MINUTE
* DAY

## 通用接口
### 测试服务器连通性 PING
```
GET /api/v3/ping
```
测试能否联通

**权重:**
1

**参数:**
NONE

**数据源:**
缓存

**响应:**
```javascript
{}
```

### 获取服务器时间
```
GET /api/v3/time
```
获取服务器时间

**权重:**
1

**参数:**
NONE

**数据源:**
缓存

**响应:**
```javascript
{
  "serverTime": 1499827319559
}
```

### 交易规范信息
```
GET /api/v3/exchangeInfo
```
获取此时的交易规范信息

**权重:**
10


**参数:**

有三种用法

|用法|举例|
----- | ----|
|不需要交易对|curl -X GET "https://api.binance.com/api/v3/exchangeInfo"|
|单个交易对|curl -X GET "https://api.binance.com/api/v3/exchangeInfo?symbol=BNBBTC"|
|多个交易对| curl -X GET "https://api.binance.com/api/v3/exchangeInfo?symbols=%5B%22BNBBTC%22,%22BTCUSDT%22%5D" or curl -g GET 'https://api.binance.com/api/v3/exchangeInfo?symbols=["BTCUSDT","BNBBTC"]'|

如果传入的交易对不存在, 接口会返回错误。

**数据源:**
缓存

**响应:**
```javascript
{
  "timezone": "UTC",
  "serverTime": 1508631584636,
  "rateLimits": [{
      "rateLimitType": "REQUESTS_WEIGHT",
      "interval": "MINUTE",
      "intervalNum": 1,
      "limit": 1200 //每分钟调用的所有接口权重之和不得超过1200
    },
    {
      "rateLimitType": "ORDERS",
      "interval": "SECOND",
      "intervalNum": 1,
      "limit": 10 //每秒钟所有订单/撤单次数不得超过10
    },
    {
      "rateLimitType": "ORDERS",
      "interval": "DAY",
      "intervalNum": 1,
      "limit": 100000 //每天订单/撤单不得超过10万
    },
    {
      "rateLimitType": "RAW_REQUESTS",
      "interval": "MINUTE",
      "intervalNum": 5,
      "limit": 5000 //每5分钟调用订单次数不得超过5000
    }
  ],
  "exchangeFilters": [],
  "symbols": [{
    "symbol": "ETHBTC",
    "status": "TRADING",
    "baseAsset": "ETH",
    "baseAssetPrecision": 8,
    "quoteAsset": "BTC",
    "quotePrecision": 8,
    "quoteAssetPrecision": 8,
    "orderTypes": ["LIMIT", "MARKET"],
    "icebergAllowed": false,
    "allowTrailingStop": false,
    "cancelReplaceAllowed": false,
    "filters": [{
      "filterType": "PRICE_FILTER",
      "minPrice": "0.00000100",
      "maxPrice": "100000.00000000",
      "tickSize": "0.00000100"
    }, {
      "filterType": "LOT_SIZE",
      "minQty": "0.00100000",
      "maxQty": "100000.00000000",
      "stepSize": "0.00100000"
    }, {
      "filterType": "MIN_NOTIONAL",
      "minNotional": "0.00100000",
      "applyToMarket": true,
      "avgPriceMins": 5
    }]
  }]
}
```

## 行情接口
### 深度信息
```
GET /api/v3/depth
```

**权重:**

限制 | 权重
------------ | ------------
1-100 | 1
101-500 | 5
501-1000 | 10
1001-5000 | 50

**参数:**

名称 | 类型 | 是否必须 | 描述
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | 默认 100; 最大 5000. 可选值:[5, 10, 20, 50, 100, 500, 1000, 5000] <br> 如果 limit > 5000, 最多返回5000条数据.

**注意:** limit=0 返回全部orderbook，但数据量会非常非常非常非常大！

**数据源:**
缓存

**响应:**
```javascript
{
  "lastUpdateId": 1027024,
  "bids": [
    [
      "4.00000000",     // 价位
      "431.00000000",   // 挂单量
      []                // 请忽略.
    ]
  ],
  "asks": [
    [
      "4.00000200",
      "12.00000000",
      []
    ]
  ]
}
```

### 近期成交
```
GET /api/v3/trades
```
获取近期成交

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 500; max 1000.

**数据源:**
缓存

**响应:**
```javascript
[
  {
    "id": 28457,
    "price": "4.00000100",
    "qty": "12.00000000",
    "time": 1499865549590,
    "isBuyerMaker": true,
    "isBestMatch": true
  }
]
```

### 查询历史成交(MARKET_DATA)
```
GET /api/v3/historicalTrades
```

**权重:**
5

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
limit | INT | NO | Default 500; max 1000.
fromId | LONG | NO | 从哪一条成交id开始返回. 缺省返回最近的成交记录


**数据源:**
数据库


**响应:**
```javascript
[
  {
    "id": 28457,
    "price": "4.00000100",
    "qty": "12.00000000",
    "time": 1499865549590,
    "isBuyerMaker": true,
    "isBestMatch": true
  }
]
```

### 近期成交(归集)
```
GET /api/v3/aggTrades
```
与trades的区别是，同一个taker在同一时间同一价格与多个maker的成交会被合并为一条记录

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
fromId | LONG | NO | 从包含fromID的成交开始返回结果
startTime | LONG | NO | 从该时刻之后的成交记录开始返回结果
endTime | LONG | NO | 返回该时刻为止的成交记录
limit | INT | NO | 默认 500; 最大 1000.

* 如果同时发送startTime和endTime，间隔必须小于一小时
* 如果没有发送任何筛选参数(fromId, startTime,endTime)，默认返回最近的成交记录

**数据源:**
数据库

**响应:**
```javascript
[
  {
    "a": 26129,         // 归集成交ID
    "p": "0.01633102",  // 成交价
    "q": "4.70443515",  // 成交量
    "f": 27781,         // 被归集的首个成交ID
    "l": 27781,         // 被归集的末个成交ID
    "T": 1498793709153, // 成交时间
    "m": true,          // 是否为主动卖出单
    "M": true           // 是否为最优撮合单(可忽略，目前总为最优撮合)
  }
]
```

### K线数据
```
GET /api/v3/klines
```
每根K线的开盘时间可视为唯一ID

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
interval | ENUM | YES | 详见枚举定义：K线间隔
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.

* 缺省返回最近的数据

**数据源:**
数据库


**响应:**
```javascript
[
  [
    1499040000000,      // 开盘时间
    "0.01634790",       // 开盘价
    "0.80000000",       // 最高价
    "0.01575800",       // 最低价
    "0.01577100",       // 收盘价(当前K线未结束的即为最新价)
    "148976.11427815",  // 成交量
    1499644799999,      // 收盘时间
    "2434.19055334",    // 成交额
    308,                // 成交笔数
    "1756.87402397",    // 主动买入成交量
    "28.46694368",      // 主动买入成交额
    "17928899.62484339" // 请忽略该参数
  ]
]
```
### 当前平均价格
```
GET /api/v3/avgPrice
```
**权重:**
1
**参数:**
Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |

**数据源:**
缓存


**响应:**
```javascript
{
  "mins": 5,
  "price": "9.35751834"
}
```

### 24hr价格变动情况
```
GET /api/v3/ticker/24hr
```
请注意，不携带symbol参数会返回全部交易对数据，不仅数据庞大，而且权重极高

**权重:**

<table>
<thead>
    <tr>
        <th>参数</th>
        <th>提供Symbol数量</th>
        <th>权重</th>
    </tr>
</thead>
<tbody>
    <tr>
        <td rowspan=2>symbol</td>
        <td>1</td>
        <td>1</td>
    </tr>
    <tr>
        <td>不提供symbol</td>
        <td>40</td>
    </tr>
    <tr>
        <td rowspan=4>symbols</td>
        <td>1-20</td>
        <td>1</td>
    </tr>
    <tr>
        <td>21-100</td>
        <td>20</td>
    </tr>
    <tr>
        <td> >= 101</td>
        <td>40</td>
    </tr>
    <tr>
        <td>不提供symbol</td>
        <td>40</td>
    </tr>
</tbody>
</table>

**参数:**

<table>
<thead>
    <tr>
        <th>名称</th>
        <th>类型</th>
        <th>是否强制要求</th>
        <th>详情</th>
    </tr>
</thead>
<tbody>
    <tr>
        <td>symbol</td>
        <td>STRING</td>
        <td>NO</td>
        <td rowspan=2>参数 `symbol` 和 `symbols` 不可以一起使用 <br> 如果都不提供, 所有symbol的ticker数据都会返回. <br><br>
         symbols参数可接受的格式：
         ["BTCUSDT","BNBUSDT"] <br>
         或 <br>
         %5B%22BTCUSDT%22,%22BNBUSDT%22%5D
        </td>
     </tr>
     <tr>
        <td>symbols</td>
        <td>STRING</td>
        <td>NO</td>
     </tr>
</tbody>
</table>

**数据源:**
缓存

**响应:**
```javascript
{
  "symbol": "BNBBTC",
  "priceChange": "-94.99999800",
  "priceChangePercent": "-95.960",
  "weightedAvgPrice": "0.29628482",
  "prevClosePrice": "0.10002000",
  "lastPrice": "4.00000200",
  "lastQty": "200.00000000",
  "bidPrice": "4.00000000",
  "bidQty": "100.00000000",
  "askPrice": "4.00000200",
  "askQty": "100.00000000",
  "openPrice": "99.00000000",
  "highPrice": "100.00000000",
  "lowPrice": "0.10000000",
  "volume": "8913.30000000",
  "quoteVolume": "15.30000000",
  "openTime": 1499783499040,
  "closeTime": 1499869899040,
  "firstId": 28385,   // 首笔成交id
  "lastId": 28460,    // 末笔成交id
  "count": 76         // 成交笔数
}
```
OR
```javascript
[
  {
    "symbol": "BNBBTC",
    "priceChange": "-94.99999800",
    "priceChangePercent": "-95.960",
    "weightedAvgPrice": "0.29628482",
    "prevClosePrice": "0.10002000",
    "lastPrice": "4.00000200",
    "lastQty": "200.00000000",
    "bidPrice": "4.00000000",
    "bidQty": "100.00000000",
    "askPrice": "4.00000200",
    "askQty": "100.00000000",
    "openPrice": "99.00000000",
    "highPrice": "100.00000000",
    "lowPrice": "0.10000000",
    "volume": "8913.30000000",
    "quoteVolume": "15.30000000",
    "openTime": 1499783499040,
    "closeTime": 1499869899040,
  "firstId": 28385,   // 首笔成交id
  "lastId": 28460,    // 末笔成交id
  "count": 76         // 成交笔数
  }
]
```


### 最新价格接口
```
GET /api/v3/ticker/price
```
返回最近价格

**权重:**

<table>
<thead>
    <tr>
        <th>参数</th>
        <th>Symbols数量</th>
        <th>权重</th>
    </tr>
</thead>
<tbody>
    <tr>
        <td rowspan=2>symbol</td>
        <td>1</td>
        <td>1</td>
    </tr>
    <tr>
        <td>不提供symbol</td>
        <td>2</td>
    </tr>
    <tr>
        <td>symbols</td>
        <td>不限</td>
        <td>2</td>
    </tr>
</tbody>
</table>


**参数:**


<table>
<thead>
    <tr>
      <th>参数名</th>
      <th>类型</th>
      <th>是否强制</th>
      <th>详情</th>
    </tr>
</thead>
<tbody>
    <tr>
        <td>symbol</td>
        <td>STRING</td>
        <td>NO</td>
        <td rowspan=2> 参数 `symbol` 和 `symbols` 不可以一起使用 <br> 如果都不提供, 所有symbol的价格数据都会返回. <br><br>
        symbols参数可接受的格式：
         ["BTCUSDT","BNBUSDT"] <br>
         或 <br>
         %5B%22BTCUSDT%22,%22BNBUSDT%22%5D
        </td>
    </tr>
    <tr>
        <td>symbols</td>
        <td>STRING</td>
        <td>NO</td>
    </tr>
</tbody>
</table>


* 不发送交易对参数，则会返回所有交易对信息

**数据源:**
缓存

**响应:**
```javascript
{
  "symbol": "LTCBTC",
  "price": "4.00000200"
}
```
OR
```javascript
[
  {
    "symbol": "LTCBTC",
    "price": "4.00000200"
  },
  {
    "symbol": "ETHBTC",
    "price": "0.07946600"
  }
]
```

### 最优挂单接口
```
GET /api/v3/ticker/bookTicker
```
返回当前最优的挂单(最高买单，最低卖单)

**权重:**

<table>
<thead>
    <tr>
        <th>参数</th>
        <th>Symbols数量</th>
        <th>权重</th>
    </tr>
</thead>
<tbody>
    <tr>
        <td rowspan=2>symbol</td>
        <td>1</td>
        <td>1</td>
    </tr>
    <tr>
        <td>不提供symbol</td>
        <td>2</td>
    </tr>
    <tr>
        <td>symbols</td>
        <td>不限</td>
        <td>2</td>
    </tr>
</tbody>
</table>


**参数:**


<table>
<thead>
    <tr>
      <th>参数名</th>
      <th>类型</th>
      <th>是否强制</th>
      <th>详情</th>
    </tr>
</thead>
<tbody>
    <tr>
        <td>symbol</td>
        <td>STRING</td>
        <td>NO</td>
        <td rowspan=2> 参数 `symbol` 和 `symbols` 不可以一起使用 <br> 如果都不提供, 所有symbol的bookTicker数据都会返回. <br><br>
        symbols参数可接受的格式：
         ["BTCUSDT","BNBUSDT"] <br>
         或 <br>
         %5B%22BTCUSDT%22,%22BNBUSDT%22%5D
        </td>
    </tr>
    <tr>
        <td>symbols</td>
        <td>STRING</td>
        <td>NO</td>
    </tr>
</tbody>
</table>

* 不发送交易对参数，则会返回所有交易对信息

**数据源:**
缓存


**响应:**
```javascript
{
  "symbol": "LTCBTC",
  "bidPrice": "4.00000000",//最优买单价
  "bidQty": "431.00000000",//挂单量
  "askPrice": "4.00000200",//最优卖单价
  "askQty": "9.00000000"//挂单量
}
```
OR
```javascript
[
  {
    "symbol": "LTCBTC",
    "bidPrice": "4.00000000",
    "bidQty": "431.00000000",
    "askPrice": "4.00000200",
    "askQty": "9.00000000"
  },
  {
    "symbol": "ETHBTC",
    "bidPrice": "0.07946700",
    "bidQty": "9.00000000",
    "askPrice": "100000.00000000",
    "askQty": "1000.00000000"
  }
]
```


## 滚动窗口价格变动统计


```
GET /api/v3/ticker
```

**注意:** 此接口和 `GET /api/v3/ticker/24hr` 有所不同.

此接口统计的时间范围比请求的`windowSize`多不超过59999ms.

接口的 `openTime` 是某一分钟的起始，而结束是当前的时间. 所以实际的统计区间会比请求的时间窗口多不超过59999ms.

比如, 结束时间 `closeTime` 是 1641287867099 (January 04, 2022 09:17:47:099 UTC) , `windowSize` 为 `1d`. 那么开始时间 `openTime` 则为 1641201420000 (January 3, 2022, 09:17:00 UTC)

**权重(IP):** 2/交易对. <br><br> 如果`symbols`请求的交易对超过50, 上限是100.

**参数**
<table>
  <tr>
    <th>Name</th>
    <th>Type</th>
    <th>Mandatory</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>symbol</td>
    <td rowspan=2>STRING</td>
    <td rowspan=2>YES</td>
    <td rowspan=2> 提供 symbol或者symbols 其中之一  <br> <tt>symbols</tt> 可以传入的格式: <br> ["BTCUSDT","BNBUSDT"] <br>or <br>%5B%22BTCUSDT%22,%22BNBUSDT%22%5D <br><br> <tt>symbols</tt> 允许最多100个交易对
    </td>
  </tr>
  <tr>
     <td>symbols</td>
  </tr>
  <tr>
     <td>windowSize</td>
     <td>ENUM</td>
     <td>NO</td>
     <td>默认为 <tt>1d</tt> <br> <tt>windowSize</tt> 支持的值: <br> 如果是分钟: <tt>1m</tt>,<tt>2m</tt>....<tt>59m</tt> <br> 如果是小时: <tt>1h</tt>, <tt>2h</tt>....<tt>23h</tt> <br> 如果是天: <tt>1d</tt>...<tt>7d</tt> <br><br>  不可以组合使用, 比如<tt>1d2h</tt></td>
  </tr>
  <tr>
  </tr>
</table>


**数据源:** 数据库

**响应**

使用参数 `symbol` 返回:

```javascript
{
  "symbol":             "BNBBTC",
  "priceChange":        "-8.00000000",  // 价格变化
  "priceChangePercent": "-88.889",      // 价格变化百分比
  "weightedAvgPrice":   "2.60427807",  
  "openPrice":          "9.00000000",
  "highPrice":          "9.00000000",
  "lowPrice":           "1.00000000",
  "lastPrice":          "1.00000000",
  "volume":             "187.00000000",
  "quoteVolume":        "487.00000000",
  "openTime":           1641859200000,  // ticker的开始时间
  "closeTime":          1642031999999,  // ticker的结束时间
  "firstId":            0,              // 统计时间内的第一笔trade id
  "lastId":             60,
  "count":              61              // 统计时间内交易笔数
}
```

使用参数 `symbols` 返回:

```javascript
[
  {
    "symbol": "BTCUSDT",
    "priceChange": "-154.13000000",
    "priceChangePercent": "-0.740",
    "weightedAvgPrice": "20677.46305250",
    "openPrice": "20825.27000000",
    "highPrice": "20972.46000000",
    "lowPrice": "20327.92000000",
    "lastPrice": "20671.14000000",
    "volume": "72.65112300",
    "quoteVolume": "1502240.91155513",
    "openTime": 1655432400000,
    "closeTime": 1655446835460,
    "firstId": 11147809,
    "lastId": 11149775,
    "count": 1967
  },
  {
    "symbol": "BNBBTC",
    "priceChange": "0.00008530",
    "priceChangePercent": "0.823",
    "weightedAvgPrice": "0.01043129",
    "openPrice": "0.01036170",
    "highPrice": "0.01049850",
    "lowPrice": "0.01033870",
    "lastPrice": "0.01044700",
    "volume": "166.67000000",
    "quoteVolume": "1.73858301",
    "openTime": 1655432400000,
    "closeTime": 1655446835460,
    "firstId": 2351674,
    "lastId": 2352034,
    "count": 361
  }
]
```

## 账户接口
### 下单  (TRADE)
```
POST /api/v3/order  (HMAC SHA256)
```

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
side | ENUM | YES | 详见枚举定义：订单方向
type | ENUM | YES | 详见枚举定义：订单种类
timeInForce | ENUM | NO | 详见枚举定义：Time in force
quantity | DECIMAL | NO |
quoteOrderQty | DECIMAL | NO |
price | DECIMAL | NO |
newClientOrderId | STRING | NO | 用户自定义的orderid，如空缺系统会自动赋值
stopPrice | DECIMAL | NO | 仅 `STOP_LOSS`, `STOP_LOSS_LIMIT`, `TAKE_PROFIT`, `TAKE_PROFIT_LIMIT` 需要此参数
trailingDelta|LONG|NO| 用于 `STOP_LOSS`, `STOP_LOSS_LIMIT`, `TAKE_PROFIT`, 和 `TAKE_PROFIT_LIMIT` 类型的订单.
icebergQty | DECIMAL | NO | 仅有限价单(包括条件限价单与限价做事单)可以使用该参数，含义为创建冰山订单并指定冰山订单的尺寸
newOrderRespType | ENUM | NO | 指定响应类型 `ACK`, `RESULT`, or `FULL`; `MARKET` 与 `LIMIT` 订单默认为`FULL`, 其他默认为`ACK`. 
recvWindow | LONG | NO |
timestamp | LONG | YES |

根据 order `type`的不同，某些参数强制要求，具体如下:

Type | 强制要求的参数 | 其他信息
------------ | ------------ | ------------
`LIMIT` | `timeInForce`, `quantity`, `price`
`MARKET` | `quantity` |  市价买卖单可用`quantity`参数来设置`base asset`数量.<br> 例如：BTCUSDT 市价单，BTC 买卖数量取决于`quantity`参数. <br><br>市价买卖单可用`quoteOrderQty`参数来设置`quote asset`数量. 正确的`quantity`取决于市场的流动性与`quoteOrderQty`<br> 例如: 市价 `BUY` BTCUSDT，单子会基于`quoteOrderQty`- USDT 的数量，购买 BTC.<br> 市价 `SELL` BTCUSDT，单子会卖出 BTC 来满足`quoteOrderQty`- USDT 的数量.
`STOP_LOSS` | `quantity`, `stopPrice`, `trailingDelta` | 条件满足后会下`MARKET`单子. (例如：达到`stopPrice`或`trailingDelta`被启动)
`STOP_LOSS_LIMIT` | `timeInForce`, `quantity`,  `price`, `stopPrice`, `trailingDelta`
`TAKE_PROFIT` | `quantity`, `stopPrice`, `trailingDelta` | 条件满足后会下`MARKET`单子. (例如：达到`stopPrice`或`trailingDelta`被启动)
`TAKE_PROFIT_LIMIT` | `timeInForce`, `quantity`, `price`, `stopPrice`, `trailingDelta`
`LIMIT_MAKER` | `quantity`, `price` | 订单大部分情况下与普通的限价单没有区别，但是如果在当前价格会立即吃对手单并成交则下单会被拒绝。因此使用这个订单类型可以保证订单一定是挂单方，不会成为吃单方。

其他:

* 任何`LIMIT`或`LIMIT_MAKER`只要填`icebergQty`参数都可以下冰上订单。 
* 冰山订单的 `timeInForce`必须设置为`GTC`。
* `STOP_LOSS`, `STOP_LOSS_LIMIT`, `TAKE_PROFIT_LIMIT` 与 `TAKE_PROFIT` 单子都能同时填上`trailingDelta`与`stopPrice`。
* 填上`quoteOrderQty`的市价单不会触犯过滤器的`LOT_SIZE`限制。订单的`quantity`会尽量满足`quoteOrderQty`的数量。

条件单的触发价格必须:

* 比下单时当前市价高: `STOP_LOSS` `BUY`, `TAKE_PROFIT` `SELL`
* 比下单时当前市价低: `STOP_LOSS` `SELL`, `TAKE_PROFIT` `BUY`

关于 newOrderRespType的三种选择

**数据源:**
撮合引擎

**Response ACK:**
返回速度最快，不包含成交信息，信息量最少
```javascript
{
  "symbol": "BTCUSDT",
  "orderId": 28,
  "clientOrderId": "6gCrw2kRUAF9CvJDGP16IP",
  "transactTime": 1507725176595
}
```

**Response RESULT:**
返回速度居中，返回吃单成交的少量信息
```javascript
{
  "symbol": "BTCUSDT",
  "orderId": 28,
  "clientOrderId": "6gCrw2kRUAF9CvJDGP16IP",
  "transactTime": 1507725176595,
  "price": "1.00000000",
  "origQty": "10.00000000",
  "executedQty": "10.00000000",
  "cummulativeQuoteQty": "10.00000000",
  "status": "FILLED",
  "timeInForce": "GTC",
  "type": "MARKET",
  "side": "SELL"
}
```

**Response FULL:**
返回速度最慢，返回吃单成交的详细信息
```javascript
{
  "symbol": "BTCUSDT",
  "orderId": 28,
  "clientOrderId": "6gCrw2kRUAF9CvJDGP16IP",
  "transactTime": 1507725176595,
  "price": "1.00000000",
  "origQty": "10.00000000",
  "executedQty": "10.00000000",
  "cummulativeQuoteQty": "10.00000000",
  "status": "FILLED",
  "timeInForce": "GTC",
  "type": "MARKET",
  "side": "SELL",
  "fills": [
    {
      "price": "4000.00000000",
      "qty": "1.00000000",
      "commission": "4.00000000",
      "commissionAsset": "USDT",
      "tradeId": 56
    },
    {
      "price": "3999.00000000",
      "qty": "5.00000000",
      "commission": "19.99500000",
      "commissionAsset": "USDT",
      "tradeId": 57
    },
    {
      "price": "3998.00000000",
      "qty": "2.00000000",
      "commission": "7.99600000",
      "commissionAsset": "USDT",
      "tradeId": 58
    },
    {
      "price": "3997.00000000",
      "qty": "1.00000000",
      "commission": "3.99700000",
      "commissionAsset": "USDT",
      "tradeId": 59
    },
    {
      "price": "3995.00000000",
      "qty": "1.00000000",
      "commission": "3.99500000",
      "commissionAsset": "USDT",
      "tradeId": 60
    }
  ]
}
```

### 测试下单接口 (TRADE)
```
POST /api/v3/order/test (HMAC SHA256)
```
用于测试订单请求，但不会提交到撮合引擎

**权重:**
1

**参数:**

参考 `POST /api/v3/order`

**数据源:**
缓存

**响应:**
```javascript
{}
```


### 撤消挂单再下单 (TRADE)

```
POST /api/v3/order/cancelReplace
```
撤消挂单并在同个交易对上重新下单。

在撤消订单和下单前会判断: 1) 过滤器参数, 以及 2) 目前下单数量。

即使请求中没有尝试发送新订单，比如(`newOrderResult: NOT_ATTEMPTED`)，下单的数量仍然会加1。

**Weight(IP):** 1

**Parameters:**
Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
side   |ENUM| YES|
type   |ENUM| YES|
cancelReplaceMode|ENUM|YES| 指定类型：`STOP_ON_FAILURE` - 如果撤消订单失败将不会继续重新下单。<br> `ALLOW_FAILURE` - 不管撤消订单是否成功都会继续重新下单。
timeInForce|ENUM|NO|
quantity|DECIMAL|NO|
quoteOrderQty |DECIMAL|NO
price |DECIMAL|NO
cancelNewClientOrderId|STRING|NO | 用户自定义的id，如空缺系统会自动赋值
cancelOrigClientOrderId|STRING| NO| 必须提供`cancelOrigClientOrderId` 或者 `cancelOrderId`。 如果两个参数都提供, `cancelOrderId` 会占优先。
cancelOrderId|LONG|NO| 必须提供`cancelOrigClientOrderId` 或者 `cancelOrderId`。 如果两个参数都提供, `cancelOrderId` 会占优先。
newClientOrderId |STRING|NO| 用于辨识新订单。
stopPrice|DECIMAL|NO|
trailingDelta|LONG|NO|
icebergQty|DECIMAL|NO|
newOrderRespType|ENUM|NO|指定响应类型: <br> 指定响应类型 `ACK`, `RESULT`, or `FULL`; `MARKET` 与 `LIMIT` 订单默认为`FULL`, 其他默认为`ACK`.
recvWindow | LONG | NO | 不能大于 `60000`
timestamp | LONG | YES |


如同 `POST /api/v3/order` , 额外的强制参数取决于 `type` 。

响应格式根据消息的处理是成功、部分成功还是失败而有所不同。

**数据来源:**
撮合引擎


**Response SUCCESS:**
```javascript
// 撤单和下单都成功
{
  "cancelResult": "SUCCESS",
  "newOrderResult": "SUCCESS",
  "cancelResponse": {
    "symbol": "BTCUSDT",
    "origClientOrderId": "DnLo3vTAQcjha43lAZhZ0y",
    "orderId": 9,
    "orderListId": -1,
    "clientOrderId": "osxN3JXAtJvKvCqGeMWMVR",
    "price": "0.01000000",
    "origQty": "0.000100",
    "executedQty": "0.00000000",
    "cummulativeQuoteQty": "0.00000000",
    "status": "CANCELED",
    "timeInForce": "GTC",
    "type": "LIMIT",
    "side": "SELL"
  },
  "newOrderResponse": {
    "symbol": "BTCUSDT",
    "orderId": 10,
    "orderListId": -1,
    "clientOrderId": "wOceeeOzNORyLiQfw7jd8S",
    "transactTime": 1652928801803,
    "price": "0.02000000",
    "origQty": "0.040000",
    "executedQty": "0.00000000",
    "cummulativeQuoteQty": "0.00000000",
    "status": "NEW",
    "timeInForce": "GTC",
    "type": "LIMIT",
    "side": "BUY",
    "fills": []
  }
}
```

**选择了STOP_ON_FAILURE, 撤单出现错误**
```javascript
{
  "code": -2022,
  "msg": "Order cancel-replace failed.",
  "data": {
    "cancelResult": "FAILURE",
    "newOrderResult": "NOT_ATTEMPTED",
    "cancelResponse": {
      "code": -2011,
      "msg": "Unknown order sent."
    },
    "newOrderResponse": null
  }
}
```

**响应：撤单成功，下单失败**
```javascript
{
  "code": -2021,
  "msg": "Order cancel-replace partially failed.",
  "data": {
    "cancelResult": "SUCCESS",
    "newOrderResult": "FAILURE",
    "cancelResponse": {
      "symbol": "BTCUSDT",
      "origClientOrderId": "86M8erehfExV8z2RC8Zo8k",
      "orderId": 3,
      "orderListId": -1,
      "clientOrderId": "G1kLo6aDv2KGNTFcjfTSFq",
      "price": "0.006123",
      "origQty": "10000.000000",
      "executedQty": "0.000000",
      "cummulativeQuoteQty": "0.000000",
      "status": "CANCELED",
      "timeInForce": "GTC",
      "type": "LIMIT_MAKER",
      "side": "SELL"
    },
    "newOrderResponse": {
      "code": -2010,
      "msg": "Order would immediately match and take."
    }
  }
}
```

**选择ALLOW_FAILURE, 撤单出现错误**
```javascript
{
  "code": -2021,
  "msg": "Order cancel-replace partially failed.",
  "data": {
    "cancelResult": "FAILURE",
    "newOrderResult": "SUCCESS",
    "cancelResponse": {
      "code": -2011,
      "msg": "Unknown order sent."
    },
    "newOrderResponse": {
      "symbol": "BTCUSDT",
      "orderId": 11,
      "orderListId": -1,
      "clientOrderId": "pfojJMg6IMNDKuJqDxvoxN",
      "transactTime": 1648540168818
    }
  }
}
```
>**响应：撤单和下单失败**
```javascript
{
  "code": -2022,
  "msg": "Order cancel-replace failed.",
  "data": {
    "cancelResult": "FAILURE",
    "newOrderResult": "FAILURE",
    "cancelResponse": {
      "code": -2011,
      "msg": "Unknown order sent."
    },
    "newOrderResponse": {
      "code": -2010,
      "msg": "Order would immediately match and take."
    }
  }
}
```


### 查询订单 (USER_DATA)
```
GET /api/v3/order (HMAC SHA256)
```
查询订单状态

**权重:**
2

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
recvWindow | LONG | NO |
timestamp | LONG | YES |

注意:
* 至少需要发送 `orderId` 与 `origClientOrderId`中的一个
* 某些订单中`cummulativeQuoteQty`<0，是由于这些订单是cummulativeQuoteQty功能上线之前的订单。

**数据源:**
数据库

**响应:**
```javascript
{
  "symbol": "LTCBTC",
  "orderId": 1,
  "clientOrderId": "myOrder1",
  "price": "0.1",
  "origQty": "1.0",
  "executedQty": "0.0",
  "cummulativeQuoteQty": "0.0",
  "status": "NEW",
  "timeInForce": "GTC",
  "type": "LIMIT",
  "side": "BUY",
  "stopPrice": "0.0",
  "icebergQty": "0.0",
  "time": 1499827319559,
  "updateTime": 1499827319559,
  "isWorking": true
}
```

### 撤销订单 (TRADE)
```
DELETE /api/v3/order  (HMAC SHA256)
```

**权重:**
1

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO |
origClientOrderId | STRING | NO |
newClientOrderId | STRING | NO |  用户自定义的本次撤销操作的ID(注意不是被撤销的订单的自定义ID)。如无指定会自动赋值。
recvWindow | LONG | NO |
timestamp | LONG | YES |

* `orderId` 与 `origClientOrderId` 必须至少发送一个.
* 如果两个参数一起发送, `orderId`优先被考虑.

**数据源:**
撮合引擎

**响应:**
```javascript
{
  "symbol": "LTCBTC",
  "orderId": 28,
  "origClientOrderId": "myOrder1",
  "clientOrderId": "cancelMyOrder1",
  "transactTime": 1507725176595,
  "price": "1.00000000",
  "origQty": "10.00000000",
  "executedQty": "8.00000000",
  "cummulativeQuoteQty": "8.00000000",
  "status": "CANCELED",
  "timeInForce": "GTC",
  "type": "LIMIT",
  "side": "SELL"
}
```

``
DELETE /api/v3/openOrders
``

撤销单一交易对下所有挂单, 包括OCO的挂单。

**权重(IP):**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
recvWindow | LONG | NO | 不能大于 ```60000```
timestamp | LONG | YES |

**数据源:**
撮合引擎


## 查询订单 (USER_DATA)


**响应**
```javascript
{
  "symbol": "LTCBTC", // 交易对
  "orderId": 1, // 系统的订单ID
  "orderListId": -1, // OCO订单的ID，不然就是-1
  "clientOrderId": "myOrder1", // 客户自己设置的ID
  "price": "0.1", // 订单价格
  "origQty": "1.0", // 用户设置的原始订单数量
  "executedQty": "0.0", // 交易的订单数量
  "cummulativeQuoteQty": "0.0", // 累计交易的金额
  "status": "NEW", // 订单状态
  "timeInForce": "GTC", // 订单的时效方式
  "type": "LIMIT", // 订单类型， 比如市价单，现价单等
  "side": "BUY", // 订单方向，买还是卖
  "stopPrice": "0.0", // 止损价格
  "icebergQty": "0.0", // 冰山数量
  "time": 1499827319559, // 订单时间
  "updateTime": 1499827319559, // 最后更新时间
  "isWorking": true, // 订单是否出现在orderbook中
  "origQuoteOrderQty": "0.000000" // 原始的交易金额
}
```


### 查看账户当前挂单 (USER_DATA)
```
GET /api/v3/openOrders  (HMAC SHA256)
```
请小心使用不带symbol参数的调用

**权重:**
带symbol 3
不带40

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | NO |
recvWindow | LONG | NO |
timestamp | LONG | YES |

* 不带symbol参数，会返回所有交易对的挂单

**数据源:**
数据库

**响应:**
```javascript
[
  {
    "symbol": "LTCBTC",
    "orderId": 1,
    "clientOrderId": "myOrder1",
    "price": "0.1",
    "origQty": "1.0",
    "executedQty": "0.0",
    "cummulativeQuoteQty": "0.0",
    "status": "NEW",
    "timeInForce": "GTC",
    "type": "LIMIT",
    "side": "BUY",
    "stopPrice": "0.0",
    "icebergQty": "0.0",
    "time": 1499827319559,
    "updateTime": 1499827319559,
    "isWorking": true
  }
]
```

### 查询所有订单（包括历史订单） (USER_DATA)
```
GET /api/v3/allOrders (HMAC SHA256)
```

**权重:**
10 

**Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId | LONG | NO | 只返回此orderID之后的订单，缺省返回最近的订单
startTime | LONG | NO |
endTime | LONG | NO |
limit | INT | NO | Default 500; max 1000.
recvWindow | LONG | NO |
timestamp | LONG | YES |

**注意:**
* 如设置 `orderId` , 订单量将 >=  `orderId`。否则将返回最新订单。
* 一些历史订单 `cummulativeQuoteQty`  < 0, 是指数据此时不存在。
* 如果设置 `startTime` 和 `endTime`, `orderId` 就不需要设置。

**数据源:**
数据库

**响应:**
```javascript
[
  {
    "symbol": "LTCBTC",
    "orderId": 1,
    "clientOrderId": "myOrder1",
    "price": "0.1",
    "origQty": "1.0",
    "executedQty": "0.0",
    "cummulativeQuoteQty": "0.0",
    "status": "NEW",
    "timeInForce": "GTC",
    "type": "LIMIT",
    "side": "BUY",
    "stopPrice": "0.0",
    "icebergQty": "0.0",
    "time": 1499827319559,
    "updateTime": 1499827319559,
    "isWorking": true
  }
]
```
``
POST /api/v3/order/oco (HMAC SHA256)
``

## 发送新 OCO 订单

**权重(UID)**: 2
**权重(IP)**: 1

**参数**:

名称 |类型| 是否必需 | 描述
-----|-----|----------| -----------
symbol|STRING| YES|
listClientOrderId|STRING|NO| 整个orderList的唯一ID
side|ENUM|YES| 详见枚举定义：订单方向
quantity|DECIMAL|YES|
limitClientOrderId|STRING|NO| 限价单的唯一ID
price|DECIMAL|YES|
limitIcebergQty|DECIMAL|NO|
trailingDelta|LONG|NO|
stopClientOrderId |STRING|NO| 止损/止损限价单的唯一ID
stopPrice |DECIMAL| YES
stopLimitPrice|DECIMAL|NO| 如果提供，须配合提交`stopLimitTimeInForce`
stopIcebergQty|DECIMAL|NO|
stopLimitTimeInForce|ENUM|NO| 有效值 `GTC`/`FOK`/`IOC`
newOrderRespType|ENUM|NO| 详见枚举定义：订单返回类型
recvWindow|LONG|NO| 不能大于 `60000`
timestamp|LONG|YES|


其他信息:

* 价格限制:
  * `SELL`: 限价 > 最新成交价 >触发价
  * `BUY`: 限价 < 最新成交价 < 触发价
* 数量限制:
  * 两个 legs 必须具有同样的数量。
  * `ICEBERG`数量不必相同
* 下单rate
  * 一个`OCO`订单被算成2个普通订单.

**数据源:**
撮合引擎


## 取消 OCO 订单(TRADE)

``
DELETE /api/v3/orderList (HMAC SHA256)
``

取消整个订单列表。

**权重(IP)**: 1

**参数**

名称| 类型| 是否必需| 描述
----| ----|------|------
symbol| STRING| YES|
orderListId|LONG|NO| `orderListId` 或 `listClientOrderId` 必须被提供
listClientOrderId|STRING|NO| `orderListId` 或 `listClientOrderId` 必须被提供
newClientOrderId|STRING|NO| 用户自定义的本次撤销操作的ID(注意不是被撤销的订单的自定义ID)。如无指定会自动赋值。
recvWindow|LONG|NO|不能大于 `60000`
timestamp|LONG|YES|

其他注意点:

* 取消单个 leg 将取消整个 OCO 订单.
* 如果 `orderListId` 和 `listClientOrderId` 一起发送, `orderListId` 优先被考虑.


**数据源:**
撮合引擎

**响应**

```javascript
{
  "orderListId": 0,
  "contingencyType": "OCO",
  "listStatusType": "ALL_DONE",
  "listOrderStatus": "ALL_DONE",
  "listClientOrderId": "C3wyj4WVEktd7u9aVBRXcN",
  "transactionTime": 1574040868128,
  "symbol": "LTCBTC",
  "orders": [
    {
      "symbol": "LTCBTC",
      "orderId": 2,
      "clientOrderId": "pO9ufTiFGg3nw2fOdgeOXa"
    },
    {
      "symbol": "LTCBTC",
      "orderId": 3,
      "clientOrderId": "TXOvglzXuaubXAaENpaRCB"
    }
  ],
  "orderReports": [
    {
      "symbol": "LTCBTC",
      "origClientOrderId": "pO9ufTiFGg3nw2fOdgeOXa",
      "orderId": 2,
      "orderListId": 0,
      "clientOrderId": "unfWT8ig8i0uj6lPuYLez6",
      "price": "1.00000000",
      "origQty": "10.00000000",
      "executedQty": "0.00000000",
      "cummulativeQuoteQty": "0.00000000",
      "status": "CANCELED",
      "timeInForce": "GTC",
      "type": "STOP_LOSS_LIMIT",
      "side": "SELL",
      "stopPrice": "1.00000000"
    },
    {
      "symbol": "LTCBTC",
      "origClientOrderId": "TXOvglzXuaubXAaENpaRCB",
      "orderId": 3,
      "orderListId": 0,
      "clientOrderId": "unfWT8ig8i0uj6lPuYLez6",
      "price": "3.00000000",
      "origQty": "10.00000000",
      "executedQty": "0.00000000",
      "cummulativeQuoteQty": "0.00000000",
      "status": "CANCELED",
      "timeInForce": "GTC",
      "type": "LIMIT_MAKER",
      "side": "SELL"
    }
  ]
}
```

## 查询 OCO (USER_DATA)

``
GET /api/v3/orderList (HMAC SHA256)
``

根据提供的可选参数检索特定的OCO。

**权重(IP)**: 2

**参数**:

名称| 类型|是否必需| 描述
----|-----|----|----------
orderListId|LONG|NO|   ```orderListId``` 或 ```origClientOrderId``` 必须提供一个。
origClientOrderId|STRING|NO|  ```orderListId``` 或 ```origClientOrderId``` 必须提供一个。
recvWindow|LONG|NO| 赋值不得大于 ```60000```
timestamp|LONG|YES|

**数据源:**
数据库

**响应**

```javascript
{
    "orderListId": 27,
    "contingencyType": "OCO",
    "listStatusType": "EXEC_STARTED",
    "listOrderStatus": "EXECUTING",
    "listClientOrderId": "h2USkA5YQpaXHPIrkd96xE",
    "transactionTime": 1565245656253,
    "symbol": "LTCBTC",
    "orders": [
        {
            "symbol": "LTCBTC",
            "orderId": 4,
            "clientOrderId": "qD1gy3kc3Gx0rihm9Y3xwS"
        },
        {
            "symbol": "LTCBTC",
            "orderId": 5,
            "clientOrderId": "ARzZ9I00CPM8i3NhmU9Ega"
        }
    ]
}
```

## 查询所有 OCO (USER_DATA)

``
GET /api/v3/allOrderList (HMAC SHA256)
``

根据提供的可选参数检索所有的OCO。

**权重(IP)**: 10

**参数**

名称|类型| 是否必需| 描述
----|----|----|---------
fromId|LONG|NO| 提供该项后, `startTime` 和 `endTime` 都不可提供
startTime|LONG|NO|
endTime|LONG|NO|
limit|INT|NO| 默认值: 500; 最大值: 1000
recvWindow|LONG|NO| 赋值不能超过 `60000`
timestamp|LONG|YES|

**数据源:**
数据库

**响应**

```javascript
[
  {
    "orderListId": 29,
    "contingencyType": "OCO",
    "listStatusType": "EXEC_STARTED",
    "listOrderStatus": "EXECUTING",
    "listClientOrderId": "amEEAXryFzFwYF1FeRpUoZ",
    "transactionTime": 1565245913483,
    "symbol": "LTCBTC",
    "orders": [
      {
        "symbol": "LTCBTC",
        "orderId": 4,
        "clientOrderId": "oD7aesZqjEGlZrbtRpy5zB"
      },
      {
        "symbol": "LTCBTC",
        "orderId": 5,
        "clientOrderId": "Jr1h6xirOxgeJOUuYQS7V3"
      }
    ]
  },
  {
    "orderListId": 28,
    "contingencyType": "OCO",
    "listStatusType": "EXEC_STARTED",
    "listOrderStatus": "EXECUTING",
    "listClientOrderId": "hG7hFNxJV6cZy3Ze4AUT4d",
    "transactionTime": 1565245913407,
    "symbol": "LTCBTC",
    "orders": [
      {
        "symbol": "LTCBTC",
        "orderId": 2,
        "clientOrderId": "j6lFOfbmFMRjTYA7rRJ0LP"
      },
      {
        "symbol": "LTCBTC",
        "orderId": 3,
        "clientOrderId": "z0KCjOdditiLS5ekAFtK81"
      }
    ]
  }
]
```

## 查询 OCO 挂单 (USER_DATA)

``
GET /api/v3/openOrderList (HMAC SHA256)
``

**权重(IP)**: 3

**参数**

名称| 类型|是否必需| 描述
----|-----|---|------------------
recvWindow|LONG|NO| 赋值不能大于 ```60000```
timestamp|LONG|YES|


**数据源:**
数据库

**响应**

```javascript
[
  {
    "orderListId": 31,
    "contingencyType": "OCO",
    "listStatusType": "EXEC_STARTED",
    "listOrderStatus": "EXECUTING",
    "listClientOrderId": "wuB13fmulKj3YjdqWEcsnp",
    "transactionTime": 1565246080644,
    "symbol": "LTCBTC",
    "orders": [
      {
        "symbol": "LTCBTC",
        "orderId": 4,
        "clientOrderId": "r3EH2N76dHfLoSZWIUw1bT"
      },
      {
        "symbol": "LTCBTC",
        "orderId": 5,
        "clientOrderId": "Cv1SnyPD3qhqpbjpYEHbd2"
      }
    ]
  }
]
```

### 账户信息 (USER_DATA)
```
GET /api/v3/account (HMAC SHA256)
```

**权重:**
10

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO |
timestamp | LONG | YES |

**数据源:**
缓存 => 数据库

**响应:**
```javascript
{
  "makerCommission": 15,
  "takerCommission": 15,
  "buyerCommission": 0,
  "sellerCommission": 0,
  "canTrade": true,
  "canWithdraw": true,
  "canDeposit": true,
  "updateTime": 123456789,
  "balances": [
    {
      "asset": "BTC",
      "free": "4723846.89208129",
      "locked": "0.00000000"
    },
    {
      "asset": "LTC",
      "free": "4763368.68006011",
      "locked": "0.00000000"
    }
  ]
}
```

### 账户成交历史 (USER_DATA)
```
GET /api/v3/myTrades  (HMAC SHA256)
```
获取某交易对的成交历史

**权重:**
10

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES |
orderId|LONG|NO| 必须要和参数`symbol`一起使用.
startTime | LONG | NO |
endTime | LONG | NO |
fromId | LONG | NO |返回该fromId之后的成交，缺省返回最近的成交
limit | INT | NO | Default 500; max 1000.
recvWindow | LONG | NO |
timestamp | LONG | YES |

**数据源:**
数据库

**响应:**
```javascript
[
  {
    "symbol": "BNBBTC",
    "id": 28457,
    "orderId": 100234,
    "price": "4.00000100",
    "qty": "12.00000000",
    "commission": "10.10000000",
    "commissionAsset": "BNB",
    "time": 1499865549590,
    "isBuyer": true,
    "isMaker": false,
    "isBestMatch": true
  }
]
```
### 查询目前下单数 (TRADE)
```
GET /api/v3/rateLimit/order
```
获取用户在当前时间区间内的下单总数。

**权重(IP):**
20

**参数:**
名称 | 类型| 是否必需 | 描述
------------ | ------------ | ------------ | ------------
recvWindow | LONG | NO | 赋值不得大于 ```60000```
timestamp | LONG | YES |

**数据源:**
缓存

**响应**
```javascript
[
  {
    "rateLimitType": "ORDERS",
    "interval": "SECOND",
    "intervalNum": 10,
    "limit": 10000,
    "count": 0
  },
  {
    "rateLimitType": "ORDERS",
    "interval": "DAY",
    "intervalNum": 1,
    "limit": 20000,
    "count": 0
  }
]
```

## 用户数据流订阅接口
此处仅列出如何得到数据流名称及如何维持有效期的接口，具体订阅方式参考另一篇websocket接口文档

### 新建用户数据流 (USER_STREAM)
```
POST /api/v3/userDataStream
```
从创建起60分钟有效

**权重:**
1

**Parameters:**
NONE

**数据源:**
缓存


**响应:**
```javascript
{
  "listenKey": "pqia91ma19a5s61cv6a81va65sdf19v8a65a1a5s61cv6a81va65sdf19v8a65a1" //用于订阅的数据流名
}
```

### Keepalive (USER_STREAM)
```
PUT /api/v3/userDataStream
```
延长用户数据流有效期到60分钟之后。 建议每30分钟调用一次

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
listenKey | STRING | YES

**数据源:**
缓存

**响应:**
```javascript
{}
```

### 关闭用户数据流 (USER_STREAM)
```
DELETE /api/v3/userDataStream
```
关闭用户数据流。

**权重:**
1

**参数:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
listenKey | STRING | YES

**数据源:**
缓存

**响应:**
```javascript
{}
```

# 过滤器
过滤器，即Filter，定义了一系列交易规则。
共有两类，分别是针对交易对的过滤器`symbol filters`，和针对整个交易所的过滤器`exchange filters`

## 交易对过滤器
### PRICE_FILTER 价格过滤器
价格过滤器用于检测order订单中price参数的合法性
* `minPrice` 定义了 `price`/`stopPrice` 允许的最小值; `minPrice` == 0 的时候则失效。
* `maxPrice` 定义了 `price`/`stopPrice` 允许的最大值; `maxPrice` == 0 的时候则失效。
* `tickSize` 定义了 `price`/`stopPrice` 的步进间隔; `tickSize` == 0 的时候则失效。

以上每一项均可为0，为0时代表这一项不再做限制。

逻辑伪代码如下：
* `price` >= `minPrice`
* `price` <= `maxPrice`
* `price` % `tickSize` == 0

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "PRICE_FILTER",
    "minPrice": "0.00000100",
    "maxPrice": "100000.00000000",
    "tickSize": "0.00000100"
  }
```

### PERCENT_PRICE 价格振幅过滤器
可以理解为一个瞬时的涨跌停限制，不允许价格瞬间剧烈浮动。
`avgPriceMins` 指用过去几分钟的平均价格来计算价格基准. 0 表示用最新成交价格作为价格计准。

逻辑伪代码如下：
* `price` <= `weightedAveragePrice` * `multiplierUp`
* `price` >= `weightedAveragePrice` * `multiplierDown`

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "PERCENT_PRICE",
    "multiplierUp": "1.3000",
    "multiplierDown": "0.7000",
    "avgPriceMins": 5
  }
```

#### PERCENT_PRICE_BY_SIDE
`PERCENT_PRICE_BY_SIDE` 过滤器定义了基于交易对lastPrice的合法价格范围. 取决于`BUY`或者`SELL`, 价格范围可能有所不同.

买向订单需要满足:

* `Order price` <= `bidMultiplierUp` * `lastPrice`
* `Order price` >= `bidMultiplierDown` * `lastPrice`

卖向订单需要满足:

* `Order Price` <= `askMultiplierUp` * `lastPrice`
* `Order Price` >= `askMultiplierDown` * `lastPrice`


**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "PERCENT_PRICE_BY_SIDE",
    "bidMultiplierUp": "1.2",
    "bidMultiplierDown": "0.2",
    "askMultiplierUp": "5",
    "askMultiplierDown": "0.8",
    "avgPriceMins": 1
  }
```

### LOT_SIZE 订单尺寸
lots是拍卖术语，这个过滤器对订单中的`quantity`也就是数量参数进行合法性检查。包含三个部分：

* `minQty` 表示 `quantity`/`icebergQty` 允许的最小值.
* `maxQty` 表示 `quantity`/`icebergQty` 允许的最大值
* `stepSize` 表示 `quantity`/`icebergQty` 允许的步进值。

逻辑伪代码如下：

* `quantity` >= `minQty`
* `quantity` <= `maxQty`
* (`quantity`-`minQty`) % `stepSize` == 0

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "LOT_SIZE",
    "minQty": "0.00100000",
    "maxQty": "100000.00000000",
    "stepSize": "0.00100000"
  }
```

### MIN_NOTIONAL 最小金额
MIN_NOTIONAL过滤器定义了交易对订单所允许的最小名义价值(成交额)。
订单的名义价值是`价格`*`数量`。
如果是高级订单(比如止盈止损订单`STOP_LOSS_LIMIT`)，名义价值会按照`stopPrice` * `quantity`来计算。
如果是冰山订单，名义价值会按照`price` * `icebergQty`来计算。
`applyToMarket`确定 `MIN_NOTIONAL`过滤器是否也将应用于`MARKET`订单。   
由于`MARKET`订单没有价格，因此会在最后`avgPriceMins`分钟内使用平均价格。   
`avgPriceMins`是计算平均价格的分钟数。 0表示使用最后的价格。 


**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "MIN_NOTIONAL",
    "minNotional": "0.00100000",
    "applyToMarket": true,
    "avgPriceMins": 5
  }
```

### NOTIONAL 名义价值

**/exchangeInfo 响应中的格式:**

```javascript
{
   "filterType": "NOTIONAL",
   "minNotional": "10.00000000",
   "applyMinToMarket": false,
   "maxNotional": "10000.00000000",
   "applyMaxToMarket": false,
   "avgPriceMins": 5
}
```

名义价值过滤器(`NOTIONAL`)定义了订单在一个交易对上可以下单的名义价值区间.<br><br>
`applyMinToMarket` 定义了 `minNotional` 是否适用于市价单(`MARKET`)  <br>
`applyMaxToMarket` 定义了 `maxNotional` 是否适用于市价单(`MARKET`).

要通过此过滤器, 订单的名义价值 (单价 x 数量, `price * quantity`) 需要满足如下条件:

* `price * quantity` <= `maxNotional`
* `price * quantity` >= `minNotional`

对于市价单(`MARKET`), 用于计算的价格采用的是在 `avgPriceMins` 定义的时间之内的平均价.<br>
如果 `avgPriceMins` 为 0, 则采用最新的价格.

### ICEBERG_PARTS 冰山订单拆分数
`ICEBERG_PARTS` 代表冰山订单最多可以拆分成多少个小订单。
计算方法为 `向上取整(qty / icebergQty)`.

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "ICEBERG_PARTS",
    "limit": 10
  }
```

### MARKET_LOT_SIZE 市价订单尺寸
`MARKET_LOT_SIZE`过滤器为交易对上的`MARKET`订单定义了`数量`(即拍卖中的"手数")规则。 共有3部分：

* `minQty`定义了允许的最小`quantity`。
* `maxQty`定义了允许的最大数量。
* `stepSize`定义了可以增加/减少数量的间隔。

为了通过`market lot size`，`quantity`必须满足以下条件：

* `quantity` >= `minQty`
* `quantity` <= `maxQty`
* (`quantity`-`minQty`) % `stepSize` == 0

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "MARKET_LOT_SIZE",
    "minQty": "0.00100000",
    "maxQty": "100000.00000000",
    "stepSize": "0.00100000"
  }
```

### MAX_NUM_ORDERS 最多订单数
定义了某个交易对最多允许的挂单数量（不包括已关闭的订单）
普通订单与条件订单均计算在内

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "MAX_NUM_ORDERS",
    "maxNumOrders": 25
  }
```

### MAX_NUM_ALGO_ORDERS 最多条件单数
`MAX_NUM_ALGO_ORDERS`过滤器定义允许账户在交易对上开设的"algo"订单的最大数量。    
"Algo"订单是`STOP_LOSS`，`STOP_LOSS_LIMIT`，`TAKE_PROFIT`和`TAKE_PROFIT_LIMIT`止盈止损单。

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "MAX_NUM_ALGO_ORDERS",
    "maxNumAlgoOrders": 5
  }
```

### MAX_NUM_ICEBERG_ORDERS 最多冰山单数
`MAX_NUM_ICEBERG_ORDERS`过滤器定义了允许在交易对上开设账户的`ICEBERG`订单的最大数量。     
`ICEBERG`订单是icebergQty大于0的任何订单。

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "MAX_NUM_ICEBERG_ORDERS",
    "maxNumIcebergOrders": 5
  }
```

### MAX_POSITION 过滤器

这个过滤器定义账户允许的基于`base asset`的最大仓位。一个用户的仓位可以定义为如下资产的总和:
1. `base asset`的可用余额
1. `base asset`的锁定余额
1. 所有处于open的买单的数量总和

如果用户的仓位大于最大的允许仓位，买单会被拒绝。

如果一个订单的数量(`quantity`) 可能导致持有仓位溢出, 会触发过滤器 `MAX_POSITION`.

**/exchangeInfo 响应中的格式:**
```javascript
{
  "filterType": "MAX_POSITION",
  "maxPosition": "10.00000000"
}
```

### TRAILING_DELTA 过滤器


此过滤器定义了参数`trailingDelta`的最大和最小值.

下追踪止损订单, 需要满足条件:

对于 `STOP_LOSS BUY`, `STOP_LOSS_LIMIT_BUY`, `TAKE_PROFIT SELL` 和 `TAKE_PROFIT_LIMIT SELL` 订单:

* `trailingDelta` >= `minTrailingAboveDelta`
* `trailingDelta` <= `maxTrailingAboveDelta`

对于 `STOP_LOSS SELL`, `STOP_LOSS_LIMIT SELL`, `TAKE_PROFIT BUY`, 和 `TAKE_PROFIT_LIMIT BUY` 订单:

* `trailingDelta` >= `minTrailingBelowDelta`
* `trailingDelta` <= `maxTrailingBelowDelta`

 **/exchangeInfo format:**
```javascript
    {
          "filterType": "TRAILING_DELTA",
          "minTrailingAboveDelta": 10,
          "maxTrailingAboveDelta": 2000,
          "minTrailingBelowDelta": 10,
          "maxTrailingBelowDelta": 2000
   }
```

## 交易所级别过滤器
### EXCHANGE_MAX_NUM_ORDERS 最多订单数
`EXCHANGE_MAX_NUM_ORDERS`过滤器定义了允许在交易对上开设账户的最大订单数。    
请注意，此过滤器同时计算"algo"订单和常规订单。

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "EXCHANGE_MAX_NUM_ORDERS",
    "maxNumOrders": 1000
  }
```

### EXCHANGE_MAX_NUM_ALGO_ORDERS 最多条件单数
`EXCHANGE_MAX_ALGO_ORDERS`过滤器定义了允许在交易上开设账户的"algo"订单的最大数量。    
"Algo"订单是`STOP_LOSS`，`STOP_LOSS_LIMIT`，`TAKE_PROFIT`和`TAKE_PROFIT_LIMIT`订单。

**/exchangeInfo 响应中的格式:**
```javascript
  {
    "filterType": "EXCHANGE_MAX_NUM_ALGO_ORDERS",
    "maxNumAlgoOrders": 200
  }
```

### EXCHANGE_MAX_NUM_ICEBERG_ORDERS 冰山订单的最大订单数

此过滤器定义了允许账号持有的最大冰山订单数量.


**/exchangeInfo 响应中的格式:**

```javascript
{
  "filterType": "EXCHANGE_MAX_NUM_ICEBERG_ORDERS",
  "maxNumIcebergOrders": 10000
}
```

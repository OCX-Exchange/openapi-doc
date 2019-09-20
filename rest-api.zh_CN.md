## OCX 开发者接口 (API version 2)


**接口URI前缀:` /api/v2`**

**返回结果格式:`application/json`**



### Public/Private API

OCX开发者接口包含两类API

* Public API是不需要任何验证就可以使用的接口

* Private API是需要进行签名验证的接口

下表列出了两者的主要区别:

| item | Public API | Private API                                                  |
| ---- | ---------- | ------------------------------------------------------------ |
| 验证 | 无需验证   | 需要验证                                                     |
| 限制 | 无         | 对于每个用户, 平均一个请求/秒)。如果有更高需求可以联系OCX管理员 |
| 条件 | 直接使用   | 开发者需要前往管理中心创建API Token(access_key/secret_key)   |


### 公有接口

获取服务器时间戳
`GET /api/v2/timestamp`


获取所有市场
`GET /api/v2/markets`

获取单个市场
`GET /api/v2/markets/ocxeth`

获取所有市场行情数据
`GET /api/v2/tickers`

获取单个市场行情数据
`GET /api/v2/tickers/ocxeth`


### 私有接口

获取我的订单列表
`GET /api/v2/orders`

获取我的持仓信息
`GET /api/v2/accounts`

下单
`POST /api/v2/orders`

撤单
`POST /api/v2/orders/:order_id/cancel`

批量撤单
`POST /api/v2/orders/clear?market_code=btcusdt`


### 如何签名 (验证)

在给一个Private API请求签名之前, 你必须准备好你的`access key/secret key`.

在注册并认证通过后之后，只需访问API密钥页面就可以得到您的密钥。

所有的Private API都需要这3个用于身份验证的参数:

| 参数       | 解释                                                         |
| ---------- | ------------------------------------------------------------ |
| access_key | Your access_key                                              |
| tonce      | tonce是一个用正整数表示的时间戳，代表了从[Unix epoch](http://en.wikipedia.org/wiki/Unix_epoch)到当前时间所经过的毫秒(ms)数。tonce与服务器时间不得超过正负30秒。一个tonce只能使用一次。 |
| signature  | 使用你的secret key生成的签名                                 |

#### 签名步骤

1.通过组合HTTP方法, 请求地址和请求参数得到  `payload`.
`payload`需要进行`key`排序。

~~~Ruby
# canonical_verb 是HTTP方法，例如GET
# canonical_uri 是请求地址， 例如/api/v2/markets
# canonical_query 是请求参数通过&连接而成的字符串，参数包括access_key和tonce, 参数必须按照字母序排列，例如access_key=xxx&foo=bar&tonce=123456789
# 最后再把这三个字符串通过'|'字符连接起来，看起来就像这样:

# GET|/api/v2/markets|access_key=xxx&foo=bar&tonce=123456789
#
def payload
  "#{canonical_verb}|#{canonical_uri}|#{canonical_query}"
end
~~~



2. 对上述字符串使用`HMAC-SHA256`加密算法进行`hash`计算:

~~~
 hash = HMAC-SHA256(payload, secret_key).to_hex
~~~



#### 签名范例

假设我的secret key是"abc", 那么使用SHA256算法对上面例子中的payload计算HMAC的结果是(以hex表示)：

~~~ruby
  hash = HMAC-SHA256('GET|/api/v2/markets|access_key=xxx&foo=bar&tonce=123456789', 'abc').to_hex = '704f773b6b26772fd82bd3a8115079fb4f71d7baa1aad6b2922e99b17ed95cdc'
~~~



以`Ruby`为例， 签名的方法如下:

~~~ruby
secret = 'abc'
string_to_sign = 'GET|/api/v2/markets|access_key=xxx&foo=bar&tonce=123456789'
signature = OpenSSL::HMAC.hexdigest('sha256', secret, string_to_sign)
# => 704f773b6b26772fd82bd3a8115079fb4f71d7baa1aad6b2922e99b17ed95cdc
~~~



现在我们就可以这样来使用这个签名请求(以curl为例):

~~~shell
curl -X GET 'https://openapi.ocx.com/api/v2/markets?access_key=xxx&foo=bar&tonce=123456789&signature=704f773b6b26772fd82bd3a8115079fb4f71d7baa1aad6b2922e99b17ed95cdc'
~~~



### API



#### API 请求范例

以40000CNY的价格买入1BTC:

```shell
curl -X POST 'https://openapi.ocx.com/api/v2/orders' -d 'access_key=your_access_key&tonce=1234567&signature=computed_signature&market_code=btccny&price=40000&side=buy&volume=1'
```



#### API 返回结果

如果API调用失败，返回的请求会使用对应的HTTP status code, 同时返回包含了详细错误信息的JSON数据, 比如:

~~~json
{"error":{"code":1001,"message":"market does not have a valid value"}}
~~~

所有错误都遵循上面例子的格式，只是`code`和`message`不同。

`code`是OCX自定义的一个错误代码, 表明此错误的类别, message是具体的出错信息.

对于成功的API请求, OCX则会返回200作为HTTP status code, 同时返回请求的JSON数据.



#### API列表

1. `GET /api/v2/tickers `  获取OCX行情

* URL `https://openapi.ocx.com/api/v2/tickers`
* 请求示例

```json
# Request
GET https://openapi.ocx.com/api/v2/tickers

# Response
{
    "data": [{
        "low": "0.051",
        "high": "0.0537",
        "last" : "0.053",
        "market_code" : "ethbtc",
        "open" : "0.0517",
        "volume" : "454.3",
        "timestamp" : 1529275425,
        "sell": "0.07914108",
        "buy": "0.07902933",
    }]
}
```

* 返回结果

| 字段        | 类型    | 解释     |
| ----------- | ------- | -------- |
| low         | Decimal | 最低价格 |
| high        | Decimal | 最高价格 |
| last        | Decimal | 最新价格 |
| market_code | Decimal | 交易对   |
| open        | Decimal | 开盘价格 |
| volume      | Decimal | 成交量   |
| timestamp   | Int     | 时间戳   |



2. `GET /api/v2/markets`  获取可交易市场

* URL `https://openapi.ocx.com/api/v2/markets`
* 请求示例

```json
# Request
GET https://openapi.ocx.com/api/v2/markets

# Response
 {
     "data" : [{
         "code" : "ethbtc",
         "name" : "ETH/BTC",
         "base_unit" : "eth",
         "quote_unit" : "btc"
     }]
 }
```

| 字段       | 类型   | 解释           |
| ---------- | ------ | -------------- |
| code       | String | 交易对         |
| name       | String | 交易对（大写） |
| base_unit  | String | 基准货币       |
| quote_unit | String | 报价货币       |



3. `GET /api/v2/depth ` 获取市场深度

* URL `https://openapi.ocx.com/api/v2/depth`
* 请求参数

| 参数名       | 参数类型 | 是否必须 | 解释   |
| ----------- | -------- | -------- | ------ |
| market_code | string   | 是        | 交易对 |
| limit       | integer  | 否        | 默认 20 |

* 请求示例

```json
# Request
GET https://openapi.ocx.com/api/v2/depth?market_code=ethbtc

# Response
 {
     "data" : {
         "timestamp" : 1529275554,
         "asks" : [],
         "bids" : []
     }
 }
```

* 返回结果

| 字段      | 类型    | 解释     |
| --------- | ------- | -------- |
| timestamp | Integer | 时间戳   |
| asks      | Array   | 卖单列表 |
| bids      | Array   | 买单列表 |

PS: 具体详情请查看 `获取个人订单`接口

4. `GET /api/v2/orders`  获取个人订单

* URL `https://openapi.ocx.com/api/v2/orders?market=ethbtc`
* 请求参数

| 参数名      | 参数类型 | 是否必须 | 解释   |
| -----------| -------- | -------- | ------ |
| market     | String   | No    | 市场代码    |
| state | String   | No        | 状态 (wait, done, cancel) |
| limit | Integer   | No       | 每页条数 |
| page  | Integer   | No       | 页数 |
| order_by | String | No       | 排序 (asc,desc) |


* 请求示例
```Json
# Request
GET https://openapi.ocx.com/api/v2/orders
# Response
{
    "data": [{
        "id": 3,
        "side": "sell",
        "ord_type": "limit",
        "price": "0.0",
        "avg_price": "0.0",
        "state": "wait",
        "state_i18n": "WAIT",
        "market_code": "ethbtc",
        "market_name": "ETH/BTC",
        "market_base_unit": "eth",
        "market_quote_unit": "btc",
        "created_at": "2018-06-17T22:57:00Z",
        "volume": "0.1",
        "remaining_volume": "0.1",
        "executed_volume": "0.0"
    }]
}
```

* 返回结果

| 字段              | 类型    | 解释                                                         |
| :---------------- | ------- | ------------------------------------------------------------ |
| id                | Integer | 委托订单 ID                                                  |
| side              | String  | Buy/Sell, 代表买单/卖单.                                     |
| ord_type          | String  | limit: 限价单；                                             |
| price             | decimal | 价格                                                         |
| avg_price         | decimal | 平均价格                                                     |
| state             | String  | 委托订单状态: wait、done、cancel                             |
| state_i18n        | String  | 委托订单状态(国际化)                                         |
| market_code       | String  | 交易对                                                       |
| market_name       | String  | 订单参与的交易市场                                           |
| market_base_unit  | String  | 市场基准货币                                                 |
| market_quote_unit | String  | 市场报价货币                                                 |
| created_at        | String  | 下单时间, ISO8601格式                                        |
| volume            | decimal | 交易数量（买入、卖出）volume = remaining_volume + executed_volume |
| remaining_volume  | decimal | 未成交的数量                                                 |
| executed_volume   | decimal | 已成交的数量                                                 |

5. `GET /api/v2/orders/:id `获取订单详情

* URL `https://openapi.ocx.com/api/v2/orders/:id`
* 请求参数

| 参数名称 | 参数类型 | 是否必须 | 解释     |
| -------- | -------- | -------- | -------- |
| id       | Integer  | 是       | 委托单号 |

* 请求示例

```json
# Request
GET https://openapi.ocx.com/api/v2/orders/3
# Response
{
    "data": {
        "id": 3,
        "side": "sell",
        "ord_type": "limit",
        "price": "0.0",
        "avg_price": "0.0",
        "state": "wait",
        "state_i18n": "WAIT",
        "market_code": "ethbtc",
        "market_name": "ETH/BTC",
        "market_base_unit": "eth",
        "market_quote_unit": "btc",
        "created_at": "2018-06-17T22:57:00Z",
        "volume": "0.1",
        "remaining_volume": "0.1",
        "executed_volume": "0.0"
    }
}
```

* 返回结果

  | 字段              | 类型    | 解释                                                         |
  | :---------------- | ------- | ------------------------------------------------------------ |
  | id                | Integer | 委托订单 ID                                                  |
  | side              | String  | Buy/Sell, 代表买单/卖单.                                     |
  | ord_type          | String  | limit: 限价单；                                             |
  | price             | decimal | 价格                                                         |
  | avg_price         | decimal | 平均价格                                                     |
  | state             | String  | 委托订单状态: wait、done、cancel                             |
  | state_i18n        | String  | 委托订单状态(国际化)                                         |
  | market_code       | String  | 交易对                                                       |
  | market_name       | String  | 订单参与的交易市场                                           |
  | market_base_unit  | String  | 市场基准货币                                                 |
  | market_quote_unit | String  | 市场报价货币                                                 |
  | created_at        | String  | 下单时间, ISO8601格式                                        |
  | volume            | decimal | 交易数量（买入、卖出）volume = remaining_volume + executed_volume |
  | remaining_volume  | decimal | 未成交的数量                                                 |
  | executed_volume   | decimal | 已成交的数量                                                 |

State 说明

| 类型   | 说明                                                         |
| ------ | ------------------------------------------------------------ |
| wait   | 订单正在市场上挂单, 是一个active order, 此时订单可能部分成交或者尚未成交 |
| done   | 订单已经完全成交                                             |
| cancel | 订单已经被撤销                                               |



6. `POST /api/v2/orders `下单

* URL `https://openapi.ocx.com/api/v2/orders`

	请求参数

| 参数名      | 参数类型 | 是否必须 | 解释                       |
| ----------- | -------- | -------- | -------------------------- |
| market_code | String   | 是       | 交易对                     |
| side        | String   | 是       | 买卖类型：限价单(buy/sell) |
| price       | decimal  | 是       | 下单价格                   |
| volume      | decimal  | 否       | 交易数量                   |

* 请求示例

```json
# Request
POST https://openapi.ocx.com/api/v2/orders/
# Response
{
    "data": {
        "id": 3,
        "side": "sell",
        "ord_type": "limit",
        "price": "0.0",
        "avg_price": "0.0",
        "state": "wait",
        "state_i18n": "WAIT",
        "market_code": "ethbtc",
        "market_name": "ETH/BTC",
        "market_base_unit": "eth",
        "market_quote_unit": "btc",
        "created_at": "2018-06-17T22:57:00Z",
        "volume": "0.1",
        "remaining_volume": "0.1",
        "executed_volume": "0.0"
    }
}
```

* 返回结果

| 字段              | 类型    | 解释                                                         |
| :---------------- | ------- | ------------------------------------------------------------ |
| id                | Integer | 委托订单 ID                                                  |
| side              | String  | Buy/Sell, 代表买单/卖单.                                     |
| ord_type          | String  | limit: 限价单；                                             |
| price             | decimal | 价格                                                         |
| avg_price         | decimal | 平均价格                                                     |
| state             | String  | 委托订单状态: wait、done、cancel                             |
| state_i18n        | String  | 委托订单状态(国际化)                                         |
| market_code       | String  | 交易对                                                       |
| market_name       | String  | 订单参与的交易市场                                           |
| market_base_unit  | String  | 市场基准货币                                                 |
| market_quote_unit | String  | 市场报价货币                                                 |
| created_at        | String  | 下单时间, ISO8601格式   |
| volume            | decimal | 交易数量（买入、卖出）volume = remaining_volume + executed_volume |
| remaining_volume        | decimal  | 未成交的数量 |
| executed_volume         | decimal | 已成交的数量 |


7. `POST /api/v2/orders/:id/cancel `撤单

* URL `https://openapi.ocx.com/api/v2/orders/:id/cancel`
	 请求参数

| 字段 | 类型    | 是否必须 | 解释        |
| ---- | ------- | -------- | ----------- |
| id   | Integer | 是       | 委托订单 ID |

* 请求示例

```shell
# Request
POST https://openapi.ocx.com/api/v2/orders/1/cancel

# Response
返回已经正在撤单的订单信息
```

* 返回信息

| 字段              | 类型    | 解释                                                         |
| :---------------- | ------- | ------------------------------------------------------------ |
| id                | Integer | 委托订单 ID                                                  |
| side              | String  | Buy/Sell, 代表买单/卖单.                                     |
| ord_type          | String  | limit: 限价单；                                            |
| price             | decimal | 价格                                                         |
| avg_price         | decimal | 平均价格                                                     |
| state             | String  | 委托订单状态: wait、done、cancel                             |
| state_i18n        | String  | 委托订单状态(国际化)                                         |
| market_code       | String  | 交易对                                                       |
| market_name       | String  | 订单参与的交易市场                                           |
| market_base_unit  | String  | 市场基准货币                                                 |
| market_quote_unit | String  | 市场报价货币                                                 |
| created_at        | String  | 下单时间, ISO8601格式                                        |
| volume            | decimal | 交易数量（买入、卖出）volume = remaining_volume + executed_volume |
| remaining_volume  | decimal | 未成交的数量                                                 |
| executed_volume   | decimal | 已成交的数量                                                 |

* 注意事项

**取消挂单是一个异步操作,api成功返回仅代表取消请求已经成功提交,服务器正在处理,不代表订单已经取消. 当你的挂单有尚未处理的成交(trade)事务,或者取消请求队列繁忙时,该订单会延迟取消. api返回被取消的订单,返回结果中的订单不一定处于取消状态,你的代码不应该依赖api返回结果,而应该通过/api/v2/order来得到该订单的最新状态.**

8.  `POST /api/v2/orders/clear `批量撤单

* URL `https://openapi.ocx.com/api/v2/orders/clear`
* 请求示例

```json
# Request
POST https://openapi.ocx.com/api/v2/orders/clear

# Response
返回已经正在撤单的订单信息
```

* 返回数据

| 字段              | 类型    | 解释                                                         |
| :---------------- | ------- | ------------------------------------------------------------ |
| id                | Integer | 委托订单 ID                                                  |
| side              | String  | Buy/Sell, 代表买单/卖单.                                     |
| ord_type          | String  | limit: 限价单；                                             |
| price             | decimal | 价格                                                         |
| avg_price         | decimal | 平均价格                                                     |
| state             | String  | 委托订单状态: wait、done、cancel                             |
| state_i18n        | String  | 委托订单状态(国际化)                                         |
| market_code       | String  | 交易对                                                       |
| market_name       | String  | 订单参与的交易市场                                           |
| market_base_unit  | String  | 市场基准货币                                                 |
| market_quote_unit | String  | 市场报价货币                                                 |
| created_at        | String  | 下单时间, ISO8601格式                                        |
| volume            | decimal | 交易数量（买入、卖出）volume = remaining_volume + executed_volume |
| remaining_volume  | decimal | 未成交的数量                                                 |
| executed_volume   | decimal | 已成交的数量                                                 |

* 注意事项

**取消你所有的挂单. 取消挂单是一个异步操作, api成功返回代表取消请求已经提交,服务器正在处理. api返回的结果是你当前挂单的集合,结果中的订单不一定处于取消状态.**

9. `GET /api/v2/accounts`  个人资产

* URL `https://openapi.ocx.com/api/v2/accounts`
* 请求参数

| 字段  | 类型    | 是否必须 | 解释         |
| ---- | ------- | -------- | ---------  |
| currency_code  | String | 否 | 某个币种的持仓 |


* 请求示例

```json
# Request
GET https://openapi.ocx.com/api/v2/accounts
# Response
{
    "data" : [{
        "currency_code" : "btc",
        "balance" : "10.0",
        "locked" : "0.0"
    }, {
        "currency_code" : "eth",
        "balance" : "0.0",
        "locked" : "0.0"
    }]
}
```

* 返回参数

| 字段          | 类型    | 解释     |
| ------------- | ------- | -------- |
| currency_code | String  | 币种     |
| balance       | decimal | 账户余额 |
| locked        | decimal | 被锁金额 |

PS: `balance`为账户的余额， 不包含用户`locked`的金额.

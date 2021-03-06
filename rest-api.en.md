## Official documentation for OCX Exchange APIs

** The base endpoint: https://openapi.ocx.com **

** API URI prefix: `/api/v2` **

** Content Type: `application/json`**


### Public/Private API

OCX Exchange APIs offers two kinds of API accessibility:

* The Public API opens for public with request rate limit

* The Private API requires HMAC SHA256 signature when accessing API endpoints

Difference list as below:

| item | Public API | Private API                                        |
| ---- | ---------- | -------------------------------------------------- |
| Authentication  | No           | Required                              |
| Constraint      | No           | One API request per second. Contact OCX Exchange if you need more |
| Usage condition | Use Directly | Get API token(access_key/secret_key) from application management section of offical website |

### Public APIs

1. Get server unix timestamp `GET /api/v2/timestamp`

2. Get all open markets `GET /api/v2/markets`

3. Get single market `GET /api/v2/markets/ocxeth`

4. Get all open market tickers `GET /api/v2/tickers`

5. Get single market ticker `GET /api/v2/tickers/ocxeth`

6. Get orderbook of specific market `GET /api/v2/depth?market_code=ethbtc`


### Private APIs

1. Get my orders `GET /api/v2/orders`

2. Get my assets `GET /api/v2/accounts`

3. Place an order `POST /api/v2/orders`

4. Cancel single order `POST /api/v2/orders/:order_id/cancel`

5. Cancel batch orders `POST /api/v2/orders/clear`


### How to make signature

Get your <access_key> and <secret_key> as prerequisite, then make API requests with three authentication parameters and zero or more API specific parameters. The three parameters are:

| params     | annotation                                                   |
| ---------- | ------------------------------------------------------------ |
| access_key | Your access_key                                              |
| tonce      | timestamp in integer, stands for milliseconds elapsed since Unix epoch. Tonce must be within 30 seconds of server's current time. Each tonce can only be used once |
| signature  | Signature of the API request, generated by  your secret key  |

#### Step of HMAC SHA256 signature

Signature is a hash of the request (in canonical string form), payload is a combination of HTTP verb, uri, and query string:

`hash = HMAC-SHA256(payload, secret_key).to_hex`

```ruby
# canonical_verb is HTTP verb like GET/POST in upcase
# canonical_uri is request path like /api/v2/markets
# canonical_query is the request query sorted in alphabetica order, including access_key and tonce, e.g. access_key=xxx&foo=bar&tonce=123456789
# The combined string looks like: GET|/api/v2/markets|access_key=xxx&foo=bar&tonce=123456789
def payload
  "#{canonical_verb}|#{canonical_uri}|#{canonical_query}"
end
```

Note: query parameters are sorted in payload, so it must be "access_key=xxx&foo=bar" not "foo=bar&&access_key=xxx"
Suppose my secret key is 'yyy', then the result of HMAC-SHA256 of above payload is:

```ruby
hash = HMAC-SHA256('GET|/api/v2/markets|access_key=xxx&foo=bar&tonce=123456789', 'yyy').to_hex
     # 'e324059be4491ed8e528aa7b8735af1e96547fbec96db962d51feb7bf1b64dee'
```

Now we hav a signed request which can be used like this:

`curl -X GET 'https://openapi.ocx.com/api/v2/markets?access_key=xxx&foo=bar&tonce=123456789&signature=e324059be4491ed8e528aa7b8735af1e96547fbec96db962d51feb7bf1b64dee'`

### API examples

#### Place orders

Buy 1BTC with 10000USDT

```shell
curl -XPOST 'https://openapi.ocx.com/api/v2/orders' -d
'access_key=<your_access_key>&tonce=1234567&signature=<computed_signature>&market_code=btcusdt&price=10000&side=buy&volume=1'
```

#### Response

Corresponding HTTP status code will be used in API response. More detailed error message is included with JSON structure on failed request, for example:
All errors follow the message format above, only differentiates in code and message. Code is a OCX-defined error code, indicates the error's category, message is human-readable details.

```json
{ "error":{ "code":1001, "message":"market does not have a valid value" }}
```

#### APIs

1. Get all market tickers by `GET /api/v2/tickers`

JSON Response

```json
{
    "data": [{
        "market_code" : "ethbtc", // <string>
        "high": "0.0537",         // <string>
        "low": "0.051",           // <string>
        "open" : "0.0517",        // <string>
        "last" : "0.053",         // <string>
        "volume" : "454.3",       // <string>
        "sell": "0.07914108",     // <string>
        "buy": "0.07902933",      // <string>
        "timestamp" : 1529275425, // <integer>
    }, ...]
}
```

2. Get open markets `GET /api/v2/markets`

JSON Response

```json
 {
     "data" : [{
         "code" : "ethbtc",
         "name" : "ETH/BTC",
         "base_unit" : "eth",
         "quote_unit" : "btc"
     }]
 }
```

3. Get market price depth `GET /api/v2/depth?market_code=ethbtc&limit=20`

JSON Response

```json
 {
     "data" : {
         "timestamp" : 1529275554,   // <integer>
         "asks" : [
          ["0.02137287", "2.736673"] // [price: <string>, volume: <string>]
        ],
        "bids" : [
          ["0.02132421", "8.526228"]
        ]
     }
 }
```

4. Get my orders `GET /api/v2/orders?state=<wait>&limit=10&page=1&order_by=asc`

Supported query parameters:

| param       | key      | required | Note   |
| ----------- | -------- | -------- | ------ |
| market      | string   | No       |        |
| state | string    | No       | wait, done, cancel |
| limit | integer   | No       |             |
| page  | integer   | No       |             |
| order_by | string | No       |   asc,desc  |

JSON Response

```json
{
    "data": [{
        "id": 3,                         // <integer>
        "side": "sell",                  // <string> buy or sell
        "ord_type": "limit",             // <string> only limit order supported for now
        "price": "1.0",                  // <string>
        "avg_price": "0.0",              // <string>
        "state": "wait",                 // <string>
        "state_i18n": "WAIT",            // <string>
        "market_code": "ethbtc",         // <string>
        "market_name": "ETH/BTC",        // <string>
        "market_base_unit": "eth",       // <string>
        "market_quote_unit": "btc",      // <string>
        "created_at": "2018-06-17T22:57:00Z", // <string> place order time, formatted as ISO8601
        "volume": "0.1",                      // <string> volume = remaining_volume + executed_volume
        "remaining_volume": "0.1",            // <string>
        "executed_volume": "0.0"              // <string>
    }]
}
```


5. Get single order `GET /api/v2/orders/:id`


JSON Response

```json
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

6. Place an order `POST /api/v2/orders`


Params

| Key         | Type     | Required | Note           |
| ----------- | -------- | -------- | -------------- |
| market_code | string   | YES      |                |
| side        | string   | YES      | buy or sell    |
| price       | string   | YES      |                |
| volume      | string   | NO       |                |

JSON Response

```json
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

7. Cancel single order `POST /api/v2/orders/:id/cancel`

NOTE: Cancel an order is processed as async operation, request `/api/v2/orders/:id` for the correct status to check whether it's cancelled or not

JSON Response

```json
{
    "data": {
        "id": 3,
        "side": "sell",
        "ord_type": "limit",
        "price": "0.0",
        "avg_price": "0.0",
        "state": "cancel",
        "state_i18n": "CANCEL",
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

8. Cancel batch orders `POST /api/v2/orders/clear`

Parameters:

| Key           | Type     | Required | Note      |
| -----------   | -------- | -------- | --------- |
| market_code   | string   | NO       | All market orders will be canceled without market_code |

JSON Response

```json
{
    "data": [{
        "id": 3,
        "side": "sell",
        "ord_type": "limit",
        "price": "0.0",
        "avg_price": "0.0",
        "state": "cancel",
        "state_i18n": "CANCEL",
        "market_code": "ethbtc",
        "market_name": "ETH/BTC",
        "market_base_unit": "eth",
        "market_quote_unit": "btc",
        "created_at": "2018-06-17T22:57:00Z",
        "volume": "0.1",
        "remaining_volume": "0.1",
        "executed_volume": "0.0"
    }
    // ...
  ]
}
```

9. Get your assets `GET /api/v2/accounts`

Parameters:

| Key           | Type     | Required | Note      |
| -----------   | -------- | -------- | --------- |
| currency_code | string   | NO       |           |


JSON Response

```json
{
    "data" : [{
        "currency_code": "btc", // <string>
        "balance": "10.0",      // <string>
        "locked": "0.0"         // <string> balance transfer to locked when user place an order or submit withdrawal
    }, {
        "currency_code": "eth",
        "balance": "0.0",
        "locked": "0.0"
    }]
}
```

10. `GET /api/v2/accounts/all`  all assets

* URL `https://openapi.ocx.com/api/v2/accounts/all`
* Parameters:

| key            | type    | Required | Note       |
| -------------- | ------- | -------- | ---------  |
| currency_code  | String  | NO       |            |


* example

```json
# Request
GET https://openapi.ocx.com/api/v2/accounts/all
# Response
{
    "data" : {
        accounts: [{
            "currency_code" : "btc",
            "balance" : "10.0",
            "locked" : "0.0"
        }],
        spot_accounts: [{
            "currency_code" : "btc",
            "balance" : "10.0",
            "locked" : "0.0"
        }],
        otc_accounts: [{
            "currency_code" : "btc",
            "balance" : "10.0",
            "locked" : "0.0"
        }]
    }
}
```

* Response

| Key           | Type          | Note                          |
| ------------- | ------------- | ----------------------------- |
| accounts      | Account Array | Accounts(withdraws/deposits)  |
| spot_accounts | Account Array | Spot Accounts (trade)         |
| otc_accounts  | Account Array | Otc Account (otc)             |


11. `POST /api/v2/accounts/transfer`  Transfer your assets

* URL `https://openapi.ocx.com/api/v2/accounts/all`
* Parameters:

| Key               | Type    | Required | Note                                             |
| ----              | ------- | -------- | -----------------------------------------------  |
| from_account_type | String  | Y        |  choose in (accounts spot_accounts otc_accounts) |
| to_account_type   | String  | Y        |  choose in (accounts spot_accounts otc_accounts) |
| amount            | String  | Y        |                                                  |
| currency_code     | String  | Y        |                                                  |


* example

```json
# Request
POST https://openapi.ocx.com/api/v2/accounts/transfer
# Response
transfer success
{
    "data" : {
       success: true
    }
}

transfer failure
{
    "data" : {
       error: {message: 'Balance Not Enough'}
    }
}
```
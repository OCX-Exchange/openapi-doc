# OCX Websocket streams (2018-10-01)

OCX websocket 推送接口采用兼容 [PUSHER](https://pusher.com/) 的服务器端实现，这意味着 PUSHER 提供的[客户端](https://pusher.com/docs/libraries) 都可以用来连接 OCX Websocket Streams 推送服务。我们以 [pusher-js](https://github.com/pusher/pusher-js) 为例：


## Connection

创建连接，一个 APP_KEY 建议维护一个 connection，否则 connections 数超限会触发 rate limit 错误。

```js
const socket = new Pusher(APP_KEY, {
  cluster: APP_CLUSTER,
});
```

连接断开

```js
socket.disconnect();
```

## Channel

### Public channel

公有频道指跟特定用户无关的推送信息，比如交易对实时价格(tickers)或者交易对深度信息(depth)

首先订阅频道

```js
const channel = socket.subscribe('my-channel');
```

取消订阅频道

```js
socket.unsubscribe('my-channel');
```

### Private channel

当用户委托下单后需要收到委托单的成交推送，则需要用到私有频道，私有频道以 'private-' 为前缀

```js
const channel = socket.subscribe('private-my-channel');
```

同理，取消订阅私有频道

```js
socket.unsubscribe('private-my-channel');
```

## Event

最后，需要绑定某个频道上的事件(Event)才能真正实现数据交换

```js
channel.bind('new-message', function (data) {
  console.log(data.message);
});
```

同理，取消对某个事件的绑定

```js
channel.unbind('new-message');
```

## OCX Channel List

OCX Websocket Streams 提供以下几种数据频道：

* { channel: 'tickers@global@3000ms', event: 'updateTickers'}, 即每隔 3 秒推送全部交易对的 ticker 数据

返回结果示例：

```js
{
	'channel': 'tickers@global@3000ms',
	'event': 'updateTickers',
	'data': [
		{
			's': 'ethbtc',     // symbol, <string>, 交易对代码
			'o': '0.0001',     // open,   <string>, 开盘价
			'c': '0.0002',     // close,  <string>, 收盘价
			'h': '0.0003',     // high,   <string>, 最高价
			'l': '0.00005',    // low,    <string>, 最低价
			'v': '1000000',    // volume, <string>, 成交量
			'E': '1538901885'  // Timestamp, <Integer>, 10 位的 Unix Epoch Time, 精确到秒
		}
	]
}
```

* { channel: 'depth@ocxeth@10', event: 'updateDepth' }, 即当深度变化时会以买卖各 10 档进行推送

返回结果示例

```js
{
	'channel': 'depth@ethbtc@10',
	'event': 'updateDepth',
	'data': {
		'asks': [
			['0.0002', '100'],  // [ask_price_1<string>, ask_volume_1<string>] => [卖1价, 卖1量]
			...
		],
		'bids': [
			['0.0001', '200'],  // [bid_price_1<string>, bid_volume_1<string>] => [买1价, 买1量]
			...
		]
	}
```

* { channel: 'private-COINX2TXAEIWX', event: 'order|account|accounts' }, 其中 COINX2TXAEIWX 指用户标识符, 该标识符可以通过网站 账户中心 -> 我的邀请 -> 我的推荐ID 可以获得

返回结果示例

1. 委托单成交变化

```js
{
	'channel': 'private-COINX2TXAEIWX',
	'event': 'order',
	'data': {
		'id': 1234567,       // <Integer>, 委托单 id
		'market': 'ocxeth',  // <string>,  交易对代号
		'kind': 'ask',       // <string>,  委托单类型, ask=卖，bid=买
		'price': '0.00001',  // <string>,  委托单价格
		'state': 'wait',     // <string>,  委托状态, wait=未成交, cancel=取消, done=完成
		'volume': '100',     // <string>,  目前未成交委托数量
		'origin_volume': '100',  // <string>, 委托数量
	}
}
```

2. 特定币种持仓账户变化

```js
{
	'channel': 'private-COINX2TXAEIWX',
	'event': 'account',
	'data': {
		'balance': '10000.1234', // <string>, 持仓余额
		'locked':   '0.0',       // <string>, 持仓冻结
		'currency': 'ocx'        // <string>, 持仓币种
	}
}
```

## 连接授权

公有频道建立连接前先通过官方渠道获得

* APP_KEY
* APP_CLUSTER
* WS_HOST
* WSS_PORT

如果需要连接私有频道需要通过 API 服务提供的 <AUTH_URL> 并传入 <AUTH_TOKEN> 进行授权连接

* AUTH_URL
* AUTH_TOKEN

其中获取 AUTH_TOKEN, 请通过网站 -> 账户中心 -> 我的 API，点击查看按钮会展示 apiKey 和 secretKey, 其中 apiKey 即我们所需的 AUTH_TOKEN

## 完整示例

```js
const pusher = new Pusher(env.APP_KEY, {
		cluster: env.APP_CLUSTER,
		wsHost: env.WS_HOST,
		wssPort: env.WSS_PROT
		encrypted: true,
		forceTLS: true,
		authEndpoint: AUTH_URL,
		auth: {
				headers: { Authorization: <AUTH_TOKEN> }
		},
});
const channel = socket.subscribe('tickers@global@3000ms');
channel.bind('updateTickers', function(data) {
	console.log(data);
});

// clear stuff
channel.unbind('updateTickers');
channel.unsubscribe();
pusher.disconnect();
```

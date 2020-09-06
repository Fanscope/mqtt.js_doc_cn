![mqtt.js](https://raw.githubusercontent.com/mqttjs/MQTT.js/137ee0e3940c1f01049a30248c70f24dc6e6f829/MQTT.js.png)
=======

![Github Test Status](https://github.com/mqttjs/MQTT.js/workflows/MQTT.js%20CI/badge.svg)  [![codecov](https://codecov.io/gh/mqttjs/MQTT.js/branch/master/graph/badge.svg)](https://codecov.io/gh/mqttjs/MQTT.js)
MQTT.js 是基于 [MQTT](http://mqtt.org/) 协议的一个客户端代码库，用 JavaScript 编写，适用于 node.js 与浏览器。

* [更新日志](#notes)
* [安装](#install)
* [示例代码](#example)
* [命令行工具](#cli)
* [API](#api)
* [浏览器](#browser)
* [微信小程序](#weapp)
* [关于 QoS](#qos)
* [TypeScript](#typescript)
* [Contributing](#contributing)
* [License](#license)

MQTT.js 是一个开源项目，相关解释： [Contributing](#contributing)

[![JavaScript Style
Guide](https://cdn.rawgit.com/feross/standard/master/badge.svg)](https://github.com/feross/standard)

<a name="notes"></a>
## 对现有用户的一些重要说明

__v4.0.0__ (Released 04/2020) removes support for all end of life node versions, and now supports node v12 and v14. It also adds improvements to
debug logging, along with some feature additions.

As a __breaking change__, by default a error handler is built into the MQTT.js client, so if any
errors are emitted and the user has not created an event handler on the client for errors, the client will
not break as a result of unhandled errors. Additionally, typical TLS errors like `ECONNREFUSED`, `ECONNRESET` have been
added to a list of TLS errors that will be emitted from the MQTT.js client, and so can be handled as connection errors.

__v3.0.0__ adds support for MQTT 5, support for node v10.x, and many fixes to improve reliability.

__Note:__ MQTT v5 support is experimental as it has not been implemented by brokers yet.

__v2.0.0__ removes support for node v0.8, v0.10 and v0.12, and it is 3x faster in sending
packets. It also removes all the deprecated functionality in v1.0.0,
mainly `mqtt.createConnection` and `mqtt.Server`. From v2.0.0,
subscriptions are restored upon reconnection if `clean: true`.
v1.x.x is now in *LTS*, and it will keep being supported as long as
there are v0.8, v0.10 and v0.12 users.

As a __breaking change__, the `encoding` option in the old client is
removed, and now everything is UTF-8 with the exception of the
`password` in the CONNECT message and `payload` in the PUBLISH message,
which are `Buffer`.

Another __breaking change__ is that MQTT.js now defaults to MQTT v3.1.1,
so to support old brokers, please read the [client options doc](#client).

__v1.0.0__ improves the overall architecture of the project, which is now
split into three components: MQTT.js keeps the Client,
[mqtt-connection](http://npm.im/mqtt-connection) includes the barebone
Connection code for server-side usage, and [mqtt-packet](http://npm.im/mqtt-packet)
includes the protocol parser and generator. The new Client improves
performance by a 30% factor, embeds Websocket support
([MOWS](http://npm.im/mows) is now deprecated), and it has a better
support for QoS 1 and 2. The previous API is still supported but
deprecated, as such, it is not documented in this README.

<a name="install"></a>
## 安装

```sh
npm install mqtt --save
```

<a name="example"></a>
## 示例

简单起见，我们把订阅端和发布端放到同一个文件中：

```js
var mqtt = require('mqtt')
var client  = mqtt.connect('mqtt://test.mosquitto.org')

client.on('connect', function () {
  client.subscribe('presence', function (err) {
    if (!err) {
      client.publish('presence', 'Hello mqtt')
    }
  })
})

client.on('message', function (topic, message) {
  // message is Buffer
  console.log(message.toString())
  client.end()
})
```

输出：
```
Hello mqtt
```

如果你想运行自己的 MQTT 服务端 (broker), 你可以使用 [Mosquitto](http://mosquitto.org) 或者[Mosca](http://mcollina.github.io/mosca/)。你也可以使用公共的测试实例: test.mosquitto.org 和 test.mosca.io。

如果想要装载一个独立的服务端，你可以尝试 [mqtt-connection](https://www.npmjs.com/package/mqtt-connection)。

在浏览器中使用 MQTT.js，详细说明： [browserify](#browserify)

<a name="promises"></a>
## Promise 支持

如果你想在JavaScript中使用新的 [async-await](https://blog.risingstack.com/async-await-node-js-7-nightly/) 语法，或相较于 callback 更偏好 Promise，[async-mqtt](https://github.com/mqttjs/async-mqtt) 版本尽可能地使用了 Promise 语法特性。

<a name="cli"></a>
## 命令行工具

MQTT.js 提供了与服务端（broker）交互的命令行指令。
为了保证在您的目录下可用, 需要全局安装 mqtt.js：

```sh
npm install mqtt -g
```

然后，在一个终端（terminal）中：

```
mqtt sub -t 'hello' -h 'test.mosquitto.org' -v
```

另一个：

```
mqtt pub -t 'hello' -h 'test.mosquitto.org' -m 'from MQTT.js'
```

帮助文档请参考： `mqtt help <command>`

<a name="debug"></a>
## Debug Logs

MQTT.js uses the [debug](https://www.npmjs.com/package/debug#cmd) package for debugging purposes. To enable debug logs, add the following environment variable on runtime :
```ps
# (example using PowerShell, the VS Code default)
$env:DEBUG='mqttjs*'

```

<a name="reconnecting"></a>
## 关于断线重连

连接断开后的重连和相关施对任何 websocket 连接工具来说都是一个重要部分，MQTT 有内置的重连支持，同时也可以根据具体应用的需要进行配置。

#### Refresh Authentication Options / Signed Urls with `transformWsUrl` (Websocket Only)

当一个 mqtt 连接断开并进行重连的时候，在现有认证机制下保持原状是基本要求。如有的应用会基于现有的连接，通过带有配置选项的令牌，也有一些云服务要求每个连接要有一个带签名的 url。

另外若重连发生在应用的生命周期中，原先的认证数据有可能会过期。

在重连的时候我们可以使用钩子函数 `transformWsUrl` 来处理连接url或客户端配置选项可以解决这个问题。

示例（更新每个重连连接的 clientId & username）：
```
    const transformWsUrl = (url, options, client) => {
      client.options.username = `token=${this.get_current_auth_token()}`;
      client.options.clientId = `${this.get_updated_clientId()}`;

      return `${this.get_signed_cloud_url(url)`;
    }

    const connection = await mqtt.connectAsync(<wss url>, {
      ...,
      transformWsUrl: transformUrl,
    });

```

现在每当一个新的 WebSocket 连接被打开（但愿不会太频繁），我们都会得到一个新的签名 url 或令牌数据。

注: 目前这个钩子不支持 promises，即为了使用最新的令牌，你必须要运行一些处理应用级认证刷新的外部机制处理，这样 websocket 连接就能简单地获取最新的有效令牌或签名 url。


#### 通过 `reconnectPeriod` 选项启用重连

为了保证mqtt客户端掉线后自动尝试重连，你必须将客户端的 `reconnectPeriod` 属性设置为一个大于0的值。值为0时将禁止重连，掉线即会终止最后的连接。

默认值为1000毫秒，表示掉线后1秒后开始尝试重新连接。



<a name="api"></a>
## API

  * <a href="#connect"><code>mqtt.<b>connect()</b></code></a>
  * <a href="#client"><code>mqtt.<b>Client()</b></code></a>
  * <a href="#publish"><code>mqtt.Client#<b>publish()</b></code></a>
  * <a href="#subscribe"><code>mqtt.Client#<b>subscribe()</b></code></a>
  * <a href="#unsubscribe"><code>mqtt.Client#<b>unsubscribe()</b></code></a>
  * <a href="#end"><code>mqtt.Client#<b>end()</b></code></a>
  * <a href="#removeOutgoingMessage"><code>mqtt.Client#<b>removeOutgoingMessage()</b></code></a>
  * <a href="#reconnect"><code>mqtt.Client#<b>reconnect()</b></code></a>
  * <a href="#handleMessage"><code>mqtt.Client#<b>handleMessage()</b></code></a>
  * <a href="#connected"><code>mqtt.Client#<b>connected</b></code></a>
  * <a href="#reconnecting"><code>mqtt.Client#<b>reconnecting</b></code></a>
  * <a href="#getLastMessageId"><code>mqtt.Client#<b>getLastMessageId()</b></code></a>
  * <a href="#store"><code>mqtt.<b>Store()</b></code></a>
  * <a href="#put"><code>mqtt.Store#<b>put()</b></code></a>
  * <a href="#del"><code>mqtt.Store#<b>del()</b></code></a>
  * <a href="#createStream"><code>mqtt.Store#<b>createStream()</b></code></a>
  * <a href="#close"><code>mqtt.Store#<b>close()</b></code></a>

-------------------------------------------------------
<a name="connect"></a>
### mqtt.connect([url], options)

根据特定的url和选项连接服务端 (broker)，并返回一个 [Client](#client) 实例。

URL可以用于以下协议：'mqtt', 'mqtts', 'tcp', 'tls', 'ws', 'wss'。URL也可以是由 [`URL.parse()`](http://nodejs.org/api/url.html#url_url_parse_urlstr_parsequerystring_slashesdenotehost) 返回的一个对象，那种情况融合了这两个对象，也就是你可以只传一个同时有 URL 和连接选项的对象。

You can also specify a `servers` options with content: `[{ host:
'localhost', port: 1883 }, ... ]`, in that case that array is iterated
at every connect.

你也可以这样：`[{ host:'localhost', port: 1883 }, ... ]` 指明一个 `servers` 的选项，那种情况下每个连接都会进行数组遍历。

所有与MQTT相关的配置选项，参考 [Client](#client) 的构造函数

-------------------------------------------------------
<a name="client"></a>
### mqtt.Client(streamBuilder, options)

类 `Client` 包装了一个基于专门通信协议 (TCP, TLS, WebSocket, 等) 的客户-服务端 (broker) 连接。

`Client` 自动处理了以下内容：

* Regular server pings
* QoS 流
* 自动重连
* 连接前的消息发布

参数：

* `streamBuilder` 方法返回 `Stream` 的派生类，支持 `connect` 事件。通常为 `net.Socket`
* `options` 为客户端连接选项 (见 [connect packet](https://github.com/mcollina/mqtt-packet#connect))
  * `wsOptions`: WebSocket连接选项，默认情况是 `{}`，只适用于 WebSockets，可选项可参考：https://github.com/websockets/ws/blob/master/doc/ws.md.
  * `keepalive`: `60` 秒，设为 `0` 可禁止
  * `reschedulePings`: reschedule ping messages after sending packets (default `true`)
  * `clientId`: `'mqttjs_' + Math.random().toString(16).substr(2, 8)`
  * `protocolId`: `'MQTT'`
  * `protocolVersion`: `4`
  * `clean`: `true`, 设为 false 来接收 QoS 1 和 2 的离线消息
  * `reconnectPeriod`: `1000`毫秒，表明两次重连的间隔。设为`0`可禁止
  * `connectTimeout`: `30 * 1000` 毫秒，在收到一个 CONNACK 前的等待时间
  * `username`: 服务端 (broker) 需要的用户名，可选
  * `password`: 服务端 (broker) 需要的密码，可选
  * `incomingStore`: 存储接收中的包的 [Store](#store)
  * `outgoingStore`: 存储发送中的包的 [Store](#store)
  * `queueQoSZero`: 如果连接已销毁，将发送中的包组成队列（默认 `true`）
  * `customHandleAcks`: MQTT 5 feature of custom handling puback and pubrec packets. Its callback:
      ```js
        customHandleAcks: function(topic, message, packet, done) {/*some logic wit colling done(error, reasonCode)*/}
      ```
  * `properties`: MQTT 5.0. 参数，支持以下参数的 `object`：
    * `sessionExpiryInterval`: 表明 Session 过期间隔（秒） `number`
    * `receiveMaximum`: 表明接收的最大值 `number`
    * `maximumPacketSize`: 表明客户端期望接收的数据包的最大值 `number`
    * `topicAliasMaximum`: representing the Topic Alias Maximum value indicates the highest value that the Client will accept as a Topic Alias sent by the Server `number`
    * `requestResponseInformation`: 客户端通过本字段指明是否要求服务端在 CONNACK 中返回响应信息 `boolean`
    * `requestProblemInformation`: 客户端通过本字段指明是否要求服务端在失败时返回错误信息或用户参数 `boolean`
    * `userProperties`: The User Property is allowed to appear multiple times to represent multiple name, value pairs `object`,
    * `authenticationMethod`: the name of the authentication method used for extended authentication `string`,
    * `authenticationData`: 含验证数据的二进制数据 `binary`
  * `authPacket`: settings for auth packet `object`
  * `will`: a message that will sent by the broker automatically when
     the client disconnect badly. The format is:
    * `topic`: the topic to publish
    * `payload`: the message to publish
    * `qos`: the QoS
    * `retain`: the retain flag
    * `properties`: properties of will by MQTT 5.0:
      * `willDelayInterval`: representing the Will Delay Interval in seconds `number`,
      * `payloadFormatIndicator`: Will Message is UTF-8 Encoded Character Data or not `boolean`,
      * `messageExpiryInterval`: value is the lifetime of the Will Message in seconds and is sent as the Publication Expiry Interval when the Server publishes the Will Message `number`,
      * `contentType`: describing the content of the Will Message `string`,
      * `responseTopic`: String which is used as the Topic Name for a response message `string`,
      * `correlationData`: The Correlation Data is used by the sender of the Request Message to identify which request the Response Message is for when it is received `binary`,
      * `userProperties`: The User Property is allowed to appear multiple times to represent multiple name, value pairs `object`
  * `transformWsUrl` : optional `(url, options, client) => url` function
        For ws/wss protocols only. Can be used to implement signing
        urls which upon reconnect can have become expired.
  * `resubscribe` : if connection is broken and reconnects,
     subscribed topics are automatically subscribed again (default `true`)

In case mqtts (mqtt over tls) is required, the `options` object is
passed through to
[`tls.connect()`](http://nodejs.org/api/tls.html#tls_tls_connect_options_callback).
If you are using a **self-signed certificate**, pass the `rejectUnauthorized: false` option.
Beware that you are exposing yourself to man in the middle attacks, so it is a configuration
that is not recommended for production environments.

If you are connecting to a broker that supports only MQTT 3.1 (not
3.1.1 compliant), you should pass these additional options:

```js
{
  protocolId: 'MQIsdp',
  protocolVersion: 3
}
```

This is confirmed on RabbitMQ 3.2.4, and on Mosquitto < 1.3. Mosquitto
version 1.3 and 1.4 works fine without those.

#### 事件 `'connect'`

`function (connack) {}`

连接/重连接成功后触发 (i.e. connack rc=0).
* `connack` 接收到的消息确认包（connack packet）。若清除标记（`clean`）为 `false` 且服务器存在该 `clientId` 的会话（session）连接选项，标志 `connack.sessionPresent` 将会为 `true`。When that is the case, you may rely on stored session and prefer not to send subscribe commands for the client.

#### 事件 `'reconnect'`

`function () {}`

开始重连时触发

#### 事件 `'close'`

`function () {}`

断开连接后触发

#### 事件 `'disconnect'`

`function (packet) {}`

接收到服务器关闭连接的包后触发（MQTT 5.0）

#### 事件 `'offline'`

`function () {}`

离线时触发

#### 事件 `'error'`

`function (error) {}`

客户端无法连接 (i.e. connack rc != 0) 或解析错误时触发

以下的 TLS 错误 将会触发 `error` 事件：

* `ECONNREFUSED`
* `ECONNRESET`
* `EADDRINUSE`
* `ENOTFOUND`

#### 事件 `'end'`

`function () {}`

调用 <a href="#end"><code>mqtt.Client#<b>end()</b></code></a> 时触发

如果某回调函数（callback）传入到了 `mqtt.Client#end()`，事件将在回调函数返回时触发

#### 事件 `'message'`

`function (topic, message, packet) {}`

客户端接收到发布消息（publish packet）时触发
* `topic` 包话题（topic）
* `message` 包荷载（payload）
* `packet` 接收到的包，定义在：
  [mqtt-packet](https://github.com/mcollina/mqtt-packet#publish)

#### 事件 `'packetsend'`

`function (packet) {}`

客户端发送消息时触发，包括了 .published() 的包，也包括了 MQTT 管理订阅和连接所发送的包
* `packet` 同上

#### 事件 `'packetreceive'`

`function (packet) {}`

客户端接收到消息时触发。包括了订阅话题下的消息，也包括了 MQTT 管理订阅和连接所发送的包
* `packet` 同上

-------------------------------------------------------
<a name="publish"></a>
### mqtt.Client#publish(topic, message, [options], [callback])

在一个主题下发送消息

* `topic` 消息所发送的主题, `String`
* `message` 消息主体，`Buffer` 或 `String`
* `options` 选项配置，包括：
  * `qos` QoS 级别, `Number`, 默认 `0`
  * `retain` retain 标志，`Boolean`，默认 `false`
  * `dup` mark as duplicate flag, `Boolean`, default `false`
  * `properties`: MQTT 5.0 参数配置 `object`
    * `payloadFormatIndicator`: 荷载（payload）是否为 UTF-8 编码 `boolean`
    * `messageExpiryInterval`: 应用消息的有效时长，秒 `number`
    * `topicAlias`: 用于识别主题（topic）的值 `number`
    * `responseTopic`: 作为响应消息的主题字符串 `string`
    * `correlationData`: 用于发送方识别获取到的响应消息响应的是哪一个请求 `binary`,
    * `userProperties`: The User Property is allowed to appear multiple times to represent multiple name, value pairs `object`,
    * `subscriptionIdentifier`: representing the identifier of the subscription `number`,
    * `contentType`: String describing the content of the Application Message `string`
  * `cbStorePut` - `function ()`, 若 QoS 为 `1` 或 `2`，当消息被装入 `outgoingStore` 时触发
* `callback` - `function (err)`，在 QoS 处理结束后，或 QoS 为0的下一时间片（next tick）触发。 掉线时会抛一个异常。

-------------------------------------------------------
<a name="subscribe"></a>
### mqtt.Client#subscribe(topic/topic array/topic object, [options], [callback])

订阅一个或多个主题

* `topic` 为一个 `String` 类型的主题或若干个主题组成的 `Array`。也可以传入一个对象，其中键为主题，值为对应的QoS。例如 `{'test1': {qos: 0}, 'test2': {qos: 1}}`。MQTT `topic` 支持通配符 (`+` - for single level and `#` - for multi level)
* `options` 订阅时的选项，包括：
  * `qos` QoS 订阅级别, 默认 0
  * `nl` No Local MQTT 5.0 flag (If the value is true, Application Messages MUST NOT be forwarded to a connection with a ClientID equal to the ClientID of the publishing connection)
  * `rap` Retain as Published MQTT 5.0 flag (If true, Application Messages forwarded using this subscription keep the RETAIN flag they were published with. If false, Application Messages forwarded using this subscription have the RETAIN flag set to 0.)
  * `rh` Retain Handling MQTT 5.0 (This option specifies whether retained messages are sent when the subscription is established.)
  * `properties`: `object`
    * `subscriptionIdentifier`:  representing the identifier of the subscription `number`,
    * `userProperties`: The User Property is allowed to appear multiple times to represent multiple name, value pairs `object`
* `callback` - `function (err, granted)`
  callback fired on suback where:
  * `err` a subscription error or an error that occurs when client is disconnecting
  * `granted` is an array of `{topic, qos}` where:
    * `topic` is a subscribed to topic
    * `qos` is the granted QoS level on it

-------------------------------------------------------
<a name="unsubscribe"></a>
### mqtt.Client#unsubscribe(topic/topic array, [options], [callback])

取消订阅一个或多个主题

* `topic` 取消的主题字符串 `String` 或主题数组
* `options`: 取消订阅时的选项
  * `properties`: `object`
      * `userProperties`: The User Property is allowed to appear multiple times to represent multiple name, value pairs `object`
* `callback` - `function (err)`, fired on unsuback. An error occurs if client is disconnecting.

-------------------------------------------------------
<a name="end"></a>
### mqtt.Client#end([force], [options], [cb])

关闭客户端，支持以下配置：

* `force`: 可选参数，设为 `true` 时立即关闭客户端，不对传输中的包（packet）进行验证。
* `options`: 离线选项。
  * `reasonCode`: 离线的原因码 `number`
  * `properties`: `object`
    * `sessionExpiryInterval`: 表明会话（session）的过期周期，秒 `number`,
    * `reasonString`: 离线原因 `string`,
    * `userProperties`: The User Property is allowed to appear multiple times to represent multiple name, value pairs `object`,
    * `serverReference`: String which can be used by the Client to identify another Server to use `string`
* `cb`: will be called when the client is closed. This parameter is
  optional.

-------------------------------------------------------
<a name="removeOutgoingMessage"></a>
### mqtt.Client#removeOutgoingMessage(mid)

在发送存储区（outgoingStore）中移除一个消息

发送回调会在消息移除后以 Error('Message removed') 进行调用

在此方法调用后，messageId 会被释放，并可重用

* `mid`: 发送存储区中的 messageId

-------------------------------------------------------
<a name="reconnect"></a>
### mqtt.Client#reconnect()

再次进行连接，和 connect() 参数一致

-------------------------------------------------------
<a name="handleMessage"></a>
### mqtt.Client#handleMessage(packet, callback)

Handle messages with backpressure support, one at a time.
Override at will, but __always call `callback`__, or the client
will hang.

-------------------------------------------------------
<a name="connected"></a>
### mqtt.Client#connected

Boolean：已连接为 `true`，否则为 `false`

-------------------------------------------------------
<a name="getLastMessageId"></a>
### mqtt.Client#getLastMessageId()

Number：获取上一条已发送消息的 messageId

-------------------------------------------------------
<a name="reconnecting"></a>
### mqtt.Client#reconnecting

Boolean：正在连接服务端为 `true`，否则为 `false`

-------------------------------------------------------
<a name="store"></a>
### mqtt.Store(options)

消息存储区的内存实现

* `options` 存储选项
  * `clean`: 为 `true` 时，若 close 被调用，清除发送中的消息（默认 `true`）

`mqtt.Store` 的其他实现：

* [mqtt-level-store](http://npm.im/mqtt-level-store) which uses [Level-browserify](http://npm.im/level-browserify) to store the inflight data, making it usable both in Node and the Browser.
* [mqtt-nedb-store](https://github.com/behrad/mqtt-nedb-store) which uses [nedb](https://www.npmjs.com/package/nedb) to store the inflight data.
* [mqtt-localforage-store](http://npm.im/mqtt-localforage-store) which uses [localForage](http://npm.im/localforage) to store the inflight data, making it usable in the Browser without browserify.

-------------------------------------------------------
<a name="put"></a>
### mqtt.Store#put(packet, callback)

将一条消息存入 store中，有 `messageId` 就可以
回调函数在有消息存入后被调用

-------------------------------------------------------
<a name="createStream"></a>
### mqtt.Store#createStream()

Creates a stream with all the packets in the store.

-------------------------------------------------------
<a name="del"></a>
### mqtt.Store#del(packet, cb)

Removes a packet from the store, a packet is
anything that has a `messageId` property.
The callback is called when the packet has been removed.

-------------------------------------------------------
<a name="close"></a>
### mqtt.Store#close(cb)

Closes the Store.

<a name="browser"></a>
## Browser

<a name="cdn"></a>
### Via CDN

The MQTT.js bundle is available through http://unpkg.com, specifically
at https://unpkg.com/mqtt/dist/mqtt.min.js.
See http://unpkg.com for the full documentation on version ranges.

<a name="weapp"></a>
## WeChat Mini Program
Support [WeChat Mini Program](https://mp.weixin.qq.com/). See [Doc](https://mp.weixin.qq.com/debug/wxadoc/dev/api/network-socket.html).
<a name="example"></a>

## Example(js)

```js
var mqtt = require('mqtt')
var client = mqtt.connect('wxs://test.mosquitto.org')
```

## Example(ts)

```ts
import { connect } from 'mqtt';
const client = connect('wxs://test.mosquitto.org');
```

## Ali Mini Program
Surport [Ali Mini Program](https://open.alipay.com/channel/miniIndex.htm). See [Doc](https://docs.alipay.com/mini/developer/getting-started).
<a name="example"></a>

## Example(js)

```js
var mqtt = require('mqtt')
var client = mqtt.connect('alis://test.mosquitto.org')
```

## Example(ts)

```ts
import { connect } from 'mqtt';
const client  = connect('alis://test.mosquitto.org');
```

<a name="browserify"></a>
### Browserify

In order to use MQTT.js as a browserify module you can either require it in your browserify bundles or build it as a stand alone module. The exported module is AMD/CommonJs compatible and it will add an object in the global space.

```javascript
npm install -g browserify // install browserify
cd node_modules/mqtt
npm install . // install dev dependencies
browserify mqtt.js -s mqtt > browserMqtt.js // require mqtt in your client-side app
```

<a name="webpack"></a>
### Webpack

Just like browserify, export MQTT.js as library. The exported module would be `var mqtt = xxx` and it will add an object in the global space. You could also export module in other [formats (AMD/CommonJS/others)](http://webpack.github.io/docs/configuration.html#output-librarytarget) by setting **output.libraryTarget** in webpack configuration.

```javascript
npm install -g webpack // install webpack

cd node_modules/mqtt
npm install . // install dev dependencies
webpack mqtt.js ./browserMqtt.js --output-library mqtt
```

you can then use mqtt.js in the browser with the same api than node's one.

```html
<html>
<head>
  <title>test Ws mqtt.js</title>
</head>
<body>
<script src="./browserMqtt.js"></script>
<script>
  var client = mqtt.connect() // you add a ws:// url here
  client.subscribe("mqtt/demo")

  client.on("message", function (topic, payload) {
    alert([topic, payload].join(": "))
    client.end()
  })

  client.publish("mqtt/demo", "hello world!")
</script>
</body>
</html>
```

你的服务端（broker）应当对 websocket 连接进行相应处理 (参考 [MQTT over Websockets](https://github.com/mcollina/mosca/wiki/MQTT-over-Websockets) 以启动 [Mosca](http://mcollina.github.io/mosca/)).

<a name="qos"></a>
## 关于 QoS

QoS 的工作机制：

* QoS 0 : 接收 **至多一次** : 对客户端是否受到不作验证，只发送一次。
* QoS 1 : 接收 **至少一次** : 客户端若未收到服务端的确认，消息被发送后还会被存储下来。MQTT 保证消息会被收到，但肯能会重复。
* QoS 2 : 接收 **刚好一次** : 和 QoS 1 一致，但不会重复。

如果你比较在计算消耗，明显地 QoS 2 > QoS 1 > QoS 0

<a name="typescript"></a>
## Usage with TypeScript
This repo bundles TypeScript definition files for use in TypeScript projects and to support tools that can read `.d.ts` files.

### Pre-requisites
Before you can begin using these TypeScript definitions with your project, you need to make sure your project meets a few of these requirements:
 * TypeScript >= 2.1
 * Set tsconfig.json: `{"compilerOptions" : {"moduleResolution" : "node"}, ...}`
 * Includes the TypeScript definitions for node. You can use npm to install this by typing the following into a terminal window:
   `npm install --save-dev @types/node`

<a name="contributing"></a>
## Contributing

MQTT.js is an **OPEN Open Source Project**. This means that:

> Individuals making significant and valuable contributions are given commit-access to the project to contribute as they see fit. This project is more like an open wiki than a standard guarded open source project.

See the [CONTRIBUTING.md](https://github.com/mqttjs/MQTT.js/blob/master/CONTRIBUTING.md) file for more details.

### Contributors

MQTT.js is only possible due to the excellent work of the following contributors:

<table><tbody>
<tr><th align="left">Adam Rudd</th><td><a href="https://github.com/adamvr">GitHub/adamvr</a></td><td><a href="http://twitter.com/adam_vr">Twitter/@adam_vr</a></td></tr>
<tr><th align="left">Matteo Collina</th><td><a href="https://github.com/mcollina">GitHub/mcollina</a></td><td><a href="http://twitter.com/matteocollina">Twitter/@matteocollina</a></td></tr>
<tr><th align="left">Maxime Agor</th><td><a href="https://github.com/4rzael">GitHub/4rzael</a></td><td><a href="http://twitter.com/4rzael">Twitter/@4rzael</a></td></tr>
<tr><th align="left">Siarhei Buntsevich</th><td><a href="https://github.com/scarry1992">GitHub/scarry1992</a></td></tr>
</tbody></table>

<a name="license"></a>
## License

MIT

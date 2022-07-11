[TOC]

# 修订记录
|文档版本|API 版本|发布日期|修订说明|修订人|
|-------|----|--------------|--------|--------|
|v0.1|v1.0|2022/03/21|初始创建本文档|丁春|



# 1. 快速开始

交易所为API用户提供了两种接口模式:

* REST模式: 采用同步调用的方式(请求/应答), 完成订单创建、 撤销订单、 查询成交/订单/资产、划转等功能.
* WebSocket模式: 采用异步的方式(订阅/推送), 完成订单、交易、行情等信息的订阅推送.
  为了方便用户快速理解, 以下列出接入使用API编写交易程序的常用步骤:

## 1.1 准备工作

### 1.1.1 API-KEY的管理

* 用户需要登录交易所网站, 申请API-KEY，创建API密钥时，请确保记住以下信息：

  * 访问密钥：API访问密钥

  * 密钥：用于签名身份验证加密的密钥（仅对应用程序可见）

* 用户需要为API-KEY分配权限, 权限分为以下四种：

  * **READ** 权限允许用户调用查询交易/订单/资产/系统信息等接口,

  * **TRADE** 权限允许用户调用下单/撤单,

  * **TRANSFER** 权限允许用户调用划转接口, 只允许同一交易主账户下的子账户之间进行划转.

    **注意：** 接口文档中每个接口都有标注需要的权限, 如果没有标注则不做权限控制.

* 用户可以为API-KEY设定IP白名单列表, 如果给API-KEY设置了白名单, 只有在白名单列表中的IP地址才可以使用API-KEY发起指令调用，每个API-KEY最多可以绑定5个IP, 如果用户有多个API-KEY，需要分别为其绑定IP白名单

* 用户需要通过API-KEY在交易时进行身份认证, REST和WebSocket两种模式都需要, API-KEY的签名算法见后续章节

### 1.1.2 接入准备

在进行交易之前, 需要查询服务端信息, 以确保程序处于正确的状态:

* 用户需要通过查询服务器时间，确保本地与服务端的时间保持一致性
* 用户需要通过查询版本号, 确保每个请求头中都带上版本号, 版本号不对时, 请求可能会被拒绝

## 1.2 签名认证

### 1.2.1 签名机制

在通过互联网传输的过程中，API请求可能会被篡改。为确保请求信息的可靠性，对于除公共接口（基本信息、市场数据）之外的所有私有接口都需要使用API密钥进行数据签名和验证，以识别参数或配置在传输过程中是否发生了更改。

每个API密钥在创建时应当设置一定的权限，用于区分对不同接口的操作权限。

对API接入端的所有HTTP请求都需要身份验证和鉴权。用户可以通过创建API密钥来获取x-access-key和x-access-secret。x-access-key用于身份标识，x-access-secret用于数据签名。应将以下标头添加到所有HTTP请求中：

| Key                | Value | Description                                      |
|--------------------| -- | ------------------------------------------------ |
| x-access-key       | < API-KEY > | x-access-key 作为API接入时的身份标识               |
| x-access-sign      | < signatureOfRequest > | 基于请求内容的 Hash 值得出的数字签名或消息验证码 |
| x-access-timestamp | < timeOfRequest > | 请求发起时的时间戳(毫秒级)                       |
| x-access-version   | < versionOfRequest > | 签名协议的版本号，                  |
| Content-Type       | application/json     | 内容类型                    |

### 1.2.2 签名流程

**Example:**   
* 参与签名的字段包含header中的x-access-key、x-access-timestamp、x-access-version以及body中的所有字段(对于GET和DELETE方法，包含所有params), 按照字典序序列化为JsonString进行签名.

```java
/*
  *Signature description：required fields for Signature include apiKey, apiSecret, timestamp, version, interface fields of each request.
  *field name and field value are in the form of key-value, and the Key is sorted by ASCII code.
  *The Following is test data demo:
*/
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;
import java.util.TreeMap;
import com.alibaba.fastjson.JSONObject;
import org.apache.commons.codec.binary.Hex;

public class SignDemo {

    public static void main(String[] args) throws Exception {
        String key="84dd8e670471a888e3a7547e120886cb";
        String secret = "3211b4306fcb1d9670cbee5abc69dead";
        String timeOfRequest ="1478692862000";

        TreeMap<String,String> treeMap = new TreeMap<>();

        treeMap.put("x-access-key",key); //api-key
        treeMap.put("x-access-timestamp",timeOfRequest); //current timestamp
        treeMap.put("x-access-version","v1.0");

        treeMap.put("feild1","1");//request field mock1
        treeMap.put("feild2","2");//request field mock2
        treeMap.put("feild3","3");//request field mock3

        String requestText = JSONObject.toJSONString(treeMap);
        //Sort by ASCII of Map key {"feild1":"1","feild2":"2","feild3":"3","x-access-key":"84dd8e670471a888e3a7547e120886cb","x-access-timestamp":"1478692862000","x-access-version":"1"}

        System.out.println(createSignature(requestText,secret));
        //cVfsAceIN7w+3vDf4WEIpA+iWJnK2TjKcmwgVARi8DI=
    }

    private static String createSignature(String requestText,String secret)
            throws Exception {
        return Base64.getEncoder().encodeToString(hmacSha1(requestText, secret));
    }

    private static byte[] hmacSha1(String plainData, String secret)
            throws Exception{
        Mac mac = Mac.getInstance("HmacSHA256");
        SecretKeySpec secretKey =
                new SecretKeySpec(Hex.decodeHex(secret.toCharArray()), "HmacSHA256");
        mac.init(secretKey);
        return mac.doFinal(plainData.getBytes());
    }
}
```

## 1.2 环境相关

### 1.2.1 测试环境

Restful URL: <https://api-test.pro.hashkey.com>

WebSocket: wss://api-test.pro.hashkey.com

### 1.2.2 生产环境

**注意：** 暂未对外开放

Restful URL: <https://api.pro.hashkey.com>

WebSocket: wss://api.pro.hashkey.com

# 2. REST API

## 2.1 接入控制

### 2.1.1 REST API 流控
* 除非另有规定，否则每个API密钥的速率限制为每秒10个请求。例如，订单相关端点的速率限制为每秒10个请求。
* 除非另有规定，否则每个IP的速率限制为每秒10个请求。

### 2.1.2 请求格式

所有API请求都在RESTful框架下。参数可以由请求主体以JSON格式设置和发送。

### 2.1.3 响应格式

所有响应信息都采用JSON标准格式。包含三个字段，即“错误代码”、“错误描述”和“数据”。这些特定的业务数据包含在“数据”字段中。

| **PARAMETER** | **TYPE** | **DESCRIPTION**   |
| ------------- | -------- | ----------------- |
| error_code    | string   | API return status |
| error_message | string   | API error message |
| data          | object   | API return data   |

## 2.2 基本功能

### 2.2.1 获取当前业务时间

**Http Request:** GET info/time

**Request Content：** null

**Response Content：**

| **PARAMETER** | **TYPE** | **DESCRIPTION**        |
| ------------- | -------- | ---------------------- |
| timestamp     | int64    | millisecond time-stamp |

**Response Example：**

```json
{
    "error_code":"0000",            // 错误码
    "error_message":"",             // 错误描述
    "data":{
        "timestamp": 1478692862000  // 服务端时间戳
    }
}
```

### 2.2.2 获取API版本

**Http Request:** GET info/version

**Request Content：** null

**Response Content：**

| **PARAMETER** | **TYPE** | **DESCRIPTION** |
| ------------- | -------- | --------------- |
| version       | string   | Version No.     |

**Response Example：**

```json
{
    "error_code":"0000",        // 错误码
    "error_message":"",         // 错误描述
    "data":{
        "version":"v1.0"        // 服务端版本号
    }
}
```


## 2.3 交易相关功能

### 2.3.1 提交订单（需要“TRADE” 权限）

**Http Request:** POST /orders

**Request Content：**

| **PARAMETER**   | **TYPE** | **REQUIRED** | **DESCRIPTION**                                                                                              |
| --------------- | -------- |--------------|--------------------------------------------------------------------------------------------------------------|
| type            | string   | true         | "limit": limit order; "market": market order; "stopLimit": stop limit order; "stopMarket": stop market order |
| client_order_id | string   | true         | Max length: 20. Must be unique                                                                               |
| instrument_id   | string   | true         | e.g. "ETH-BTC"                                                                                               |
| direction       | string   | true         | "buy" or "sell"                                                                                              |
| stop_price      | string   | false        | Required when order type is stopLimit or stopMarket                                                          |
| price           | string   | false        | Limit Price. Required when order type is limit or stopLimit                                                  |
| volume          | string   | true         | Total Volume                                                                                                 |
| post_only       | bool     | false        | Only maker  default false                                                                                    |
| time_in_force (currently unused) | string | false        | default: limit and stopLimit: GTC, market and stopMarket: IOC                                                |

 **Response Content：**

| **PARAMETER**   | **TYPE** | **DESCRIPTION**                                                                                              |
| --------------- | -------- |--------------------------------------------------------------------------------------------------------------|
| type            | string   | "limit": limit order; "market": market order; "stopLimit": stop limit order; "stopMarket": stop market order |
| sys_order_id    | string   | Server order id.                                                                                             |
| client_order_id | string   | Client order id.                                                                                             |
| instrument_id   | string   | e.g. "ETH-BTC"                                                                                               |
| direction       | string   | "buy" or "sell"                                                                                              |
| stop_price      | string   | Required when order type is stopLimit or stopMarket                                                          |
| price           | string   | Limit Price. Required when order type is limit or stopLimit                                                  |
| volume          | string   | Total Volume                                                                                                 |
| post_only       | bool     | Only maker                                                                                                   |
| timestamp       | int64    | millisecond time-stamp                                                                                       |
| time_in_force (currently unused) | string   | default: limit and stopLimit: GTC, market and stopMarket: IOC                                                |

**Request Example：**

```json
{
    "type": "limit",                    // 订单类型:
                                        //    limit: 限价单;  market: 市价单;
                                        //    stopLimit: 限价止盈止损单;  stopMarket: 市价止盈止损单;
    "client_order_id":"000000001",      // 客户订单号
    "instrument_id":"ETH-BTC",          // 合约ID
    "direction":"buy",                  // 买卖方向, buy:买, sell:卖
    "price":"1",                        // 限价
    "volume":"1",                       // 数量
    "post_only": true                   // 是否只做maker
}
```

**Response Example：**

```json
{
    "error_code":"0000",    // 错误码
    "error_message":"",     // 错误描述
    "data":{
        "type":"limit",                // 订单类型
        "sys_order_id":"1550849345000001",     // 系统订单号
        "client_order_id":"000000001",  // 客户订单号
        "instrument_id":"ETH-BTC",      // 合约编号
        "direction":"buy",              // 买卖方向
        "stop_price": "0",              // 触发价格
        "price":"1",                    // 限价
        "volume":"1",                   // 数量
        "post_only": true,              // 是否只做maker
        "timestamp": 1478692862000      // 交易时间
    }
}
```

### 2.3.2 撤销单个订单（需要“TRADE” 权限）

**Http Request:** DELETE /order/

**Query Parameters：**

| **PARAMETER**   | **TYPE**   | **REQUIRED** | **DESCRIPTION**                                         |
|-----------------|------------|--------------|---------------------------------------------------------|
| sys_order_id    | string     | false        | This filed is required when client_order_id is null     |
| client_order_id | string     | false        | This filed is required when sys_order_id is null        |
| volume          | string     | false        | If it is not input, the whole order will be cancelled   |
注意：如果sys_order_id和client_order_id都存在，则忽略client_order_id

**Response Content：**
null

**Request example：**

```context
 DELETE "https://domain/order?&sys_order_id=1550849345000001"
```

**Response Example：**

```json
{
  "error_code":"0000",    // 错误码
  "error_message":""     // 错误描述
}
```


### 2.3.3 获取订单详情（需要“READ” 权限）

**Http Request:**  GET /orders

**Query Parameters：**

| **PARAMETER**   | **TYPE**     | **REQUIRED** | **DESCRIPTION**                         |
|-----------------|--------------|--------------| --------------------------------------- |
| sys_order_id    | string       | false        | Server Order ID                         |
| instrument_id   | string       | false        | e.g. "ETH-BTC"                          |
| sorting         | string       | false        | "desc" or "asc" default "asc"                      |
| direction       | string       | false        | "buy" or "sell"                         |
| type            | string       | false        | Order type                              |
| status          | array string | false        | Array with order statuses to filter by. |
| start_timestamp | string       | true         | millisecond time-stamp                  |
| end_timestamp   | string       | true         | millisecond time-stamp                  |
| limit           | string       | true        | Limit on number of results to return.  min 1 max 200 |
| page            | string       | true         | Used for pagination. Page number.       |

**Response Content：**

| **PARAMETER**    | **TYPE** | **DESCRIPTION**                                                                                              |
|------------------|----------|--------------------------------------------------------------------------------------------------------------|
| type             | string   | "limit": limit order; "market": market order; "stopLimit": stop limit order; "stopMarket": stop market order |
| sys_order_id     | string   | Server Order ID                                                                                              |
| client_order_id  | string   | Client order id.                                                                                             |
| instrument_id    | string   | e.g. "ETH-BTC"                                                                                               |
| direction        | string   | "buy" or "sell"                                                                                              |
| stop_price       | string   | Required when order type is stopLimit or stopMarket                                                          |
| price            | string   | Limit Price. Required when order type is limit or stopLimit                                                  |
| volume           | string   | Original Total Volume                                                                                        |
| status           | string   | Order status                                                                                                 |
| post_only        | bool     | Only maker                                                                                                   |
| timestamp        | int64    | millisecond time-stamp                                                                                       |
| filled_size      | string   | The size that has been filled                                                                                |
| unfilled_size    | string   | The size that has not been filled                                                                            |
| avg_filled_price | string   | Average filled price                                                                                         |
| sum_trade_amount | string   | cumulative trading amount(turnover)                                                                          |
**Order status**

| order status       | description        |
|--------------------|--------------------|
| NEW                | 订单被交易引擎接受          |
| FILLED             | 订单完全成交             |
| PARTIALLY_FILLED   | 部分订单被成交，剩余部分仍在订单簿中 |
| CANCELED           | 订单已被客户撤销，没有成交      |
| PARTIALLY_CANCELED | 订单部分成交, 剩余部分被客户撤销  |
| NOT_TRIGGERED      | 止损单未触发             |
| REJECTED           | 订单已被拒绝(目前未启用)      |

**Request Example：**

```context
 GET "http://domain/orders?sys_order_id=1550849345000001&instrument_id=ETH-BTC&start_timestamp=1656928657000&end_timestamp=1656928717000&limit=50&page=1"
```

**Response Example：**

```json
{
    "error_code":"0000",    // 错误码
    "error_message":"",     // 错误描述
    "data":[{
        "type": "limit",                // 交易类型
        "sys_order_id":"1550849345000001",     // 系统订单号
        "client_order_id":"000000001",  // 客户订单号
        "instrument_id":"ETH-BTC",      // 合约编号
        "direction":"buy",              // 买卖方向
        "stop_price":"0",               // 止损触发价格
        "price":"1",                    // 限价
        "volume":"1",                   // 原始数量
        "status": "FILLED",             // 订单状态
        "post_only": "false",           // 是否仅作为maker
        "timestamp": 1478692862000,     // 交易时间
        "filled_size": "1",             // 已成交數量
        "avg_filled_price": "1",        // 成交均价
        "unfilled_size": "0",           // 未成交数量
        "sum_trade_amount": "1"         // 累加成交额
    }]
}
```

### 2.3.4 检索交易数据（需要“READ” 权限）

**Http Request:** GET /trades

**Query Parameters** **:**

| **PARAMETER**   | **TYPE** | **REQUIRED** | **DESCRIPTION**                       |
|-----------------| -------- |--------------| ------------------------------------- |
| instrument_id   | string   | false        | e.g. "ETH-BTC"                        |
| sys_order_id    | string   | false        | Server Order ID                       |
| direction       | string   | false        | "buy" or "sell"                       |
| sorting         | string   | false        | "desc" or "asc" default "asc"                       |
| limit           | string    | true        | Limit on number of results to return.min 1 max 200.|
| page            | string    | true         | Used for pagination. Page number.     |
| start_timestamp | string    | true         | millisecond time-stamp                |
| end_timestamp   | string    | true         | millisecond time-stamp                |

 **Response Content: **

| **PARAMETER** | **TYPE** | **DESCRIPTION**        |
| ------------- | -------- |------------------------|
| trade_id      | string   | Trade ID               |
| sys_order_id  | string   | Order ID               |
| instrument_id | string   | e.g.  "ETH-BTC"        |
| direction     | string   | "buy" or "sell"        |
| price         | string   | Price                  |
| volume        | string   | Volume                 |
| fee           | string   | Fee                    |
| fee_ccy       | string   | Fee Currency           |
| timestamp     | int64    | millisecond time-stamp |
| trade_type    | string   | "Common", "Invalid"    |

**Trade Type**

| Trade type         | description |
|--------------------|-------------|
| Common             | 普通成交        |
| Invalid            | 无效成交        |

**Request Example:**

```context
 GET "http://domain/trades?sys_order_id=1550849345000001&instrument_id=ETH-BTC&start_timestamp=1656928657000&end_timestamp=1656928717000&limit=50&page=1"
```

**Response Example：**

```json
{
    "error_code":"0000",    // 错误码
    "error_message":"",     // 错误描述
    "data":[{
        "sys_order_id":"1550849345000001",     // 系统订单号
        "trade_id":"1",                 // 成交编号
        "instrument_id":"ETH-BTC",      // 合约编号
        "direction":"buy",              // 买卖方向
        "price":"1",                    // 价格
        "volume":"1",                   // 数量
        "fee":"0.05",                   // 手续费
        "fee_ccy": "BTC",               // 手续费币种
        "timestamp": 1478692862000,     // 交易时间
        "trade_type": "Common"          // 交易类型
    }]
}
```

## 2.4 资产相关功能

### 2.4.1 查询资产信息（需要“READ” 权限）

**Http Request:** GET /assets

**Request Content：** null

**Response Content：**

| **PARAMETER** | **TYPE** | **DESCRIPTION**        |
| ------------- | -------- | ---------------------- |
| asset         | string   | Asset type, e.g. "BTC" |
| free          | string   | Avalible balance       |
| freeze        | string   | Frozen balance         |

**Response Example：**

```json
{
    "error_code":"0000",
    "error_message":"",
    "data":[{
        "asset":"ETH",
        "free": "1",
        "freeze": "0",
    }, {
        // other asset
    }]
}
```

### 2.4.2 主/子交易账户之间的划转（需要“TRANSFER” 权限）

**Http Request:** POST /assets/transfer

**Request Content：**

| **PARAMETER**   | **TYPE** | **REQUIRED** | **DESCRIPTION** |
|-----------------| -------- | ------------ |-----------------|
| asset           | string   | true         | Asset ID        |
| amount          | string   | true         | Amount          |
| from_account_id | string   | true         | From Account ID |
| to_account_id   | string   | true         | To Account ID   |

**Response Content：**

| **PARAMETER** | **TYPE** | **REQUIRED** | **DESCRIPTION** |
| ------------- | -------- | ------------ | --------------- |
| error_code    | string   | true         | Error Code      |
| error_message | string   | true         | Error Message   |
| data          | string   | true         | Data            |

**Request Example：**

```json
{
    "assetId":"ETH",
    "amount":"1",
    "from_account_id": "B000000000001",
    "to_account_id":"B000000000002"
}
```

**Response Example：**

```json
{
    "error_code":"0000",
    "error_message":"",
    "data":{}
}

```

## 2.5 行情相关功能

### 2.5.1 获取K线数据

**Http Request:** GET market/kline

**Query Parameters：**

| **PARAMETER**   | **TYPE**  | **REQUIRED** | **DESCRIPTION**                                                                                                                                                     |
| --------------- |-----------|--------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| instrument_id   | string    | true         | e.g. "ETH-BTC"                                                                                                                                                      |
| period          | string    | true         | m -> minutes; h -> hours; d -> days; w -> weeks; M -> months;<br/> "1m", "3m", "5m", "15m", "30m", "1h", "2h", "4h",<br/> "6h", "8h", "12h", "1d", "3d", "1w", "1M" |
| start_timestamp | string    | true         | millisecond time-stamp  start from 000 milliseconds of this period                                                                                                  |
| end_timestamp   | string    | true         | millisecond time-stamp  end at 000 milliseconds of the next period                                                                                                  |
| page            | string     | true         | Used for pagination. Page number.                   |
| limit           | string     | true         | min 1 max 200                                                                                                                                                       |

**Response Content：**

| **PARAMETER**   | **TYPE** | **DESCRIPTION**        |
|-----------------| -------- | ---------------------- |
| instrument_id   | string   | e.g. "ETH-BTC"               |
| open            | string   | Open Price                   |
| close           | string   | Close Price                  |
| high            | string   | High Price                   |
| low             | string   | Low Price                    |
| volume          | string   | Volume in base asset, e.g, ETH in ETH-BTC |
| start_timestamp | int64    | Start millisecond time-stamp |
| end_timestamp   | int64    | End millisecond time-stamp   |

**Response Example：**

```json
{
    "error_code":"0000",    // 错误码
    "error_message":"",     // 错误描述
    "data":[
        {
            "instrument_id":"ETH-BTC",      // 合约编号
            "open":"10",                    // 开始价格
            "close":"10",                   // 结束价格
            "high":"10",                    // 最高价格
            "low":"10",                     // 最低价格
            "volume":"100",                 // 数量
            "start_timestamp":1646213700000,     // 开始时间
            "end_timestamp":1646213760000        // 结束时间
        }
    ]
}
```

### 2.5.2 获取全市场成交记录

**Http Request:** GET /trades/market

**Query Parameters：**

| **PARAMETER**    | **TYPE** | **REQUIRED** | **DESCRIPTION**                   |
|------------------| -------- |--------------|-----------------------------------|
| instrument_id    | string   | false        | e.g. "ETH-BTC"                    |
| start_timestamp  | string    | true         | millisecond time-stamp            |
| end_timestamp    | string    | true         | millisecond time-stamp            |
| limit            | string    | true        | min 1 max 200           |
| page             | string    | true         | Used for pagination. Page number. |


**Response Content：**

| **PARAMETER** | **TYPE** | **DESCRIPTION**  |
| ------------- | -------- | ---------------- |
| instrument_id | string   | e.g. "ETH-BTC"   |
| trade_id      | string   | Trade ID         |
| price         | string   | Price            |
| volume        | string   | Volume           |
| direction     | string   | "buy" or "sell"  |
| timestamp     | int64    | millisecond time-stamp |

**Response Example：**

```json
{
    "error_code":"0000",    // 错误码
    "error_message":"",     // 错误描述
    "data":[
        {
            "instrument_id": "ETH-BTC",     // 合约编号
            "trade_id": "123456789",        // 成交编号
            "price": "10",                  // 价格
            "volume": "100",                // 数量
            "direction": "buy",             // 买卖方向
            "timestamp": 1478692862000      // 时间
        }
    ]
}
```

# 3. Websocket API

## 3.1 接入控制

### 3.1.1 WebSocket API 会话限制
* 一个IP地址最多可以同时建立5个WebSocket会话，使用APIKey可以为WebSocket会话授权，每个APIKey可以同时为10个会话授权
* 未授权的会话可以订阅行情数据等公有数据，已授权的会话可以订阅私有数据


**URL for Access:**

 /stream

### 3.1.2 心跳消息

当用户的Websocket客户端应用程序与HashKey Websocket服务器连接时，服务器将定期（当前设置为每10秒）发送一条包含以下整数值的ping消息。

```json
 {"type":"ping","data":"1492420473027"}
```

当用户从Websocket客户端应用程序接收到上述消息时，应该返回一条包含相同整数值的pong消息。

```json
{"type":"pong","data":"1492420473027"}
```

**注意:** 当Websocket服务器连续发送两条ping消息，但没有收到pong消息时，服务器将终止与客户端应用程序的连接。

### 3.1.3 请求格式

所有订阅请求主体都应采用有效的JSON格式。每个主题都有自己的参数集。有关详细信息，请参考接口请求示例。

### 3.1.4 响应格式

所有响应主体都应采用有效的JSON格式。有关详细信息，请参考接口响应示例。

### 3.1.5 认证

**Request Content:**

| **PARAMETER**      | **TYPE** | **REQUIRED** | **DESCRIPTION**                                             |
| ------------------ |----------| ------------ | ----------------------------------------------------------- |
| type               | string   | true         | Default: auth                                               |
| x-access-key       | string   | true         | The API Access Key you applied for.                         |
| x-access-sign      | string   | true         | A value calculated by the hash value from request to ensure it is valid and not tampered.|
| x-access-timestamp | string    | true         | The timestamp represents the time of request (in milliseconds). |
| x-access-version   | string   | true         | Signature protocol version. Default version is 1.           |

**Request Example：**

 ```json
{
    "type":"auth",
    "auth":{
        "x-access-key":"xxxxxxxxx",
        "x-access-sign":"xxxxxxxxx",
        "x-access-timestamp": "1478692862000",
        "x-access-version":"v1.0"
    },
    "id": 1
}
 ```

**Response Example：**

```json
{
    "id": 1,
    "result": true
}
```

### 3.1.6 订阅

**Request Content:**

| **PARAMETER** | **TYPE**     | **REQUIRED** | **DESCRIPTION**      |
| ------------- |--------------| ------------ | -------------------- |
| type          | string       | true         | "sub"                |
| id            | int64        | true         | Unique request id    |
| Parameters    | object array | true         | Subscribe Parameters |
| => topic      | string       | true         | Topic                |

**Response Content:**

| **PARAMETER** | **TYPE**     | **DESCRIPTION**                                |
| ------------- |--------------|------------------------------------------------|
| id            | int64        | Unique request id                              |
| result        | object array | Topic list                                     |
| error_code    | string       | 0000: success 0100: partial success 0001: fail |

**Request Example:**

```json
{
    "type": "sub",            // 消息类型, sub:订阅
    "parameters":[{
        "topic":"order_rtn",            // 订阅主题
        "instrument_id":"ETH-USDT"      // 合约ID
    }],
    "id": 1                             // 消息ID
}
```

**Response Example:**

```json
{
    "id": 1,                          // 消息ID
    "result": null,                   // 当订阅成功时返回null, 其他情况下返回已成功订阅的主题列表
    "error_code": "0000"              // 当错误码为0000时表示所有主题订阅成功, 其他情况下返回错误的应答码表示没有全部订阅成功或全部订阅失败
}
```

### 3.1.7 取消订阅

**Request Content:**

| **PARAMETER** | **TYPE**     | **REQUIRED** | **DESCRIPTION**    |
| ------------- |--------------| ------------ | ------------------ |
| type          | string       | true         | "unsub"            |
| id            | int64        | true         | Unique request id  |
| Parameters    | object array | true         | Unsubscribe parmas |
| => topic      | string       | true         | Topic              |
| error_code    | string       | true         | Error code         |
| error_message | string       | true         | Explain error info |

**Response Content:**

| **PARAMETER** | **TYPE**     | **DESCRIPTION**                                |
| ------------- |--------------|------------------------------------------------|
| id            | int64        | Unique request id                              |
| result        | object array | Topic list                                     |
| error_code    | string       | 0000: success 0100: partial success 0001: fail |

**Request Example:**

 ```json
{
    "type":"unsub",
    "parameters":  [{
        "topic":"xxxxxxxxx",
        ......
    }],
    "id": 2
}
 ```

**Response Example:**

```json
{
    "id": 1,
    "result": null,
    "error_code": "0000"
}
```



## 3.2 Websocket 行情数据（公共流）

### 3.2.1 K线

**推送频率**
每2000毫秒推送一次

**Request Content:**

| **PARAMETER** | **TYPE** | **REQUIRED** | **DESCRIPTION**                                              |
| ------------- | -------- | ------------ | ------------------------------------------------------------ |
| type          | string   | true         | "sub"                                                        |
| topic         | string   | true         | "kline"                                                      |
| period        | string   | true         | m -> minutes; h -> hours; d -> days; w -> weeks; M -> months;<br/> "1m", "3m", "5m", "15m", "30m", "1h", "2h", "4h",<br/> "6h", "8h", "12h", "1d", "3d", "1w", "1M" |
| instrument_id | string   | true         | e.g. "ETH-USDT", "ETH-BTC"                                   |

**Response Content:**

| **PARAMETER** | **TYPE** | **DESCRIPTION**                               |
| ------------- | -------- | --------------------------------------------- |
| type          | string   | "sub-resp"                                    |
| topic         | string   | "kline"                                       |

**Data Content:**

| **PARAMETER**   | **TYPE** | **DESCRIPTION**                               |
|-----------------| -------- | --------------------------------------------- |
| instrument_id   | string   | e.g. "ETH-USDT", "ETH-BTC"                    |
| high            | string   | High  price                                   |
| open            | string   | Open price                                    |
| low             | string   | Low  price                                    |
| close           | string   | Close  price                                  |
| volume          | string   | Volume in base asset, e.g, ETH in ETH-BTC     |
| start_timestamp | int64    | millisecond time-stamp     e.g. 1646213700000 |
| end_timestamp   | int64    | millisecond time-stamp     e.g. 1646213800000 |

**How to Subscribe：**

```json
{
    "type":"sub",                       // 交易类型
    "parameters":[{
        "topic":"kline",                // 主题是K线
        "period":"1m",                  // 时间周期
        "instrument_id":"ETH-USDT"      // 合约编号
    }],
    "id": 1                             // 消息ID
}
```

**Response Example：**

```json
{
    "type":"sub-resp",                           // 交易类型
    "topic":"kline",                             // 主题
    "data":{
            "instrument_id":"ETH-BTC",           // 合约编号
            "open":"10",                         // 开始价格
            "close":"10",                        // 结束价格
            "high":"10",                         // 最高价格
            "low":"10",                          // 最低价格
            "volume":"100",                      // 数量
            "start_timestamp":1646213700000,     // 开始时间
            "end_timestamp":1646213760000        // 结束时间
    }
}
```

### 3.2.2 行情数据

**推送频率**
每1000毫秒推送一次

**Request Content:**

| **PARAMETER** | **TYPE** | **REQUIRED** | **DESCRIPTION**            |
| ------------- | -------- | ------------ | -------------------------- |
| type          | string   | true         | sub                        |
| topic         | string   | true         | market_data                |
| instrument_id | string   | true         | e.g. "ETH-USDT", "ETH-BTC" |

**Response Content:**

| **PARAMETER**        | **TYPE** | **DESCRIPTION**        |
| -------------------- | -------- | ---------------------- |
| type                 | string   | sub-resp               |
| topic                | string   | market_data            |

**Data Content:**

| **PARAMETER**        | **TYPE** | **DESCRIPTION**                           |
|----------------------| -------- |-------------------------------------------|
| high                 | string   | High price                                |
| low                  | string   | Low price                                 |
| open                 | string   | Open price                                |
| instrument_id        | string   | e.g. "ETH-USDT", "ETH-BTC"                |
| base                 | string   | Base asset, e.g, ETH in ETH-BTC           |
| quote                | string   | Quote asset, e.g, BTC in ETH-BTC          |
| last_price           | string   | Price of the latest trade                 |
| price_change_rate    | string   | Price change rate of 24 hours             |
| price_change         | string   | Price change  of 24 hours                 |
| volume               | string   | Volume in base asset, e.g, ETH in ETH-BTC |

**How to Subscribe：**

```json
{
    "type":"sub",                           // 交易类型
    "parameters":[{
        "topic":"market_data",              // 主题是MarketData
        "instrument_id":"ETH-USDT"          // 合约编号
    }],
    "id": 1                                 // 消息ID
}
```

**Response Example：**

```json
{
    "type":"sub-resp",                  // 交易类型: sub-resp: 订阅结果
    "topic":"market_data",              // 主题
    "data":[
        {
            "open":"0",                // 开始价格
            "high":"0",                 // 最高价格
            "low":"0",                  // 最低价格
            "base":"ETH",               // 基础资产
            "quote":"USDT",             // 计价资产
            "instrument_id":"ETH-USDT", // 合约编号
            "price_change_rate":"0",    // 24小时变化比例
            "price_change":"0",         // 24小时变化数量
            "last_price":"0",           // 最后一笔价格
            "volume":"0"                // 24小时交易数量
        }
    ]
}
```

### 3.2.3 深度行情数据

**推送频率**
每500毫秒推送一次(如果有变化)

**Request Content:**

| **PARAMETER** | **TYPE** | **REQUIRED** | **DESCRIPTION**            |
| ------------- | -------- | ------------ | -------------------------- |
| type          | string   | true         | "sub"                      |
| topic         | string   | true         | "depth_market_data"        |
| instrument_id | string   | true         | e.g. "ETH-USDT", "ETH-BTC" |

**Response Content:**

| **PARAMETER** | **TYPE** | **DESCRIPTION**             |
| ------------- | -------- |-----------------------------|
| type          | string   | "sub-resp"                  |
| topic         | string   | "depth_market_data"         |
| instrument_id | string   | e.g. "ETH-USDT", "ETH-BTC"  |

**Data Content:**

| **PARAMETER** | **TYPE** | **DESCRIPTION** |
|---------------|----------|-----------------|
| volume        | string   | Volume          |
| price         | string   | Price           |
| timestamp     | int64    | Timestamp       |

**How to Subscribe：**

```json
{
    "type":"sub",                           // 交易类型
    "parameters":[{
        "topic":"depth_market_data",        // 主题是depth_market_data
        "instrument_id":"ETH-USDT"          // 合约编号
    }],
    "id": 1                                 // 消息ID
}
```

**Response Example：**

```json
{
    "type":"sub-resp",                      // 交易类型: sub-resp: 订阅结果
    "topic":"depth_market_data",            // 主题
    "instrument_id":"ETH-USDT",
    "data":{
        "asks":[                            // 卖50档, 按价格从小到大排序
            {
                "volume":"3",               // 数量
                "price":"1.7"               // 价格
            },
            {
                "volume":"3",
                "price":"2"
            }
        ],
        "bids": [{                          // 买50档， 按价格从大到小排序
                "volume":"3",
                "price":"1.5"
        }]
    },
  "timestamp": 1646213700000
}
```

### 3.2.4 全市场成交数据

**Request Content:**

| **PARAMETER** | **TYPE** | **REQUIRED** | **DESCRIPTION**            |
| ------------- | -------- | ------------ | -------------------------- |
| type          | string   | true         | "sub"                      |
| topic         | string   | true         | "trade_rtn_all"            |
| instrument_id | string   | false        | e.g. "ETH-USDT", "ETH-BTC" |

**Response Content:**

| **PARAMETER** | **TYPE** | **DESCRIPTION**  |
| ------------- | -------- | ---------------- |
| type          | string   | "sub-resp"               |
| topic         | string   | Topic                    |

**Data Content:**

| **PARAMETER** | **TYPE** | **DESCRIPTION**  |
| ------------- | -------- | ---------------- |
| instrument_id | string   | Instrument Id            |
| direction     | string   | "buy" or "sell"          |
| trade_id      | string   | Trade Id                 |
| volume        | string   | Volume                   |
| price         | string   | Price                    |
| timestamp    | int64x   | millisecond time-stamp   |

**How to Subscribe：**

```json
{
    "type":"sub",
    "parameters":[{
        "topic":"trade_rtn_all",
        "instrument_id":"ETH-USDT"
    }],
    "id": 1
}
```

**Request Response：**

```json
{
    "type":"sub-resp",
    "topic":"trade_rtn_all",                    // 主题
    "data":[
            {
                "instrument_id":"ETH-BTC",      // 合约编号
                "direction":"buy",              // 买卖方向
                "trade_id":"1000001",           // 成交编号
                "volume":"2",                   // 数量
                "price":"2",                    // 价格
                "timestamp":1478692862000,      // 交易时间
                "fee":"2",                      // 手续费
                "fee_ccy": "BTC"                // 手续费币种
            },
            {
                "instrument_id":"ETH-BTC",
                "direction":"sell",
                "trade_id":"1000002",
                "volume":"2",
                "price":"2",
                "timestamp":1478692862000,
                "fee":"2",
                "fee_ccy": "BTC"
            }
        ]
}
```

## 3.3 Websocket 订单及成交数据（私有流）

### 3.3.1 订单数据

**Request Content:**

| **PARAMETER** | **TYPE** | **REQUIRED** | **DESCRIPTION**            |
| ------------- | -------- | ------------ | -------------------------- |
| type          | string   | true         | "sub"                      |
| topic         | string   | true         | "order_rtn"                |
| instrument_id | string   | false        | e.g. "ETH-USDT", "ETH-BTC" |

**Response Content:**

| **PARAMETER**   | **TYPE** | **DESCRIPTION**                                              |
| --------------- | -------- | ------------------------------------------------------------ |
| type            | string   | "sub-resp"                                                   |
| topic           | string   | "order_rtn"                                                  |

**Data Content:**

| **PARAMETER**   | **TYPE** | **DESCRIPTION**                                                                                              |
| --------------- |----------|--------------------------------------------------------------------------------------------------------------|
| type            | string   | "limit": limit order; "market": market order; "stopLimit": stop limit order; "stopMarket": stop market order |
| sys_order_id    | string   | Server Order ID                                                                                              |
| client_order_id | string   | Client order id.                                                                                             |
| instrument_id   | string   | e.g. "ETH-BTC"                                                                                               |
| direction       | string   | "buy" or "sell"                                                                                              |
| stop_price      | string   | Required when order type is stopLimit or stopMarket                                                          |
| price           | string   | Limit Price. Required when order type is limit or stopLimit                                                  |
| volume          | string   | Original Total Volume                                                                                        |
| status          | string   | Order status                                                                                                 |
| post_only       | bool     | Only maker                                                                                                   |
| timestamp       | int64    | millisecond time-stamp                                                                                       |
| filled_size     | string   | The size that has been filled                                                                                |
| unfilled_size    | string   | The size that has not been filled                                                                            |
| avg_filled_price | string   | Average filled price                                                                                         |
| sum_trade_amount | string   | cumulative trading amount(turnover)                                                                          |

**How to Subscribe：**


```json
{
    "type":"sub",
    "parameters":[{
        "topic":"order_rtn",
        "instrument_id":"ETH-USDT"
    }],
    "id": 1
}
```

**Response Example：**

```json
{
    "type":"sub-resp",
    "topic":"order_rtn",
    "data":[{
        "type": "limit",                // 交易类型
        "sys_order_id":"1550849345000001",     // 系统订单号
        "client_order_id":"000000001",  // 客户订单号
        "instrument_id":"ETH-BTC",      // 合约编号
        "direction":"buy",              // 买卖方向
        "stop_price":"0",               // 止损触发价格
        "price":"1",                    // 限价
        "volume":"1",                   // 原始数量
        "status": "FILLED",             // 订单状态
        "post_only": "false",           // 是否仅作为maker
        "timestamp": 1478692862000,     // 交易时间
        "filled_size": "1",             // 已成交數量
        "avg_filled_price": "1",        // 成交均价
        "unfilled_size": "0",           // 未成交数量
        "sum_trade_amount": "1"         // 累计成交额
    }]
}
```

### 3.3.2 成交数据

**Request Content:**

| **PARAMETER** | **TYPE** | **REQUIRED** | **DESCRIPTION**            |
| ------------- | -------- | ------------ | -------------------------- |
| type          | string   | true         | "sub"                      |
| topic         | string   | true         | "trade_rtn"                |
| instrument_id | string   | false        | e.g. "ETH-USDT", "ETH-BTC" |

**Response Content:**

| **PARAMETER**   | **TYPE** | **DESCRIPTION**                      |
| --------------- | -------- |--------------------------------------|
| type            | string   | "sub-resp"                           |
| topic           | string   | "trade_rtn"                          |
| trade_id        | string   | Trade Id                             |
| client_order_id | string   | Client Order Id                      |
| sys_order_id    | string   | Server Order Id                      |
| instrument_id   | string   | e.g. "ETH-BTC"                       |
| direction       | string   | "buy" or "sell"                      |
| price           | string   | Price                                |
| volume          | string   | Volume                               |
| fee             | string   | Fee                                  |
| fee_ccy         | string   | Transaction fee currency, e.g. "ETH" |
| timestamp      | int64    | Trade millisecond time-stamp         |
| trade_type    | string   | "Common", "Invalid"                  |


**How to Subscribe：**

```json
{
    "type":"sub",
    "parameters":[{
        "topic":"trade_rtn",
        "instrument_id":"ETH-USDT"
    }],
    "id": 1
}
```

**Response Example：**

```json
{
    "type":"sub-resp",
    "topic":"trade_rtn",
    "data":[
        {
            "sys_order_id":"1550849345000001",     // 系统订单号
            "client_order_id":"000000001",  // 客户订单号
            "trade_id":"1",                 // 成交编号
            "instrument_id":"ETH-BTC",      // 合约编号
            "direction":"buy",              // 买卖方向
            "price":"1",                    // 价格
            "volume":"1",                   // 数量
            "fee":"0.05",                   // 手续费
            "fee_ccy": "BTC",               // 手续费币种
            "timestamp": 1478692862000,     // 交易时间
            "trade_type": "Common"          // 交易类型
        },
        {
            "sys_order_id":"1550849345000002",
            "client_order_id":"000000002",
            "trade_id":"2",
            "instrument_id":"ETH-BTC",
            "direction":"sell",
            "price":"1",
            "volume":"2",
            "fee":"0.05",
            "fee_ccy": "BTC",
            "timestamp":1478692862000,
            "trade_type": "Common"
        }
    ]
}
```

# 4. 错误码

## 4.1 错误码分类标准

| **ERROR CODE RANGE** | **DESCRIPTION**    |
| -------------------- | ------------------ |
| 0000~0099            | 公共错误类         |
| 0100~0199            | WebSocket 相关错误 |

## 4.2 错误码示例

| **ERROR CODE** | **TYPE**   | **ERROR MESSAGE** |
|----------------| ---------- |-------------------|
| 0000           | 公共错误类 | 成功                |
| 0001           | 公共错误类 | 通用参数错误              |
| 0002           | 公共错误类 | 系统错误              |
| 0003           | 公共类错误 | 不允许的请求             |
| 0004           | 公共类错误 | 账户资金不足            |
| 0005           | 公共类错误 | 合约未找到             |
| 0006           | 公共类错误 | 系统订单未找到           |
| 0007           | 公共类错误 | 数量错误            |
| 0008           | 公共类错误 | 无足够数量撤销           |
| 0009           | 公共类错误 | 价格不满足精度要求         |
| 0010           | 公共类错误 | 价格错误              |
| 0011           | 公共类错误 | 不允许此用户操作     |
| 0012           | 公共类错误 | 不支持的订单类型          |
| 0013           | 公共类错误 | 重复订单              |
| 0100           | WebSocket相关错误 | 订阅失败              |
| 0101           | WebSocket相关错误 | 部分订阅成功            |


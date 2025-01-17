<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Web Socket Streams for MYX](#web-socket-streams-for-myx)
  - [Changelog](#changelog)
- [General information](#general-information)
  - [Endpoints](#endpoints)
  - [Rate Limits](#rate-limits)
  - [Message Interactions](#message-interactions)
    - [Message Format](#message-format)
    - [Subscribe to a stream](#subscribe-to-a-stream)
    - [Unsubscribe to a stream](#unsubscribe-to-a-stream)
    - [Ping/Pong](#pingpong)
    - [Sign In](#sign-in)
    - [Response codes and messages](#response-codes-and-messages)
  - [Supported Chains](#supported-chains)
- [Public Stream information](#public-stream-information)
  - [Summary Streams](#summary-streams)
  - [Ticker Streams](#ticker-streams)
  - [Candle Streams](#candle-streams)
  - [Orderbook Streams](#orderbook-streams)
  - [Trigger Streams](#trigger-streams)
  - [Trade Streams](#trade-streams)
  - [Liquidation Streams](#liquidation-streams)
- [Private Stream information](#private-stream-information)
  - [Order Streams](#order-streams)
  - [Position Streams](#position-streams)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Web Socket Streams for MYX

## Changelog

* November 12, 2024
  * Add summary streams
* September 9, 2024
  * Initial release

# General information

* All pairs for streams are **UPPERCASE**.
* Websocket server will send a `ping` event every 20 seconds.
    * If the websocket server does not receive a `pong` event back from the connection within a 60 seconds period, the connection will be disconnected.
    * When you receive a ping, you must send a pong with a copy of ping's payload as soon as possible.

## Endpoints

**Beta Env**
  
  `wss://oapi-beta.myx.finance:443/ws`

**Prod Env**
  
  `wss://oapi.myx.finance:443/ws`

## Rate Limits
* There is a limit of **60 connections per attempt every 1 minute per IP**.
* A single connection can listen to a maximum of 10 streams.
* Connections exceeding the limit will be disconnected.
* Repeatedly disconnected IPs may be disabled.

## Message Interactions

### Message Format
* Request

| field   | value                         |
|---------|-------------------------------|
| request | `sub`/`unsub`/`signin`/`pong` |
| args    | object                        |

* Response

| field  | value                  | note                                              |
|--------|------------------------|---------------------------------------------------|
| type   | `sub`/`unsub`/`signin` |                                                   |
| data   | object                 |                                                   |
| - code | integer                | response code                                     |
| - msg  | string                 | response message, **May not be returned if null** |
| - data | object                 | failed data, **May not be returned if null**      |

* Pushed messages

| field     | value             | note                                         |
|-----------|-------------------|----------------------------------------------|
| type      | `ping`/`{stream}` | {stream}: name of the stream to subscribe to |
| data      | object            |                                              |

### Subscribe to a stream
**args**: Array of stream names to subscribe to

* Request
```json5
{
  "request": "sub",
  "args": ["quote.42161.BTCUSDC", "order.42161.BTCUSDC"]
}
```

* Response(Successful)
```json5
{
  "type": "sub",
  "data": {
    "code": 9200,
    "message": null,
    "data": null
  }
}
```
* Response(Failed)
```json5
{
  "type": "sub",
  "data": {
    "code": 9901,
    "message": "Illegal Parameter",
    "data": ["quote.42161.BTCUSDC"]
  }
}
```

### Unsubscribe to a stream
**args**: Array of stream names to unsubscribe to
* Request
```json5
{
  "request": "unsub",
  "args": ["quote.42161.BTCUSDC"]
}
```
* Response

  When subscribing to multiple streams at once, the responses for each stream will be sent separately.
```json5
  {
    "type": "unsub",
    "data": {
      "code": 9200,
      "message": null,
      "data": "quote.42161.BTCUSDC"
    }
  }
```

### Ping/Pong

Ping messages are sent by the server, and when the client receives them, it needs to reply to the server with a pong message using the data of the ping.

**If the server does not receive a pong message three times in a row, it will close the connection.**

* Ping
```json5
{
  "type": "ping",
  "data": 1725844149000 // Unix timestamp(ms)
}
```
* Pong
```json5
{
  "request": "pong",
  "args": 1725844149000 // the data of the ping message
}
```

### Sign In

1. Generate a random string of 16 characters (_nonce_) and expiration time (_expires_)
   * nonce: can only use [a-zA-Z0-9] in the characters
   * expires: the unix timestamp(s)
2. Assembling a message to be signed using nonce and expires(_message_)
   * message: `Action: MYX Signature Verification\nNonce: {nonce}-{expires}`, *Note the spaces and line breaks*.
3. Sign the message with the wallet's private key(_signature_)
4. Generate a token
   * token: `ecdsa-1.{wallet address}-{nonce}-{expires}.{signature}`

e.g.
  * nonce: _S62zdaX8HbUkajsT_
  * expires: _1725844149_
  * message: _Action: MYX Signature Verification\nNonce: S62zdaX8HbUkajsT-1725844149_
  * wallet address: _0x0902Bd63695433b5303c150e7fACE75Da05A4a87_
  * signature: _0x7e85ee11ac699ff5fcf2efcec7bd84c6a591687e4517e8f1d2464029aaaec98d3599211c99edd83c4731aa8bacecc8ccb7136265f1c01ca1aad94807b12325301c_
  * token: _ecdsa-1.0x0902Bd63695433b5303c150e7fACE75Da05A4a87-S62zdaX8HbUkajsT-1725844149.0x7e85ee11ac699ff5fcf2efcec7bd84c6a591687e4517e8f1d2464029aaaec98d3599211c99edd83c4731aa8bacecc8ccb7136265f1c01ca1aad94807b12325301c_

* Request
```json5
{
  "request": "signin",
  "args": "ecdsa-1.0x0902Bd63695433b5303c150e7fACE75Da05A4a87-S62zdaX8HbUkajsT-1725844149.0x7e85ee11ac699ff5fcf2efcec7bd84c6a591687e4517e8f1d2464029aaaec98d3599211c99edd83c4731aa8bacecc8ccb7136265f1c01ca1aad94807b12325301c" // token
}
```

* Response
```json5
{
  "type": "signin",
  "data": {
    "code": 9200,
    "message": null,
    "data": null
  }
}
```

### Response codes and messages

* 9200: _No message_

  _The server successfully handled the client's request._
* 9901: **Illegal Parameter**

  _Parameter errors, including: format, range, value, etc._
* 9401: **Unauthorized**

  _Authentication failure of the user, including signature error, expiration, etc._
* 9403: **Forbidden**

  _Unprivileged operations, e.g., unauthorized access to private stream._
* 9429: **Too many requests**

* 9500: **Internal Server Error**

  _Any unknown error not caused by the client._

## Supported Chains

**Beta Env**

| Chain Name       | Chain ID |
|------------------|----------|
| Arbitrum Sepolia | 421614   |
| Linea Sepolia    | 59141    |

**Prod Env**


| Chain Name     | Chain ID |
|----------------|----------|
| Arbitrum One   | 42161    |
| Linea Mainnet  | 59144    |
| Scroll Mainnet | 534352   |

# Public Stream information

## Summary Streams

**Stream Name:** `summary.{chainId}`

**Message:**
```json5
{
  "type": "summary.42161",
  "data": {
    "E": 1725844149000,   // Event time(Unix timestamp: ms)
    "d": [{               
      "P": "BTCUSDC",     // Pair
      "C": "0.012",       // Change
      "p": "65633.66",    // Last price
      "h": "66123.45",    // High price
      "l": "65000.00",    // Low price
      "v": "1.23",        // Volume
      "T": "80811.32",    // Turnover
    }, {
      "P": "ETHUSDC",
      "C": "0.012",
      "p": "65633.66",
      "h": "66123.45",
      "l": "65000.00",
      "v": "1.23",
      "T": "80811.32",
    }]
}

```

## Ticker Streams
24hr rolling window ticker statistics for a single pair. These are NOT the statistics of the UTC day, but a 24hr rolling window for the previous 24hrs.

**Stream Name:** `ticker.{chainId}.{pair}`

**Message:**
```json5
{
  "type": "ticker.42161.BTCUSDC",
  "data": {
    "C": "0.012"        // Change
    "p": "65633.66",    // Last price
    "h": "66123.45",    // High price
    "l": "65000.00",    // Low price
    "v": "1.23",        // Volume
    "T": "80811.32",    // Turnover
    "i": "65633.60"     // Last index price
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  }
}
```

## Candle Streams

**Stream Name:** `candle.{chainId}.{pair}_{interval}`

**Interval:** `1m`/`5m`/`15m`/`30m`/`1h`/`4h`/`1d`/`1w`/`1M`
* m: minutes
* h: hours
* d: days
* w: weeks
* M: months


**Message:**
```json5
{
  "type": "candle.42161.BTCUSDC_5m",
  "data": {
    "t": 1725811200     // Candle start time(Unix timestamp: s)
    "o": "65633.66",    // Open price
    "c": "65633.66",    // Close price
    "h": "66123.45",    // High price
    "l": "65000.00",    // Low price
    "v": "1.23",        // Volume
    "T": "80811.32",    // Turnover
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  }
}
```

## Orderbook Streams

**Stream Name:** `orderbook.{chainId}.{pair}_{level}`

**Level:** `10`/`20`

**Message:**
```json5
{
  "type": "orderbook.42161.BTCUSDC_10",
  "data": {
    "a": [{
      "p": "65535.00",    // Price
      "s": "0.12",        // Size
    },{
      "p": "65535.00",
      "s": "0.12",
    }],                   // Asks
    "b": [{
      "p": "65535.00",
      "s": "0.12",
    },{
      "p": "65535.00",
      "s": "0.12",
    }],                   // Bids
    "E": 1725844149000    // Event time(Unix timestamp: ms)
  }
}
```

## Trigger Streams

**Stream Name:** `trigger.{chainId}.{pair}_{level}`

**Level:** `10`/`20`

**Message:**
```json5
{
  "type": "trigger.42161.BTCUSDC_10",
  "data": {
    "a": [{
      "p": "65535.57",    // Price
      "s": "0.12",        // Size
    },{
      "p": "65555.90",
      "s": "0.89",
    }],                   // Asks
    "b": [{
      "p": "65301.98",
      "s": "0.72",
    },{
      "p": "65202.06",
      "s": "0.81",
    }],                   // Bids
    "E": 1725844149000    // Event time(Unix timestamp: ms)
  }
}
```

## Trade Streams

**Stream Name:** `trade.{chainId}.{pair}`

**Message:**
```json5
{
  "type": "trade.42161.BTCUSDC",
  "data": {
    "S": "LONG",        // Side: LONG/SHORT
    "p": "65535.00",    // Price
    "s": "0.12",        // Size
    "t": 1725844149     // Trade time(Unix timestamp: s)
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  }
}
```

## Liquidation Streams

**Stream Name:** `liquidation.{chainId}.{pair}`

**Message:**
```json5
{
  "type": "liquidation.42161.BTCUSDC",
  "data": {
    "S": "LONG",        // Side: LONG/SHORT
    "p": "65535.00",    // Price
    "s": "0.12",        // Size
    "t": 1725844149     // Liquidation time(Unix timestamp: s)
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  }
}
```

# Private Stream information

## Order Streams

**Stream Name:** `order.{chainId}.{pair}`

_To subscribe to messages for all pairs in the chain, set pair to `*`._

**Message:**

* New order
```json5
{
  "type": "order.42161.BTCUSDC",
  "data": {
    "id": 12345678,     // Order id
    "p": "65535.00",    // Order price
    "s": "0.12",        // Order size
    "O": "INCREASE",    // Order type: INCREASE/DECREASE
    "S": "LONG",        // Order side: LONG/SHORT
    "y": "LIMIT",       // Trade type: MARKET/LIMIT/TP/SL
    "x": "NEW",         // Order status
    "t": 1725844149     // Update time(Unix timestamp: s)
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  }
}
```
* Execute order
```json5
{
  "type": "order.42161.BTCUSDC",
  "data": {
    "id": 12345678,     // Order id
    "p": "65535.00",    // Order price
    "s": "0.12",        // Order size
    "O": "INCREASE",    // Order type: INCREASE/DECREASE
    "S": "LONG",        // Order side: LONG/SHORT
    "y": "LIMIT",       // Trade type: MARKET/LIMIT/TP/SL
    "C": "0"            // Cumulative trade size
    "P": "65535.00",    // Last price
    "x": "FILLED",      // Order status: FILLED/PARTIALLY_FILLED
    "f": "0.000111",    // Trading fee
    "F": "0.000111",    // Funding fee
    "N": "0.000123",    // Network fee
    "T": "USDC",        // Token of payment of network fee
    "t": 1725844149     // Update time(Unix timestamp: s)
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  }
}
```
* Cancel order
```json5
{
  "type": "order.42161.BTCUSDC",
  "data": {
    "id": 12345678,               // Order id
    "y": "LIMIT",                 // Order type: MARKET/LIMIT/TP/SL
    "x": "CANCELED",              // Order status
    "r": "Insufficient balance",  // Reson for cancellation
    "t": 1725844149               // Update time(Unix timestamp: s)
    "E": 1725844149000            // Event time(Unix timestamp: ms)
  }
}
```

## Position Streams

**Stream Name:** `position.{chainId}.{pair}`

_To subscribe to messages for all pairs in the chain, set pair to `*`._

**Message:**
```json5
{
  "type": "position.42161.BTCUSDC",
  "data": {
    "S": "LONG",        // Side: LONG/SHORT
    "p": "65535.00",    // Average position price
    "s": "0.12",        // Postion size
    "c": "6530.1",      // Collateral for position
    "t": 1725844149     // Update time(Unix timestamp: s)
    "E": 1725844149000  // Event time(Unix timestamp: ms)
  }
}
```
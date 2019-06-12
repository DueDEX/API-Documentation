# DueDEX Data Types

These are the data types used across restful and WebSocket APIs in DueDEX.

## Entities

### Margin

| Field | Type | Remark |
| - | - | - |
| currency | string | Currency symbol |
| available | decimal | The available balance amount of the currency |
| orderMargin | decimal | The amount allocated to open orders |
| positionMargin | decimal | The amount allocated as position maintainance margin |
| realisedPnl | decimal | The total realised pnl in this currency |
| unrealisedPnl | decimal | The unrealised pnl of open positions calculated at mark prices |

### Order

| Field | Type | Remark |
| - | - | - |
| instrument | string | Id of the instrument |
| orderId | long | Order id |
| clientOrderId | string | Client order id |
| type | [OrderType](#OrderType) | Order type |
| isCloseOrder | bool | Whether the order is a close order |
| side | [OrderSide](#OrderSide) | Order side |
| price | decimal | Order price. Sent only when `type` is `limit` |
| size | long | Order size |
| timeInForce | [TimeInForce](#TimeInForce) | Time in force of the order |
| notionalValue | decimal | The order's notional value in settlement currency |
| status | [OrderStatus](#OrderStatus) | Order status |
| fillPrice | decimal | The average fill price. `0` if no matches |
| filledSize | long | The size matched |
| accumulatedFees | decimal | The accumulated fees incurred by this order |
| createTime | DateTime | The timestamp of order placement |
| updateTime | DateTime | The timestamp of last update |

### Position

| Field | Type | Remark |
| - | - | - |
| instrument | string | Id of the instrument |
| quantity | long | The number of contracts holding. Negative for short positions |
| leverage | decimal | The position leverage. `0` for cross margin mode |
| entryValue | decimal | The entry value in settlement currency |
| entryPrice | decimal | The average entry price of the current position. `0` if no active position |
| markPrice | decimal | The current mark price. `0` if no active position |
| liquidationPrice | decimal | The liquidation price. `0` if no active position |
| positionMargin | decimal | Amount of maintanance margin allocated to the position |
| buyOrderMargin | decimal | Amount of order margin allocated to open buy orders |
| sellOrderMargin | decimal | Amount of order margin allocated to open sell orders |
| unrealisedPnl | decimal | Unrealised pnl in settlement currency |
| realisedPnl | decimal | Realised pnl of the current active position |
| riskValue | decimal | Risk value of the current active position |
| riskLimit | decimal | The current risk limit |

### Orderbook

| Field | Type | Remark |
| - | - | - |
| bids | [decimal, long][] |A two-dimensional array containing price levels on the bid side |
| asks | [decimal, long][] |A two-dimensional array containing price levels on the ask side |

### Ticker

| Field | Type | Remark |
| - | - | - |
| instrument | string | Id of the instrument |
| bestBid | decimal | The real-time best bid price |
| bestAsk | decimal | The real-time best ask price |
| indexPrice | decimal | The real-time index price |
| markPrice | decimal | The real-time mark price |
| fundingRate | decimal | The funding rate of the current period |
| nextFundingTime | DateTime | Timestamp of the end of the current funding period |
| open | decimal | The open price of the sliding 24-hour window. Updated every 5 seconds  |
| high | decimal | The high price of the sliding 24-hour window. Updated every 5 seconds  |
| low | decimal | The low price of the sliding 24-hour windo. Updated every 5 seconds  |
| close | decimal | The close price of the sliding 24-hour window. Updated every 5 seconds  |
| volume | long | The number of contracts traded in the sliding 24-hour window. Updated every 5 seconds |
| openInterest | long | The real-time open interest in contracts |

### Match

| Field | Type | Remark |
| - | - | - |
| instrument | string | Id of the instrument |
| matchId | long | Id of the match. Unique under every instrument |
| price | decimal | The match price |
| size | long | The number of contracts traded |
| side | [OrderSide](#OrderSide) | The order side of the ***aggressing order*** |
| tickDirection | [TickDirection](#TickDirection) | The tick direction |
| timestamp | DateTime | Timestamp of the trade |

## Enumeration Types

These enumeration types are used in requests and responses.

### OrderType

Represents an order's type

| Value | Remark |
| - | - |
| limit | Limit order |
| market | Market order |

### OrderSide

Represents an order's direction

| Value | Remark |
| - | - |
| long | Buy / long order |
| short | Sell / short order |

### TimeInForce

Represents an order's time in force

| Value | Remark |
| - | - |
| gtc | Good till cancelled |
| ioc | Immediate or cancel |
| fok | Fill or kill |

### OrderStatus

| Value | Remark |
| - | - |
| new | New open order without any fills |
| partiallyFilled | Open order partillay filled |
| filled | Order completely filled |
| cancelled | Order cancelled |

### TickDirection

| Value | Remark |
| - | - |
| minusTick | Price lower than the last match |
| zeroMiusTick | Price equal to the last match, but lower than the last match at a different price |
| zeroPlusTick | Price equal to the last match, but higher than the last match at a different price |
| plusTick | Price higher than the last match |
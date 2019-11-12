# DueDEX REST API Documentation

# General

All restful apis, public or private, use the domain:

[https://api.duedex.com](https://api.duedex.com)

For Testnet, use this domain instead:

[https://api.testnet.duedex.com](https://api.testnet.duedex.com)

If you need to maintain a real-time state of your margins, orders, or positions, you should use our [fully-featured WebSocket channels](./WEBSOCKET_API.MD), the exact same data synchronisation mechanism used in the official website. Polling restful interfaces will result in stale data and rate limit breaches.

For REST requests with a body, the media type **must** be set to `application/json`. JSON is the only body format accepted by DueDEX.

# Rate Limit

In order to maintain high quality services to our clients, all API endpoints on DueDEX are rate-limited using a leaky bucket algorithm. Each user or IP address is granted a certain `quota` per minute. Quotas are consumed based on the `weight` of the operation.

Rate limits are imposed on `users` if the request is authenticated, and on `ip addresses` otherwise. Note that you may want to authenticate your requests even to public endpoints, as user quotas are higher and you won't be affected by other users under the same network as opposed to anonymous requests.

Since rate limits are imposed on `users`, different API keys of the same user share a single limit. Creating more API keys doesn't give you more quota. If you believe your API quota is insufficient for your needs, please contact us at [support@duedex.com](mailto:support@duedex.com).

The following headers are sent in all API responses indicating your total quota, quota used, and the unix timestamp (in seconds) when your quota fully recovers:

    X-Rate-Limit-Limit: 300
    X-Rate-Limit-Remaining: 100
    X-Rate-Limit-Reset: 1557850500

If your request will cause your quota to be overspent, the request will fail with status code **429** (too many requests). The failed request, despite not taking effect, will still consume your quota by the weight of `1`.

If the client keeps sending requests that result in **429**, the user / ip address will be automatically banned from using the API. Banned users will receive status code **403**, with `Retry-After` header indicating the time when the user / ip address will be unbanned.

# Authentication

This section demonstrates how authentication is done. Note that while some endpoints *require* authentication, requests to all endpoints *can* be authenticated.

## Authentication Headers

The following headers and compulsory for authenticated requests:

    Ddx-Timestamp: The unix timestamp in milliseconds when the request is created
    Ddx-Key: The api key id
    Ddx-Signature: The request signature (with example below)

An additional optional header can be sent to indicate when the request will expire:

    Ddx-Expiration: The unix timestamp in milliseconds when the request expires

If `Ddx-Expiration` is not sent, it is default to `5 seconds` after `Ddx-Timestamp`.

For a request to be valid, the following conditions must be met:

1. Request timestamp not set in the future (with deviations below 5 seconds allowed):

        Ddx-Timestamp < serverTime + 5,000

2. Request expiration not reached:

        serverTime < Ddx-Expiration

Note that if any of the four headers above is provided, the request will be considered authenticated and be failed if the authentication infomation doesn't match (even to public endpoints).

## Signature Algorithm

The request signature is a `HMAC-SHA256` hash with the API key secret as the secret key of the following message:

    message = "METHOD|PATH|TIMESTAMP|EXPIRATION|QUERY_STRING|RAW_BODY"

- METHOD

    The http method used (e.g. "GET", "POST") in uppercase

- PATH

    The full path to the request endpoint (e.g. "/v1/contract/BTCUSD")

- TIMESTAMP

    The same value as the header `Ddx-Timestamp`

- EXPIRATION

    The same value as the header `Ddx-Expiration`, ***only if*** `Ddx-Expiration` is set explicitly. If `Ddx-Expiration` is not set, this field ***must*** be left empty.

    That is, if the header is not set, the message effectively becomes:

        message = "METHOD|PATH|TIMESTAMP||QUERY_STRING|RAW_BODY"

- QUERY_STRING

    The exact query string used in the request, defined as the string after the question mark (?) in the request uri.

    If your request does not include a query string, leave this field empty.

- RAW_BODY

    The exact content sent with the request. Simply fill this field with the raw content sent over the wire.

    If your request does not include a body, leave this field empty.

Before applying the signature algorithm, the secret key must first be extracted from the Base64 encoded API secret:

    secretKey = base64.decode(apiSecret)

Then, apply the `HMAC-SHA256` algorithm on the message:

    signature = HMACSHA256(message, secretKey)

Finally, convert the signature into hexadecimal form (case-insensitive):

    Ddx-Signature: signature.toHex()

## Signature Example

In this example, we'll assume the following:

    Endpoint: POST /v1/order

    API Key: 13f1ab93-771d-4d59-bb6a-fe96f6b609ea
    API Secret: 2W2eSP3e0dp+lYMuY1MBUTqF2+8VbNRxDZ88zA7MliU=

    Timestamp: 1559211656342

    Formatted Request Payload (not the actual content sent):

    // For legibility only
    {
        instrument = "BTCUSD",
        type = "limit",
        side = "long",
        price = 8000,
        size = 10,
        timeInForce = "ioc"
    }

    Raw Request Payload (actual request body):

    {"instrument":"BTCUSD","type":"limit","side":"long","price":8000,"size":10,"timeInForce":"ioc"}

First, construct the complete message (note that since we're neither setting the expiration explicitly nor using a query string, both fields are left empty):

    message = "POST|/v1/order|1559211656342|||{\"instrument\":\"BTCUSD\",\"type\":\"limit\",\"side\":\"long\",\"price\":8000,\"size\":10,\"timeInForce\":\"ioc\"}"

Decode the API secret:

    secretKey = base64.decode("2W2eSP3e0dp+lYMuY1MBUTqF2+8VbNRxDZ88zA7MliU=")

    [217, 109, 158, 72, 253, 222, 209, 218, 126, 149, 131, 46, 99, 83, 1, 81, 58, 133, 219, 239, 21, 108, 212, 113, 13, 159, 60, 204, 14, 204, 150, 37]

Sign the message:

    signature = HMACSHA256(message, secretKey)

    [202, 60, 213, 245, 53, 243, 95, 23, 165, 191, 118, 232, 200, 143, 195, 197, 104, 219, 72, 49, 222, 203, 133, 207, 186, 82, 80, 138, 64, 252, 229, 241]

Convert the signature to hex:

    signature.toHex()

    ca3cd5f535f35f17a5bf76e8c88fc3c568db4831decb85cfba52508a40fce5f1

Set the signature header:

    Ddx-Signature: ca3cd5f535f35f17a5bf76e8c88fc3c568db4831decb85cfba52508a40fce5f1

# Responses

All restful endpoints share a common response data structure, with the only difference being the `data` field (if any).

## General Structure

The common response structure:

    {
        "code": 0,
        "data": {
            ...
        },
        "message": "A short message ONLY WHEN the request is not successful"
    }

Note that you should *not* assume that the `data` filed is always an object (could be an array), or even present. Some endpoints have no response data and this field will simply not be sent.

Also note that the content of the `message` field is for *diagnostics only*. You *must not* rely on this message in your application's login. We might change the error messages without prior notice.

The content of `message` is internationalisable. Supply an `Accept-Language` header with a supported language to find out.

## Response Codes

The `code` field in the response represents the execution result of your request. A response code of `0` indicates success; non-zero codes indicate errors. The latest list of response codes are shown below.

Note that you should expect this to *not* be a exhaustive list and handle unknown response codes properly, as we might add new response codes without prior notice.

| Code | Message |
| - | - |
| 0 | Success |

# API Endpoints

## Orders

### Place a New Order

    POST /v1/order

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **trade** permission

- Query Parameters

    This endpoint does not accept any query parameters

- Body Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | clientOrderId | string | true | - | Unique client id under the instrument with a maximum of 36 characters |
    | type | [OrderType](./DATA_TYPES.MD#OrderType) | false | | Order type |
    | isCloseOrder | bool | true | false if `type` is `limit` or `market`; true if `type` is `stopLimit` or `stopMarket` | Whether the order is a close order that can only reduce positions |
    | side | [OrderSide](./DATA_TYPES.MD#OrderSide) | true | - | Order side. Can be omitted only when `type` is `limit` or `market` and `isCloseOrder` is `true` |
    | price | decimal | true | - | Order price. Required only when `type` is `limit` or `stopLimit` |
    | size | long | true | - | Order quantity. Can be omitted only when `type` is `limit` or `market` and `isCloseOrder` is `true` |
    | stopPrice | decimal | true | - | Order stop price. Required only when `type` is `stopMarket` or `stopLimit` |
    | triggerType | [StopTriggerType](./DATA_TYPES.MD#StopTriggerType) | true | lastPrice | Stop order trigger type. Required only when `type` is `stopMarket` or `stopLimit` |
    | timeInForce | [TimeInForce](./DATA_TYPES.MD#TimeInForce) | true | `gtc` if `type` if `limit` or `stopLimit`; `ioc` if `type` is `market` or `stopMarket` | Time in force of the order |
    | postOnly | bool | true | false | Whether this order can only be maker |

    > **NOTE**
    >
    > The `price`, `stopPrice`, and `triggerType` fields are required only under certain conditions. You **must not** supply them otherwise. 

    > **NOTE**
    >
    > Fields `side` and `size` can be omitted when `type` is `limit` or `market` and `isCloseOrder` is `true`. If you omit them, the system will generate them for you automatically based on your active position.

    > **NOTE**
    >
    > You must supply either both or none of fields `side` and `size`. In other words, if you supply `side`, then you must supply `size` as well, and vice versa.

- Response

    The response data field is an instance of the [Order](./DATA_TYPES.MD#Order) structure.

### Cancel Order

    DELETE /v1/order

> **NOTE**
>
> To cancel all active orders under an instrument, do not send `clientOrderId` or `orderId`.

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **trade** permission

- Query Parameters

    This endpoint does not accept any query parameters

- Body Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | clientOrderId | string | true | - | Client order id. One and only one of `clientOrderId` and `orderId` must be provided to cancel a specific order. Ignore this parameter if you want to cancel all orders under an instrument |
    | orderId | long | true | - | Id of the order. One and only one of `clientOrderId` and `orderId` must be provided to cancel a specific order. Ignore this parameter if you want to cancel all orders under an instrument |

- Response

    This endpoint has no response data.

### Amend Order

    PATCH /v1/order

> **NOTE**
>
> Currently only order price amendment is supported. More amendment capabilities coming soon.

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **trade** permission

- Query Parameters

    This endpoint does not accept any query parameters

- Body Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | clientOrderId | string | true | - | Client order id. One and only one of `clientOrderId` and `orderId` must be provided |
    | orderId | long | true | - | Id of the order. One and only one of `clientOrderId` and `orderId` must be provided |
    | price | decimal | true | - | New order price |

- Response

    This endpoint has no response data.

### Get Active Orders

    GET /v1/order/active

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **view** permission

- Query Parameters

    This endpoint does not accept any query parameters

- Body Parameters

    This endpoint does not accept any body parameters

- Response

    The response data field is an array of the [Order](./DATA_TYPES.MD#Order) structure.

### Get a Single Order

    GET /v1/order

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **view** permission

- Query Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | orderId | long | true | - | Id of the order. One and only one of `clientOrderId` and `orderId` msut be provided |
    | clientOrderId | string | true | - | Client order id. One and only one of `clientOrderId` and `orderId` msut be provided |

- Body Parameters

    This endpoint does not accept any body parameters

- Response

    The response data field is an instance of the [Order](./DATA_TYPES.MD#Order) structure.

## Positions

### Change Leverage

    POST /v1/position/leverage

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **trade** permission

- Query Parameters

    This endpoint does not accept any query parameters

- Body Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | leverage | decimal | false | | Target leverage. `0` for cross margin |

- Response

    This endpoint has no response data.

### Change Risk Limit

    POST /v1/position/riskLimit

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **trade** permission

- Query Parameters

    This endpoint does not accept any query parameters

- Body Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | riskLimit | decimal | false | | Target risk limit |

- Response

    This endpoint has no response data.

### Transfer Margin

    POST /v1/position/margin/transfer

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **trade** permission

- Query Parameters

    This endpoint does not accept any query parameters

- Body Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | amount | decimal | false | | Amount of margin to transfer into the position. Negative for outward transfers |

- Response

    This endpoint has no response data.

### Get a Single Position

    GET /v1/position/{instrument}

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **view** permission

- Query Parameters

    This endpoint does not accept any query parameters

- Body Parameters

    This endpoint does not accept any body parameters

- Response

    The response data field is an instance of the [Position](./DATA_TYPES.MD#Position) structure.

## Margins

### Get Margin of Currency

    GET /v1/margin/{currencySymbol}

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **view** permission

- Query Parameters

    This endpoint does not accept any query parameters

- Body Parameters

    This endpoint does not accept any body parameters

- Response

    The response data field is an instance of the [Margin](./DATA_TYPES.MD#Margin) structure.
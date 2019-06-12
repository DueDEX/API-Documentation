# DueDEX REST API Documentation

# General

All restful apis, public or private, use the domain:

[https://api.duedex.com](https://api.duedex.com)

For Testnet, use this domain instead:

[https://api.testnet.duedex.com](https://api.testnet.duedex.com)

If you need to maintain a real-time state of your margins, orders, or positions, you should use our [fully-featured WebSocket channels](./WEBSOCKET_API.MD), the exact same data synchronisation mechanism used in the official website. Polling restful interfaces will result in stale data and rate limit breaches.

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

    message = "METHOD|PATH|TIMESTAMP|EXPIRATION|PARLIST"

- METHOD

    The http method used (e.g. "GET", "POST") in uppercase

- PATH

    The full path to the request endpoint (e.g. "/v1/contract/BTCUSD")

- TIMESTAMP

    The same value as the header `Ddx-Timestamp`

- EXPIRATION

    The same value as the header `Ddx-Expiration`, ***only if*** `Ddx-Expiration` is set explicitly. If `Ddx-Expiration` is not set, this field ***must*** be left empty.

    That is, if the header is not set, the message effectively becomes:

        message = "METHOD|PATH|TIMESTAMP||PARLIST"

- PARLIST

    The parameter name-value list sorted by name, separated by `&` (as in http query strings), with values URL-encoded (again, as in http query strings).

    For `GET` requests, the parameter list is the sorted query string. If no query parameter is provided, the field must be left empty.

    For requests with a body, the parameter list includes parameters from *both* the query string and request body. For example, the following request:

        POST /v1/example?b=100&a=200

        {"c":300.0,"d":"my string"}
    
    would have the field set to:

        a=200&b=100&c=300.0&d=my%20string

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

    Request Payload:
    {
        instrument = "BTCUSD",
        type = "limit",
        side = "long",
        price = 8000,
        size = 10,
        timeInForce = "ioc"
    }

First, sort the parameter list:

    parList = "instrument=BTCUSD&price=8000&side=long&size=10&timeInForce=ioc&type=limit"

Prepend other fields to construct the complete message (note that since we're not setting the expiration explicitly, the field is left empty):

    message = "POST|/v1/order|1559211656342||instrument=BTCUSD&price=8000&side=long&size=10&timeInForce=ioc&type=limit"

Decode the API secret:

    secretKey = base64.decode("2W2eSP3e0dp+lYMuY1MBUTqF2+8VbNRxDZ88zA7MliU=")

    [217, 109, 158, 72, 253, 222, 209, 218, 126, 149, 131, 46, 99, 83, 1, 81, 58, 133, 219, 239, 21, 108, 212, 113, 13, 159, 60, 204, 14, 204, 150, 37]

Sign the message:

    signature = HMACSHA256(message, secretKey)

    [121, 234, 227, 119, 15, 52, 49, 162, 191, 26, 7, 188, 44, 36, 133, 2, 92, 204, 66, 215, 250, 173, 250, 76, 165, 109, 4, 20, 204, 96, 104, 228]

Convert the signature to hex:

    signature.toHex()

    79eae3770f3431a2bf1a07bc2c2485025ccc42d7faadfa4ca56d0414cc6068e4

Set the signature header:

    Ddx-Signature: 79eae3770f3431a2bf1a07bc2c2485025ccc42d7faadfa4ca56d0414cc6068e4

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

- Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | clientOrderId | string | true | - | Unique client id under the instrument with a maximum of 36 characters |
    | type | [OrderType](./DATA_TYPES.MD#OrderType) | false | | Order type |
    | isCloseOrder | bool | true | false | Whether the order is a close order |
    | side | [OrderSide](./DATA_TYPES.MD#OrderSide) | true | - | Order side. Required only when `isCloseOrder` is `false` |
    | price | decimal | true | - | Order price. Required only when `type` is `limit` |
    | size | long | true | - | Order quantity. Required only when `isCloseOrder` is `false` |
    | timeInForce | [TimeInForce](./DATA_TYPES.MD#TimeInForce) | true | `gtc` if `type` if `limit`; `ioc` if `type` is `market` | Time in force of the order |

    > **NOTE**
    >
    > `side`, `price`, and `size` fields are required only under certain conditions. You **must not** supply them otherwise. 

- Response

    The response data field is an instance of the [Order](./DATA_TYPES.MD#Order) structure.

### Cancel Order

    DELETE /v1/order

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **trade** permission

- Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | clientOrderId | string | true | - | Client order id. One and only one of `clientOrderId` and `orderId` msut be provided |
    | orderId | long | true | - | Id of the order. One and only one of `clientOrderId` and `orderId` msut be provided |

- Response

    This endpoint has no response data.

## Positions

### Change Leverage

    POST /v1/position/leverage

- Weight

    This endpoint has a rate limit weight of `1`

- Permission

    This endpoint requires the **trade** permission

- Parameters

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

- Parameters

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

- Parameters

    | Field | Type | Optional | Default | Remark |
    | - | - | - | - | - |
    | instrument | string | false | | Id of the instrument |
    | amount | decimal | false | | Amount of margin to transfer into the position. Negative for outward transfers |

- Response

    This endpoint has no response data.
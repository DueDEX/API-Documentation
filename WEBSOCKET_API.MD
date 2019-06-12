# DueDEX Websocket API Documentation

# General

DueDEX offers a fully-featured WebSocket API for real-time streaming, *the same endpoint* used by our official website, thus enabling you to automate any real-time functionality you see on the web app.

To connect to the WebSocket feed, use this endpoint:

**wss://feed.duedex.com/v1/feed**

For Testnet, use this endpoint instead:

**wss://feed.testnet.duedex.com/v1/feed**

# Authentication

You need to authenticate your session if you want to subscribe to private channels. In case the connection is lost, your have to go through the authentication process again after reconnecting.

Note that authentication is *not* mandatory if you're only subscribing to public channels.

Your API key would need to have `view` permission to successfully authenticate.

## Authentication Flow

To authenticate, the client first sends a `challenge` message:

    {
        "type": "challenge"
    }

Upon receiving the `challenge` request, the server responds with a `challenge` response, with a random string for you to sign:

    {
        "type": "challenge",
        "challenge": "1f94d8db-a9ca-4cf4-82ee-9692f937032e"
    }

The client then signs (with algorithm and example below) the challenge to generate an answer, and sends it via an `auth` message alongside your API key:

    {
        "type": "auth",
        "key": "YOUR_API_KEY_ID_HERE",
        "answer": "3563bd49f03177a4aa5efce5ad2042d4416452523eb42d8ddf11b946b0e1d267"
    }

If the answer is correct, the server responds with an `auth` message with your user id:

    {
        "type": "auth",
        "userId": 10
    }

If the answer is wrong, the connection is terminated immediately and you need to reconnect to start over.

## Signature Algorithm

Each API key comes with an ID and a secret (which is shown only once upon creation). The key ID is sent in the `auth` request, and the secret is used to sign the challenge in order to generate an answer.

The secret is Base64 encoded. To use it for signature purposes, decode it first:

    secretKey = base64.decode(apiSecret)

Then, apply the `HMAC-SHA256` algorithm on the challenge (treat the challenge as plain string for signature):

    signature = HMACSHA256(challenge, secretKey)

Finally, convert the signature into hexadecimal form (case-insensitive):

    answer = signature.toHex()

### Example

In this example, we'll assume the following:

    API Key: 13f1ab93-771d-4d59-bb6a-fe96f6b609ea
    API Secret: 2W2eSP3e0dp+lYMuY1MBUTqF2+8VbNRxDZ88zA7MliU=

    Challenge: fd14408d-1740-447d-b335-c019f9201b6e

First, decode the API secret into byte array:

    secretKey = base64.decode("2W2eSP3e0dp+lYMuY1MBUTqF2+8VbNRxDZ88zA7MliU=")

    [217, 109, 158, 72, 253, 222, 209, 218, 126, 149, 131, 46, 99, 83, 1, 81, 58, 133, 219, 239, 21, 108, 212, 113, 13, 159, 60, 204, 14, 204, 150, 37]

Then apply the hashing algorithm:

    signature = HMACSHA256("fd14408d-1740-447d-b335-c019f9201b6e", secretKey)

    [180, 24, 237, 212, 102, 155, 130, 171, 55, 165, 182, 213, 68, 107, 157, 239, 56, 102, 88, 232, 243, 239, 3, 22, 89, 53, 255, 107, 114, 254, 167, 16]

Some libraries return the hexadecimal representation in this step directly. If your library doesn't, convert the byte array into the hex form:

    answer = signature.toHex()

    b418edd4669b82ab37a5b6d5446b9def386658e8f3ef03165935ff6b72fea710

# Using WebSocket Streams

After successful connection to the WebSocket endpoint, no messages will be sent to you by default. To receive real-time data and maintain an up-to-date state in-memory, follow these steps:

## Subscribe

Send a `subscribe` request (with details in the [next section](#Subscription)) to listen to updates in channels.

## Handle `snapshot` Messasge

After successful subscription, the server will send a message with type `snapshot`, which contains the latest data in the channel.

> **NOTE**
>
> A `snapshot` message will always be sent upon subscription, regardless of whether you have already subscribed to the channel.

For example, in the `level2` channel, the `snapshot` message contains the complete orderbook data with all bids and asks.

The `snapshot` message contains a `sequence` field, which is used for processing incremental updates.

## Handle `update` Message

After the `snapshot` message is sent, the server will start pushing `update` messages. The `update` messages contain *only* incremental updates to the snapshot. The client is responsible for applying the updates to the snapshot locally to keep the state up-to-date.

While the server tries to send the `snapshot` message right after a subscription request is received, it's still possible that the client receives `update` messages before the big `snapshot` message arrives. In this case, the client *must* buffer these `update` messages and process them after the `snapshot` arrives.

`update` messages also contain a `sequence` field corresponding to the one in `snapshot` messages.

### Relationship between `sequence` in `update` and `snapshot`

- The initial empty snapshot has a `sequence` of `0`

- The first-ever `update` to the snapshot has a `sequence` of `1`

- After applying the update `#1`, the `sequence` of the snapshot increments to `1` as well

- The next update has a sequence of `2`...

### Ignoring `update` Messages

As mentioned above, it is possible the client receives `update` messages before the `snapshot` message. The correct logic for handling such case is as follows:

- Subscribe to the channel and listen to messages

- Buffer `update` messages if they arrive before the `snapshot` message

- Receive `snapshot` message

- If the buffer is **not** empty, remove `update` messages with `sequence` smaller than or equal to that of `snapshot`

- If the buffer is still **not** empty, apply the remaining `update` messages

- Clear the buffer

- Apply subsequent `update` messages directly on the snapshot

### Cases in which You should Resubscribe

In rare cases, the `sequence` in `snapshot` and `update` messages might not connect:

- The first `update` message received has a `sequence` greater than the snapshot `sequence` plus one

- `sequence` field in a `update` message does not equal the last `sequence` plus one

In these cases, you should first unsubscribe and then subscribe again to retry.

# Subscription

## Channel Types

With regard to whether authentication is required, channels can be divided into:

- **Public Channels**

    Authentication is not required (e.g. `level2`)

- **Private Channels**

    Authentication is required (e.g. `orders`)

With regard to whether the channel is instrument-specific, channels can be divided into:

- **Instrument Channels**

    The channel is instrument-specific (e.g. `matches`)

- **Non-Instrument Channels**
    
    The channel is not instrument-specific (e.g. `positions`)

## Subcribe

To subscribe to a channel, send a `subscribe` request:

    {
        "type": "subscribe",
        "channels": [
            {
                "name": "level2",
                "instruments": [
                    "BTCUSD"
                ]
            }
        ]
    }

Note that you *must not* send the `instruments` field for non-instrument channels. You can set to to an empty array or ignore the field altogether.

If the request is invalid (invalid instrument ids, not authenticated for private channels, etc.), the connection will be aborted immediately.

Upon successful subscription, a response is sent from the server:

    {
        "type": "subscriptions",
        "channels": [
            {
                "name": "level2",
                "instruments": [
                    "BTCUSD"
                ]
            }
        ]
    }

## Unsubscribe

Unsubcribing is the same as subscribing, except the `type` field is changed from `subscribe` to `unsubscribe`. For instance, to unsubscribe from the channel above:

    {
        "type": "unsubscribe",
        "channels": [
            {
                "name": "level2",
                "instruments": [
                    "BTCUSD"
                ]
            }
        ]
    }

# Atomicity

DueDEX supports batching of atomic updates. Multiple updates sent within an `update` message are atomic, meaning that no intermediate state exists between each update.

In your application, you must *not* trigger actions on each update item. Instead, only trigger processing after the whole `update` message is completely processed.

# Channel Messages

The structures for `snapshot` and `update` messages are the same across all channels, which significantly simplifies development.

## `snapshot` Messages

`snapshot` message format:

    {
        "type": "snapshot",
        "channel": CHANNEL_NAME,
        "instrument": INSTRUMENT_ID_IF_ANY,
        "sequence": SNAPSHOT_SEQUENCE,
        "data": CHANNEL_DATA_TYPE,
        "timestamp": UNIX_TIMESTAMP_MILLISECONDS
    }

where the `data` field is the channel-specific data type.

The `snapshot` message contains a complete state.

An example `snapshot` message:

    {
        "type": "snapshot",
        "channel": "level2",
        "instrument": "BTCUSD",
        "sequence": 100,
        "data": {
            "bids": [
                [
                    "8804.00",
                    100
                ],
                [
                    "8803.00",
                    100
                ]
            ],
            "asks": [
                [
                    "8855.00",
                    600
                ],
                [
                    "8856.00",
                    1000
                ],
                [
                    "8857.00",
                    1000
                ]
            ]
        },
        "timestamp": "2019-05-30T00:00:00.000Z"
    }

## `update` Messages

`update` message format:

    {
        "type": "update",
        "channel": CHANNEL_NAME,
        "instrument": INSTRUMENT_ID_IF_ANY,
        "sequence": SNAPSHOT_SEQUENCE,
        "data": CHANNEL_DATA_TYPE,
        "timestamp": UNIX_TIMESTAMP_MILLISECONDS
    }

which is identical to the format of `snapshot` messages except the `type` field.

As opposed to `snapshot`, `update` messages only contain the incremental updates. The `data` field (or each element within, if `data` is an array) contains *only* the item's primary key for you to locate the item, and the fields being updated.

If the primary key provided doesn't correspond to an item in your local state, it means a new item is created.

> **TIPS**
>
> Some channels are insert-only (e.g. the `matches` channel - obviously you can't modify an existing match). In these channels, the `data` field will present all the fields of the model in `update` messages.

An example `update` message:

    {
        "type": "update",
        "channel": "margins",
        "sequence": 9,
        "data": [
            {
                "currency": "BTC",
                "available": "1.948500",
                "orderMargin": "0.011500"
            }
        ],
        "timestamp": "2019-05-30T00:00:00.000Z"
    }

# Channels

## `level2` Channel

This channel maintains the latest level-2 orderbook under an instrument.

- **Channel Name**

    level2

- **Channel Type**

    Public Instrument Channel

- **Data Model**

    The `data` field on this channel is an **instance** of [Orderbook](./DATA_TYPES.MD#Orderbook)

## `ticker` Channel

This channel maintains the latest ticker under an instrument.

- **Channel Name**

    ticker

- **Channel Type**

    Public Instrument Channel

- **Data Model**

    The `data` field on this channel is an **instance** of [Ticker](./DATA_TYPES.MD#Ticker)

## `matches` Channel

This channel maintains the recent matches under an instrument.

- **Channel Name**

    matches

- **Channel Type**

    Public Instrument Channel

- **Data Model**

    The `data` field on this channel is an **array** of [Match](./DATA_TYPES.MD#Match)

## `orders` Channel

This channel maintains the open orders of a user.

- **Channel Name**

    orders

- **Channel Type**

    Private Non-Instrument Channel

- **Data Model**

    The `data` field on this channel is an **array** of [Order](./DATA_TYPES.MD#Order)

## `positions` Channel

This channel maintains positions under all instruments of a user.

> **NOTE**
>
> The position record won't be removed from state even if you don't have an active position. Instead, empty positions have their `quantity` fields set to `0`.

- **Channel Name**

    positions

- **Channel Type**

    Private Non-Instrument Channel

- **Data Model**

    The `data` field on this channel is an **array** of [Position](./DATA_TYPES.MD#Position)

## `margins` Channel

This channel maintains margins of all currencies of a user.

- **Channel Name**

    margins

- **Channel Type**

    Private Non-Instrument Channel

- **Data Model**

    The `data` field on this channel is an **array** of [Margin](./DATA_TYPES.MD#Margin)

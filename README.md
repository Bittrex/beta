# Bittrex Beta API Documentation
## Overview
On March 27, 2018, Bittrex released the Beta version of its new website, which includes several performance, usability, and security improvements to our API. 

The following sections detail the most significant changes. Anyone using the Bittrex API should read carefully the information below to understand whether there are any potential impacts to their systems. While unlikely, if we determine additional API changes are needed during our Beta site testing, we will provide users the updated information when the official website launches. 

### Access to Account and Exchange Data
We have identified common API patterns used by bots and other trading software to obtain up-to-date information from Bittrex. In order to help streamline the process for developers and to prevent potentially abusive behavior, we are providing new methods for accessing account and exchange data, and we strongly encourage all developers to leverage these tools. (See [API Best Practices](#api-best-practices))

### Throttling
Improper API use affects the efficiency of the platform for our customers, and we have enabled throttling on all endpoints to mitigate the adverse effects of this improper behavior. Accounts will be permitted to make a maximum of 60 API calls per minute, and calls after the limit will fail, with throttle settings automatically resetting at the start of the next minute.

_**Note:** Corporate and high-volume accounts may contact customer support for additional information to ensure that they may continue operating at an optimal level._

### Websocket API (WS API)
To help ensure these changes do not present issues for customers, we are encouraging developers to use our Websocket API to have the most commonly-requested data pushed to them instead of needing to poll the REST API. The WS API supplies public market data (e.g. exchange status, summary ticks) and account-level information such as order and balance status.



## API Best Practices
Below, users may find best practices for the most common API scenarios. By implementing these recommendations, customers using API may ensure timely access to account and exchange data while minimizing the possibility of being throttled due to improper use or blacklisted due to suspected malicious behavior.

### Obtaining Order and Balance Status
Instead of polling the REST-based account and market APIs for open order and balance information, developers should use the WS API and subscribe to account-level data, detailed in [WS API Overview](#ws-api-overview).

### Tracking an Order Book
Many users poll the REST API as quickly as possible for updated order data. At very high call rates, the API most often returns the same data because we only update it **_once per second_**. By using the WS API’s `QueryExchangeState` and `SubscribeToExchangeDeltas` functions, developers can maintain their own copies of one or more order books.



## WS API Overview
The Bittrex Websocket API is implemented using [SignalR](https://www.asp.net/signalr), and while several SignalR clients exist for different languages, if one is not available for your environment, you will need to write one.

_**Note:** Several third-party libraries have code written for the Bittrex API. Bittrex does not review, support, or endorse any of them. Users should beware and not provide their API key or any other credentials to unknown and/or untrusted code._

### Type Conventions
Type information is given in the JSON-like payload descriptions.

Type|Description
----|-----------
`date` | ISO8601 UTC date-time string.
`decimal` | string-formatted decimal with 18 significant digits and 8 digit precision.
`guid` | string-formatted UUID
`int` | signed 32-bit integer
`long` | signed 64-bit integer

### Connecting
The Beta WS API endpoint is https://beta.bittrex.com/signalr. Once connected, be sure to connect to the `c2` hub. No other hubs are supported for use by Bittrex customers.

### Response Handling
All responses are compressed by the server using GZip (via a 'deflate' API - there are no headers) and base64 encoded prior to transmission. Users must reverse this process to retrieve the JSON payload. Further, field keys in the response JSON are minified. [Appendix A](#appendix-a-minified-json-keys) contains a table of the keys and their un-minified counterparts.

### Subscribing to Account-Level Data
Once connected, developers can obtain account-level data using the following steps:

1.	Obtain an authentication challenge by calling `GetAuthContext`.
2.	Sign the challenge using the request-signing method documented in the [v1.1 API](https://bittrex.com/Home/Api).
3.	Call `Authenticate`.

The details for each API call are documented in the next section.



## WS API Reference

### `GetAuthContext`
#### Description
Generates a challenge developers can sign and use in the `Authenticate` call to verify their identity and begin receiving account-level notifications.
#### Parameters

Name|Type|Description
----|----|-----------
apiKey|string|A valid  API key for your account.
#### Response
A string of challenge data to be used in `Authenticate`.

---

### `Authenticate`
#### Description
Verifies a user’s identity to the server and begins receiving account-level notifications. Users must sign the challenge returned by `GetAuthContext` using the [v1.1 API's](https://bittrex.com/Home/Api) request signing method before calling this function.

To receive the account-level notifications enabled by authenticating, the caller must register callbacks for the `uO` and `uB` events through their SignalR client. See [Appendix B](#appendix-b-callbacks-and-payloads) for event payload details and see the [sample code](./samples/WebsocketSample.cs) for an example of how to subscribe using the C# SignalR library.
#### Parameters
Name|Type|Description
----|----|-----------
apiKey|string|A valid  API key for your account.
response|string|Signed challenge from `GetAuthContext`.
#### Response
Boolean indication of success or failure.

---

### `QueryExchangeState`
#### Description
Allows the caller to retrieve the full order book for a specific market.
#### Parameters
Name|Type|Description
----|----|-----------
marketName|string|The market identifier, e.g. `BTC-ETH`.
#### Response
JSON object containing market state.
```
{
    MarketName : string,
    Nonce      : int,
    Buys: 
    [
        {
            Quantity : decimal,
            Rate     : decimal
        }
    ],
    Sells: 
    [
        {
            Quantity : decimal,
            Rate     : decimal
        }
    ],
    Fills: 
    [
        {
            Id        : int,
            TimeStamp : date,
            Quantity  : decimal,
            Price     : decimal,
            Total     : decimal,
            FillType  : string,
            OrderType : string
        }
    ]
}
```

---

### `QuerySummaryState`
#### Description
Allows the caller to retrieve the full state for all markets.
#### Response
JSON object containing state data for all markets.
```
{
    Nonce     : int,
    Summaries : 
    [
        {
            MarketName     : string,
            High           : decimal,
            Low            : decimal,
            Volume         : decimal,
            Last           : decimal,
            BaseVolume     : decimal,
            TimeStamp      : date,
            Bid            : decimal,
            Ask            : decimal,
            OpenBuyOrders  : int,
            OpenSellOrders : int,
            PrevDay        : decimal,
            Created        : date
        }
    ]
}
```

---

### `SubscribeToExchangeDeltas`
#### Description
Allows the caller to receive real-time updates to the state of a single market. The caller must register a callback for the uE event through their SignalR client. Upon subscribing, the callback will be invoked with market deltas as they occur. See [Appendix B](#appendix-b-callbacks-and-payloads) for event payload details and see the [sample code](./samples/WebsocketSample.cs) for an example of how to subscribe using the C# SignalR library.

_**Note:** This feed only contains updates to exchange state. To form a complete picture of exchange state, users must first call QueryExchangeState and merge deltas into the data structure returned in that call._
#### Parameters
Name|Type|Description
----|----|-----------
marketName|String|The market identifier, e.g. BTC-ETH
#### Response
Boolean indicating whether the user was subscribed to the feed.

---

### `SubscribeToSummaryDeltas`
#### Description
Allows the caller to receive real-time updates of the state of all markets. The caller must register a callback for the `uS` event through their SignalR client. Upon subscribing, the callback will be invoked with market deltas as they occur. See [Appendix B](#appendix-b-callbacks-and-payloads) for event payload details and see the [sample code](./samples/WebsocketSample.cs) for an example of how to subscribe using the C# SignalR library.

_**Note:** Summary delta callbacks are verbose. A subset of the same data limited to the market name, the last price, and the base currency volume can be obtained via `SubscribeToSummaryLiteDeltas`, just use `uL` instead of `uS.`_
#### Response
Boolean indicating whether the user was subscribed to the feed.



# Appendix A: Minified JSON Keys
```
"A" = "Ask"
"a" = "Available"
"B" = "Bid"
"b" = "Balance"
"C" = "Closed"
"c" = "Currency"
"CI" = "CancelInitiated"
"D" = "Deltas"
"d" = "Delta"
"DT" = "OrderDeltaType"
"E" = "Exchange"
"e" = "ExchangeDeltaType"
"F" = "FillType"
"FI" = "FillId"
"f" = "Fills"
"G" = "OpenBuyOrders"
"g" = "OpenSellOrders"
"H" = "High"
"h" = "AutoSell"
"I" = "Id"
"i" = "IsOpen"
"J" = "Condition"
"j" = "ConditionTarget"
"K" = "ImmediateOrCancel"
"k" = "IsConditional"
"L" = "Low"
"l" = "Last"
"M" = "MarketName"
"m" = "BaseVolume"
"N" = "Nonce"
"n" = "CommissionPaid"
"O" = "Orders"
"o" = "Order"
"OT" = "OrderType"
"OU" = "OrderUuid"
"P" = "Price"
"p" = "CryptoAddress"
"PD" = "PrevDay"
"PU" = "PricePerUnit"
"Q" = "Quantity"
"q" = "QuantityRemaining"
"R" = "Rate"
"r" = "Requested"
"S" = "Sells"
"s" = "Summaries"
"T" = "TimeStamp"
"t" = "Total"
"TY" = "Type"
"U" = "Uuid"
"u" = "Updated"
"V" = "Volume"
"W" = "AccountId"
"w" = "AccountUuid"
"X" = "Limit"
"x" = "Created"
"Y" = "Opened"
"y" = "State"
"Z" = "Buys"
"z" = "Pending"
```


# Appendix B: Callbacks and Payloads

## `Balance Delta - uB`
### Callback For
`Authenticate`
### JSON Payload
```
{
    Nonce : int,
    Delta : 
    {
        Uuid          : guid,
        AccountId     : int,
        Currency      : string,
        Balance       : decimal,
        Available     : decimal,
        Pending       : decimal,
        CryptoAddress : string,
        Requested     : bool,
        Updated       : date,
        AutoSell      : bool
    }
}
```


## `Market Delta - uE`
### Callback For
`SubscribeToExchangeDeltas`
### Notes
`The Type key can be one of the following values: 0 = ADD, 1 = REMOVE, 2 = UPDATE`
### JSON Payload
```
{
    MarketName : string,
    Nonce      : int,
    Buys: 
    [
        {
            Type     : int,
            Rate     : decimal,
            Quantity : decimal
        }
    ],
    Sells: 
    [
        {
            Type     : int,
            Rate     : decimal,
            Quantity : decimal
        }
    ],
    Fills: 
    [
        {
            FillId    : int,
            OrderType : string,
            Rate      : decimal,
            Quantity  : decimal,
            TimeStamp : date
        }
    ]
}
```


## `Lite Summary Delta - uL`
### Callback For
`SubscribeToSummaryLiteDeltas`
### JSON Payload
```
{
    Deltas : 
    [
        {
            MarketName : string,
            Last       : decimal,
            BaseVolume : decimal
        }
    ]
}
```


## `Order Delta - uO`
### Callback For
`Authenticate`
### Notes
`The Type key can be one of the following values: 0 = OPEN, 1 = PARTIAL, 2 = FILL, 3 = CANCEL`
### JSON Payload
```
{
    AccountUuid : Guid,
    Nonce       : int,
    Type        : int,
    Order: 
    {
        Uuid              : guid,
        Id                : long,
        OrderUuid         : guid,
        Exchange          : string,
        OrderType         : string,
        Quantity          : decimal,
        QuantityRemaining : decimal,
        Limit             : decimal,
        CommissionPaid    : decimal,
        Price             : decimal,
        PricePerUnit      : decimal,
        Opened            : date,
        Closed            : date,
        IsOpen            : bool,
        CancelInitiated   : bool,
        ImmediateOrCancel : bool,
        IsConditional     : bool,
        Condition         : string,
        ConditionTarget   : decimal,
        Updated           : date
    }
}
```


## `Summary Delta - uS`
### Callback For
`SubscribeToSummaryDeltas`
### JSON Payload
```
{
    Nonce : int,
    Deltas : 
    [
        {
            MarketName     : string,
            High           : decimal,
            Low            : decimal,
            Volume         : decimal,
            Last           : decimal,
            BaseVolume     : decimal,
            TimeStamp      : date,
            Bid            : decimal,
            Ask            : decimal,
            OpenBuyOrders  : int,
            OpenSellOrders : int,
            PrevDay        : decimal,
            Created        : date
        }
    ]
}
```

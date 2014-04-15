# INTRODUCTION #

## Ripple-REST API (BETA) ##

`ripple-rest` API is currently in BETA and subject to multiple changes and iterations as it is being finalized. Please double check with the <a href="https://github.com/ripple/ripple-rest" target="_blank">`ripple-rest`</a> github repo for the most up-to-date versions and documentation. Feel free to poke around the branches to see what we're working on changing as well.

The `ripple-rest` API makes it easy to access the Ripple system via a RESTful web interface.  In this section, we will cover the concepts you need to understand, and get you started accessing the API and learning how to use it.

While there are different APIs that you can use, for example by accessing the `rippled` server directly via a web socket, this documentation focuses on the `ripple-rest` API as this is the high-level API recommended for working with the Ripple system.

**The test version of the `ripple-rest` server has been temporarily taken down to enhance security around a sandboxed environment. We encourage you to install a local version of `ripple-rest` to test for the meantime. Instructions can be found <a href="https://github.com/ripple/ripple-rest" target="_blank">here</a>**

## Available API Routes ##

* [`GET /v1/accounts/:account/payments/paths/:dst_account/:dst_amount`](#preparing-a-payment)
* [`POST /v1/payments`](#submitting-a-payment)
* [`GET /v1/accounts/:account/payments/:transaction_id`](#confirming-a-payment) 
* [`GET /v1/accounts/:account/payments`](#payment-history)
* [`GET /v1/accounts/:account/balances`](#account-balances)
* [`GET /v1/accounts/:account/settings`](#account-settings)
* [`GET /v1/server/connected`](#check-connection-state)
* [`GET /v1/status`](#check-server-status)
* [`GET /v1/transactions/:tx_hash`](#retrieve-ripple-transaction)
* [`GET /v1/uuid1`](#create-client-resource-id)

## API Overview ##

### Ripple Concepts ###

Ripple is a system for making financial transactions.  You can use Ripple to send money anywhere in the world, in any currency, instantly and for free.

In the Ripple world, each account is identified by a <a href="https://ripple.com/wiki/Account" target="_blank">Ripple Address</a>.  A Ripple address is a string that uniquely identifies an account, for example: `rNsJKf3kaxvFvR8RrDi9P3LBk2Zp6VL8mp`

A Ripple ___payment___ can be sent using Ripple's native currency, XRP, directly from one account to another.  Payments can also be sent in other currencies, for example US dollars, Euros, Pounds or Bitcoins, though the process is slightly more complicated.

Payments are made between two accounts, by specifying the ___source___ and ___destination___ address for those accounts.  A payment also involves an ___amount___, which includes both the numeric amount and the currency, for example: `100+XRP`.

When you make a payment in a currency other than XRP, you also need to include the Ripple address of the ___issuer___.  The issuer is the gateway or other entity who holds the foreign-currency funds on your behalf.  For foreign-currency payments, the amount will look something like this: `100+USD+rNsJKf3kaxvFvR8RrDi9P3LBk2Zp6VL8mp`.

While the `ripple-rest` API provides a high-level interface for sending and receiving payments, there are other endpoints within the API that you can use to work with generic ripple transactions, and to check the status of the Ripple server.

### Sending Payments ###

Sending a payment involves three steps:

1. You need to create the payment object.  If the payment is to be made in a currency other than XRP, the Ripple system will identify the chain of trust, or ___path___, that connects the source and destination accounts; when creating the payment, the `ripple-rest` API will automatically find the set of possible paths for you.

2. You can modify the payment object if necessary, and then ___submit___ it to the API for processing.

3. Finally, you have to ___confirm___ that the payment has gone through by checking the payment's ___status___.

Note that when you submit a payment for processing, you have to assign a unique ___client resource ID___ to that payment.  This is a string which uniquely identifies the payment, and ensures that you do not accidentally submit the same payment twice.  You can also use the client resource ID to retrieve a payment once it has been submitted.

### Transaction Types ###

The Ripple protocol supports multiple types of transactions other than just payments. Transactions are considered to be any changes to the database made on behalf of a Ripple Address. Transactions are first constructed and then submitted to the network. After transaction processing, meta data is associated with the transaction which itemizes the resulting changes to the ledger.

+ `Payment` - Payment transactions is an authorized transfer of balance from one address to another.

+ `Trustline` - Trustline transactions is an authorized grant of trust between two addresses.

## Getting Started ##

### Setup ###

Before you can use the `ripple-rest` API, you will need to have two things:

 * An activated Ripple account.  If you don't have a Ripple account, you can use the Ripple web client to create one, as described in the <a href="https://ripple.com/wiki/Client_Manual" target="_blank">Client Manual</a>.  Make sure you have a copy of the Ripple address for your account; the address can be found by clicking on the __Receive__ tab in the web client.
 
 * The URL of the server running the `ripple-rest` API that you wish to use.  In this documentation, we will assume that the server is installed and running on a server you have connectivity to. 
 
As a programmer, you will also need to have a suitable HTTP client library that allows you to make secure HTTP (`HTTPS`) requests.  To follow the examples below, you will need to have access to the `curl` command-line tool.


### Exploring the API ###

Let's start by using `curl` to see if the `ripple-rest` API is currently running.  Type the following into a terminal window:

<a href="https://[ripple-rest-server]/v1/status" target="_blank">`curl https://[ripple-rest-server]/v1/status`</a>

After a short delay, the following response should be displayed:

```js
{
       "api_server_status": "online",
       "rippled_server_url": "wss://s_west.ripple.com:443",
       "rippled_server_status": {
         "info": {
           "build_version": "0.21.0-rc2",
           "complete_ledgers": "32570-5254146",
           "hostid": "WEAN",
           "last_close": {
             "converge_time_s": 2.022,
             "proposers": 5
           },
           "load_factor": 1,
           "peers": 52,
           "pubkey_node": "n9LVyJ9GGBwHeeZ1bwPQUKj5P6vyD5tox2ozMPadMDvXx8CrPPmJ",
           "server_state": "full",
           "validated_ledger": {
             "age": 6,
             "base_fee_xrp": 1.0e-5,
             "hash": "ADF8BEFA91F4D355C60AE37E7ED79E91591704D052114F2BBDB6AF892E5E749E",
             "reserve_base_xrp": 20,
             "reserve_inc_xrp": 5,
             "seq": 5254146
           },
           "validation_quorum": 3
         }
       },
       "api_documentation_url": "https://github.com/ripple/ripple-rest"
     }
```

#### Using the API ####

The `ripple-rest` API conforms to the following general behavior for a web interface:

* The HTTP method identifies what you are trying to do.  Generally, HTTP `GET` requests are used to retrieve information, while HTTP `POST` requests are used to make changes or submit information.

* You make HTTP (or HTTPS) requests to the API endpoint, including the desired resources within the URL itself.

* If more complicated information needs to be sent, it will be included as JSON-formatted data within the body of the HTTP POST request.

* Upon completion, the server will return an HTTP status code of 200 (OK), and a `Content-Type` value of `application/json`.  The body of the response will be a JSON-formatted object containing the information returned by the endpoint.

* The returned JSON object will include a `success` field indicating whether the request was successful or not.


### Errors ###

There are two different ways in which errors are returned by the `ripple-rest` API:

* Low-level errors are indicated by the server returning an appropriate HTTP status code.  The following status codes are currently supported:

> > Bad Request (400)  
> > Method Not Accepted (404)  
> > Gateway Timeout (502)  
> > Bad Gateway (504)

* Application-level errors are indicated by an `OK` (200) or `Accepted` (202) status code, where the body of the response is a JSON object with the following fields:

> > `success`
> > 
> > > This will be set to `false` if an error occurred.
> > 
> > `error`
> > 
> > > A short string identifying the error that occurred.
> > 
> > `message`
> > 
> > > A longer human-readable string explaining what went wrong.


### API Objects ###

#### <a id="amount_object"></a> 1. Amount ####

All currencies on the Ripple Network have issuers, except for XRP. In the case of XRP, the `issuer` field may be omitted or set to `""`. Otherwise, the `issuer` must be a valid Ripple address of the gateway that issues the currency.

For more information about XRP see  <a href="https://ripple.com/wiki/XRP" target="_blank">the Ripple wiki page on XRP</a>. For more information about using currencies other than XRP on the Ripple Network see <a href="https://ripple.com/wiki/Ripple_for_Gateways" target="_blank">the Ripple wiki page for gateways</a>.

<!-- I suggest removing the following paragraph -- it just adds confusion.  All examples show the value as a string, and users will copy this behaviour from the documentation.  Strings are always going to be 100% accurate, so we don't need to concern the user with the issue, or even the possibility of sending values as numeric values -- especially as users may not know what the "BigNumber" library is.

Original text:

Note that the `value` can either be specified as a string or a number. Internally this API uses a BigNumber library to retain higher precision if numbers are inputted as strings.

End of original text.
-->

Amount Object:

```js
{
  "value": "1.0",
  "currency": "USD",
  "issuer": "r..."
}
```

or for XRP:

```js
{
  "value": "1.0",
  "currency": "XRP",
  "issuer": ""
}
```

#### <a id="payment_object"></a> 2. Payment ####

The `Payment` object is a simplified version of the standard Ripple transaction format. 

This `Payment` format is intended to be straightforward to create and parse, from strongly or loosely typed programming languages. Once a transaction is processed and validated it also includes information about the final details of the payment.

A minimal `Payment` object will look like this:

```js
{
  "src_address": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
  "dst_address": "rNw4ozCG514KEjPs5cDrqEcdsi31Jtfm5r",
  "dst_amount": {
    "value": "0.001",
    "currency": "XRP",
    "issuer": ""
  }
}
```

where:

 + `src_address` is the Ripple address for the source account, as a string.
 
 + `dst_address` is the Ripple address for the destination account, as a string.
 
 + `dst_amount` is an [Amount](#amount_object) object representing the amount that should be deposited into the destination account.

The full set of fields accepted on `Payment` submission is as follows:

+ `src_tag` is an optional unsigned 32 bit integer (0-4294967294, inclusive) that is generally used if the sender is a hosted wallet at a gateway. This should be the same as the `dst_tag` used to identify the hosted wallet when they are receiving a payment.

+ `dst_tag` is an optional unsigned 32 bit integer (0-4294967294, inclusive) that is generally used if the receiver is a hosted wallet at a gateway.

+ `src_slippage` can be specified to give the `src_amount` a cushion and increase its chance of being processed successfully. This is helpful if the payment path changes slightly between the time when a payment options quote is given and when the payment is submitted. The `src_address` will never be charged more than `src_slippage` + the `value` specified in `src_amount`.

+ `invoice_id` is an optional 256-bit hash field that can be used to link payments to an invoice or bill.

+ `paths` is a "stringified" version of the Ripple PathSet structure. Most users of this API will want to treat this field as opaque. See the [Ripple Wiki](https://ripple.com/wiki/Payment_paths) for more information about Ripple pathfinding.

+ `flag_no_direct_ripple` is a boolean that can be set to `true` if `paths` are specified and the sender would like the Ripple Network to disregard any direct paths from the `src_address` to the `dst_address`. This may be used to take advantage of an arbitrage opportunity or by gateways wishing to issue balances from a hot wallet to a user who has mistakenly set a trustline directly to the hot wallet. Most users will not need to use this option.

+ `flag_partial_payment` is a boolean that, if set to true, indicates that this payment should go through even if the whole amount cannot be sent because of a lack of liquidity or funds in the `src_address` account. The vast majority of senders will never need to use this option.

```js
{
    /* User Specified */

    "src_address": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
    "src_tag": "",
    "src_amount": {
        "value": "0.001",
        "currency": "XRP",
        "issuer": ""
    },
    "src_slippage": "0",
    "dst_address": "rNw4ozCG514KEjPs5cDrqEcdsi31Jtfm5r",
    "dst_tag": "",
    "dst_amount": {
        "value": "0.001",
        "currency": "XRP",
        "issuer": ""
    },

    /* Advanced Options */

    "invoice_id": "",
    "paths": "[]",
    "flag_no_direct_ripple": false,
    "flag_partial_payment": false
}
```


When a payment is confirmed in the Ripple ledger, it will have additional fields added:

+ `tx_result` will be `tesSUCCESS` if the transaction was successfully processed. If it was unsuccessful but a transaction fee was claimed the code will start with `tec`. More information about transaction errors can be found on the [Ripple Wiki](https://ripple.com/wiki/Transaction_errors).

+ `tx_timestamp` is the UNIX timestamp for when the transaction was validated.

+ `tx_fee` is the network transaction fee charged for processing the transaction. For more information on fees, see the [Ripple Wiki](https://ripple.com/wiki/Transaction_fees). In the standard Ripple transaction format fees are expressed in drops, or millionths of an XRP, but for clarity the new formats introduced by this API always use the full XRP unit.

+ `tx_src_bals_dec` is an array of [`Amount`](#amount_object) objects representing all of the balance changes of the `src_address` caused by the payment. Note that this includes the `tx_fee`.

+ `tx_dst_bals_inc` is an array of [`Amount`](#amount_object) objects representing all of the balance changes of the `dst_address` caused by the payment

```js
{
    /* ... */

    /* Generated After Validation */

    "tx_direction": "outgoing",
    "tx_state": "confirmed",
    "tx_result": "tesSUCCESS",
    "tx_ledger": 4696959,
    "tx_hash": "55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7",
    "tx_timestamp": 1391025100000,
    "tx_timestamp_human": "2014-01-29T19:51:40.000Z",
    "tx_fee": "0.000012",
    "tx_src_bals_dec": [{
        "value": "-0.001012",
        "currency": "XRP",
        "issuer": ""
    }],
    "tx_dst_bals_inc": [{
        "value": "0.001",
        "currency": "XRP",
        "issuer": ""
    }]
}
```

#### 3. Notification ####

Notifications are new type of object not used elsewhere on the Ripple Network but intended to simplify the process of monitoring accounts for new activity.

If there is a new `Notification` for an account it will contain information about the type of transaction that affected the account, as well as a link to the full details of the transaction and a link to get the next notification. 

If there is a new `notification` for an account, it will come in this format:

+ `timestamp` is the UNIX timestamp for when the transaction was validated, or the number of milliseconds since January 1st, 1970 (00:00 UTC)
+ `timestamp_human` is the transaction validation time represented in the format `YYYY-MM-DDTHH:mm:ss.sssZ`. The timezone is always UTC as denoted by the suffix "Z"
+ `transaction_url` is a URL that can be queried to retrieve the full details of the transaction. If it the transaction is a payment it will be returned in the `Payment` object format, otherwise it will be returned in the standard Ripple transaction format
+ `next_notification_url` is a URL that can be queried to get the notification following this one for the given address
+ `source_transaction_id` will be the same as the `source_transaction_id` originally submitted by the sender. Senders should look for the `source_transaction_id`'s of payments they have submitted to `ripple-rest` amongst `Notification`s of validated payments. If the `source_transaction_id` of a particular payment appears in a `Notification` with the `state` listed as `validated`, then that payment has been successfully written into the Ripple Ledger

```js
{
  "address": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
  "type": "payment",
  "tx_direction": "outgoing",
  "tx_state": "confirmed",
  "tx_result": "tesSUCCESS",
  "tx_ledger": 4696959,
  "tx_hash": "55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7",
  "tx_timestamp": 1391025100000,
  "tx_timestamp_human": "2014-01-29T19:51:40.000Z",
 "tx_url":"http://api/v1/addresses/r../payments/55B..",
 "next_notification_url":"http://api/v1/addresses/r../next_notification/5.."
  "confirmation_token": "55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7"
}
```

If there are no new notifications, the empty `Notification` object will be returned in this format:

+ `type` will be `none` if there are no new notifications
+ `tx_state` will be `pending` if there are still transactions waiting to clear and `empty` otherwise
+ `next_notification_url` will be provided whether there are new notifications or not so that that field can always be used to query the API for new notifications.

```js
{
  "address": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
  "type": "none",
  "tx_direction": "",
  "tx_state": "empty",
  "tx_result": "",
  "tx_ledger": "",
  "tx_hash": "",
  "tx_timestamp": "",
  "tx_timestamp_human": "",
  "tx_url": "",
  "next_notification_url": "http://api/v1/addresses/r../next_notification/5..",
  "confirmation_token": ""
}
```

# PAYMENTS #

`ripple-rest` provides access to `ripple-lib`'s robust transaction submission processes. This means that it will set the fee, manage the transaction sequence numbers, sign the transaction with your secret, and resubmit the transaction up to 10 times if `rippled` reports an initial error that can be solved automatically.

## Making Payments ##

### Preparing a Payment ###

> __`GET /v1/accounts/:account/payments/paths/:dst_account/:dst_amount`__

To prepare a payment, you first make an HTTP `GET` call to the above endpoint.  This will generate a list of possible payments between the two parties for the desired amount, taking into account the established trustlines between the two parties for the currency being transferred.  You can then choose one of the returned payments, modify it if necessary (for example, to set slippage values or tags), and then submit the payment for processing.

The following parameters are required by this API endpoint:

> `account`
> 
> > The Ripple address for the source account.
> 
> `dst_account`
> 
> > The Ripple address for the destination account.
> 
> `dst_amount`
> 
> > The amount to be sent to the destination account.  Note that this value uses `+` characters to separate the `value`, `currency` and `issuer` fields.  For XRP, the format is:
> > 
> > > `0.1+XRP`
> > 
> > For other currencies, you need to include the Ripple address of the currency's issuer, like this:  
> > 
> > > `0.1+USD+r...`

Optionally, you can also include the following as a query string parameter:

> `source_currencies`
> 
> > A comma-separated list of source currencies.  This is used to filter the returned list of possible payments.  Each source currency can be specified either as a currency code (eg, `USD`), or as a currency code and issuer (eg, `USD+r...`).  If the issuer is not specified for a currency other than XRP, then the results will be limited to the specified currency, but any issuer for that currency will be included in the results.

Note that this call is a wrapper around the [Ripple path-find](https://ripple.com/wiki/RPC_API#path_find) command, and returns an array of [`Payment`](#payment_object) objects, like this:

```js
{
  "success": true,
  "payments": [
    { /* Payment */ },
    { /* Payment */ },
    ...
  ]
}
```
You can then select the desired payment, modify it if necessary, and submit the payment object to the [`POST /v1/payments`](#submitting-a-payment) endpoint for processing.

__NOTE:__ This command may be quite slow. If the command times out, please try it again.

### Submitting a Payment ###

> __`POST /v1/payments`__

Before you can submit a payment, you will need to have three pieces of information:

> * The [`Payment`](#payment_object) object to be submitted.
> 
> * The ___secret___, or private key for your Ripple account.
>
> > > __DO NOT SUBMIT YOUR SECRET TO AN UNTRUSTED REST API SERVER__ -- this is the key to your account and your money. If you are using the test server provided, only use test accounts to submit payments.
> 
> * A ___client resource ID___ that will uniquely identify this payment.  This is a 36-character UUID (universally unique identifier) value which will uniquely identify this payment within the `ripple-rest` API.  Note that you can use the [`GET /v1/uuid`](#calculating_a_uuid) endpoint to calculate a UUID value if you do not have a UUID generator readily available.

This HTTP `POST` request must have a content-type of `application/json`, and the body of the request should look like this:

```js
{
  "secret": "s...",
  "client_resource_id": "...",
  "payment": { /* Payment */ }
}
```

Upon completion, the server will return a JSON object which looks like the following:

```js
{
  "success": true,
  "client_resource_id": "f2f811b7-dc3b-4078-a2c2-e4ca9e453981",
  "status_url": ".../v1/accounts/r1.../payments/f2f811b7-dc3b-4078-a2c2-e4ca9e453981"
}
```

The `status_url` value is a URL that can be queried to get the current status for this payment.  This will be a reference to the `GET /v1/accounts/{account}/payments` endpoint, with the client resource ID filled in to retrieve the details of the payment.  More information on this endpoint can be found in the section on [confirming a payment](#confirming-a-payment).

If an error occurred that prevented the payment from being submitted, the response object will look like this:

```js
{
  "success": false,
  "error": "tecPATH_DRY",
  "message": "Path could not send partial amount. Please ensure that the src_address has sufficient funds (in the src_amount currency, if specified) to execute this transaction."
}
```

More information about transaction errors can be found on the [Ripple Wiki](https://ripple.com/wiki/Transaction_errors).

Note that payments cannot be cancelled once they have been submitted.

### Confirming a Payment ###

> __`GET /v1/accounts/:account/payments/:transaction_id`__

To confirm that your payment has been submitted successfully, you can call this API endpoint.  The `:transaction_id` value can either be the transaction hash for the desired payment, or the payment's client resource ID.

The server will return the details of your payment:

```js
{
  "success": true,
  "payment": {
    /* Payment */
  }
}
```

You can then check the `state` field to see if the payment has gone through; it will have the value "validated" when the payment has been validated and written to the Ripple ledger.

If the payment cannot be found, then an error will be returned instead:

```js
{
  "success": true,
  "error": "Payment Not Found",
  "message": "This may indicate that the payment was never validated and written into the Ripple ledger and it was not submitted through this ripple-rest instance. This error may also be seen if the databases of either ripple-rest or rippled were recently created or deleted."
}
```

Note that there can be a delay in processing a submitted payment; if the payment does not exist yet, or has not been validated, you should wait for a short period of time before checking again.

## Receiving Payments ##

Applications will not only need to submit payments but monitor all incoming transactions for a desired Ripple address.

> ___NOTE___: The API currently seems to be broken here -- you can only retrieve notifications for a single transaction hash or client resource ID, not across all transactions for an account.  I have commented this section out until we can figure out how to receive notifications across all transactions for an account.

<!--

1. Checking the most recent notification
2. Checking the next notification

## Checking Notifications ##

### Most recent notification ###

#### `GET /api/v1/addresses/:address/next_notification` ####

Use this to retrieve the most recent notification on the account:

To find out more information about that payment follow the link at `tx_url`. Otherwise follow the `next_notification_url` and check for the next notification.

If notifications are being retrieved from a `rippled` server that does not have a full historical database, the response may have serveral blank fields.

```js
{
  "success": true,
  "notification": {
    "address": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
    "type": "payment",
    "tx_direction": "incoming",
    "tx_state": "confirmed",
    "tx_result": "tesSUCCESS",
    "tx_ledger": 4716034,
    "tx_hash": "EC19E24AA51D39E809597A5DCF3A7E253F98C27FE3287CB919319A5C59AD8302",
    "tx_timestamp": 1391130630000,
    "tx_timestamp_human": "2014-01-31T01:10:30.000Z",
    "tx_url": "http://api/v1/addresses/rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz/payments/EC19E24AA51D39E809597A5DCF3A7E253F98C27FE3287CB919319A5C59AD8302?ledger=4716034",
    "next_notification_url": "http://ripple-rest.herokuapp.com:49598/api/v1/addresses/rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz/next_notification/EC19E24AA51D39E809597A5DCF3A7E253F98C27FE3287CB919319A5C59AD8302?ledger=4716034"
    "confirmation_token": ""
  }
}
```

### Checking next notification ###

#### `GET /api/v1/addresses/:address/next_notification/:prev_tx_hash` ####

By checking for the most recent notification above, take the hash from the most recent notification to monitor if another notification has arrived. If there is no notifications newer than the most recent, than you will receive the notification object with:

`"type": "none"`
`"tx_state": "empty"`

Because the `type` is `none` and the `tx_state` is `empty`, that means there is no next notification (yet) and there are no transactions pending in the outgoing queue. A `tx_state` of `pending` would indicate that there are still transactions waiting for confirmation.

If there is a newer notification than the one you are checking on, than the response will contain a new notification object.

-->

# ACCOUNTS #

## Payment History ##
<span></span>
> __`GET /v1/accounts/:account/payments`__

This API endpoint can be used to browse through an account's payment history.  The following query string parameters can be used to filter the list of returned payments:

> `source_account`
> 
> > Filter the results to only include payments sent by the given account.
> 
> `destination_account`
> 
> > Filter the results to only include payments received by the given account.
> 
> `exclude_failed`
> 
> > If set to `true`, the results will only include payments which were successfully validated and written into the ledger.  Otherwise, failed payments will be included.
> 
> `earliest_first`
> 
> > If set to `true`, the payments will be returned in ascending date order.  Otherwise, the payments will be returned in descending date order (ie, the most recent payment will be returned first).  Defaults to `true`.
> 
> `start_ledger`
> 
> > The index for the starting ledger.  If `earliest_first` is `true`, this will be the oldest ledger to be queried; otherwise, it will be the most recent ledger.  Defaults to the first ledger in the `rippled` server's database.
> 
> `end_ledger`
> 
> > The index for the ending ledger.  If `earliest_first` is `true`, this will be the most recent ledger to be queried; otherwise, it will be the oldest ledger.  Defaults to the most recent ledger in the `rippled` server's database.
> 
> `results_per_page`
> 
> > The maximum number of payments to be returned at once.  Defaults to 20.
> 
> `page`
> 
> > The page number to be returned.  The first page of results will have page number `1`, the second page will have page number `2`, and so on.  Defaults to `1`.

Upon completion, the server will return a JSON object which looks like the following:

```js
{
  "success": true,
  "payments": [
    {
      "client_resource_id": "3492375b-d4d0-42db-9a80-a6a82925ccd5",
      "payment": {
        /* Payment */
      }
    }, {
      "client_resource_id": "4a4e3fa5-d81e-4786-8383-7164c3cc9b01",
      "payment": {
        /* Payment */
      }
    }
  ]
}
```

If the server returns fewer than `results_per_page` payments, then there are no more pages of results to be returned.  Otherwise, increment the page number and re-issue the query to get the next page of results.

Note that the `ripple-rest` API has to retrieve the full list of payments from the server and then filter them before returning them back to the caller.  This means that there is no speed advantage to specifying more filter values.

## Account Balances ##
<span></span>
> __`GET /v1/accounts/:account/balances`__

Retrieve the current balances for the given Ripple account.

The `:account` parameter should be set to the Ripple address of the desired account.  The server will return a JSON object which looks like the following:

```js
{
  "success": true,
  "balances": [
    {
      "currency": "XRP",
      "amount": "1046.29877312",
      "issuer": ""
    },
    {
      "currency": "USD",
      "amount": "512.79",
      "issuer": "r...",
    }
    ...  
  ]
}
```

There will be one entry in the `balances` array for the account's XRP balance, and additional entries for each combination of currency code and issuer.

## Account Settings ##

You can retrieve an account's settings by using the following endpoint:

> __`GET /v1/accounts/:account/settings`__

The server will return a list of the current settings in force for the given account, in the form of a JSON object:

```js
{
  "success": true,
  "settings": {
    /* settings */
  }
}
```

The following account settings are currently supported:

> `PasswordSpent`
> 
> > `true` if the password has been "spent", else `false`.
> > 
> > > NOTE: This is not currently listed in the account settings schema, so I'm not sure what this setting is used for.
> 
> `RequireDestTag`
>
> > If this is set to `true`, incoming payments will only be validated if they include a `destination_tag` value.  Note that this is used primarily by gateways that operate exclusively with hosted wallets.
> 
> `RequireAuth`
> 
> > If this is set to `true`, incoming trustlines will only be validated if this account first creates a trustline to the counterparty with the authorized flag set to true.  This may be used by gateways to prevent accounts unknown to them from holding currencies they issue.
> 
> `DisallowXRP`
> 
> > If this is set to `true`, payments in XRP will not be allowed.
> 
> `DisableMaster`
> 
> > This is not currently documented.
> 
> `EmailHash`
> 
> > The MD5 128-bit hash of the account owner's email address, if known.
> 
> `WalletLocator`
> 
> > This is not currently documented.
> 
> `WalletSize`
> 
> > This is not currently documented.
> 
> `MessageKey`
> 
> > An optional public key, represented as a hex string, that can be used to allow others to send encrypted messages to the account owner.
> 
> `Domain`
> 
> > The domain name associated with this account.
> 
> `TransferRate`
> 
> > The rate charged each time a holder of currency issued by this account transfers some funds.  The default rate is `"1.0"; a rate of `"1.01"` is a 1% charge on top of the amount being transferred.  Up to nine decimal places are supported.
> 
> `Signers`
> 
> > This is not currently documented.

To change an account's settings, make an HTTP `POST` request to the above endpoint.  The request must have a content-type of `application/json`, and the body of the request should look like this:

```js
{
  "account": "r...",
  "secret": "s...",
  /* Settings */
}
```

The given settings will be updated.

# RIPPLED SERVER STATUS #

The following two endpoints can be used to check if the `ripple-rest` API is currently connected to a `rippled` server, and to retrieve information about the current status of the API.

## Check Connection State ##
<span></span>
> __`GET /v1/server/connected`__

Checks to see if the `ripple-rest` API is currently connected to a `rippled` server, and is ready to be used.  This provides a quick and easy way to check to see if the API is up and running, before attempting to process transactions.

No additional parameters are required.  Upon completion, the server will return `true` if the API is connected, and `false` otherwise.

## Get Server Status ##
<span></span>
> __`GET /v1/server`__

Retrieve information about the current status of the `ripple-rest` API and the `rippled` server it is connected to.

This endpoint takes no parameters, and returns a JSON object with information on the current status of the API:

```js
{
  "api_server_status": "online",
  "rippled_server_url": "wss://s_west.ripple.com:443",
  "rippled_server_status": {
    "info": {
      "build_version": "0.21.0-rc2",
      "complete_ledgers": "32570-4805506",
      "hostid": "BUSH",
      "last_close": {
        "converge_time_s": 2.011,
        "proposers": 5
      },
      "load_factor": 1,
      "peers": 51,
      "pubkey_node": "n9KNUUntNaDqvMVMKZLPHhGaWZDnx7soeUiHjeQE8ejR45DmHyfx",
      "server_state": "full",
      "validated_ledger": {
        "age": 2,
        "base_fee_xrp": 0.00001,
        "hash": "2B79CECB06A500A2FB92F4FB610D33A20CF8D7FB39F2C2C7C3A6BD0D75A1884A",
        "reserve_base_xrp": 20,
        "reserve_inc_xrp": 5,
        "seq": 4805506
      },
      "validation_quorum": 3
    }
  },
  "api_documentation_url": "https://github.com/ripple/ripple-rest"
}
```

If the server is not currently connected to the Ripple network, the following error will be returned:

```js
{
  "success": false,
  "error": "rippled Disconnected",
  "message": "ripple-rest is unable to connect to the specified rippled server, or the rippled server is unable to communicate with the rest of the Ripple Network. Please check your internet and rippled server settings and try again"
}
```

# UTILITIES #

## Retrieve Ripple Transaction ##

While the `ripple-rest` API is a high-level API built on top of the `rippled` server, there are times when you may need to access an underlying Ripple transaction rather than dealing with the `ripple-rest` API directly.  When you need to do this, you can retrieve the standard Ripple transaction by using the following endpoint:

> __`GET /v1/transactions/:tx_hash`__

This retrieves the underlying Ripple transaction with the given transaction hash value.  Upon completion, the server will return following JSON object:

```js
{
  "success": true,
  "tx": { /* Ripple Transaction */ }
}
```

Please refer to the [Transaction Format](https://ripple.com/wiki/Transactions) page in the Ripple Wiki for more information about Ripple transactions.

If the given transaction could not be found in the `rippled` server's historical database, the following error will be returned:

```js
{
  "success": false,
  "error": "txnNotFound",
  "message": "Transaction not found."
}
```


## Create Client Resource ID ##
<span></span>
> __`GET /v1/uuid`__

This endpoint creates a universally unique identifier (UUID) value which can be used to calculate a client resource ID for a payment.  This can be useful if the application does not have a UUID generator handy.

This API endpoint takes no parameters, and returns a JSON object which looks like the following:

```js
{
  "success": true,
  "uuid": "xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx"
}
```


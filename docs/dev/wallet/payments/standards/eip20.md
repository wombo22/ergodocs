# ErgoPay: an interaction protocol between wallet application and dApp

> 🔗 From [EIP-0020](https://raw.githubusercontent.com/ergoplatform/eips/master/eip-0020.md)


* Author: @aslesarenko, @MrStahlfelge
* Status: Proposed
* Created: 18-August-2021
* License: CC0
* Forking: not needed 

## Contents
- [ErgoPay: an interaction protocol between wallet application and dApp](#ergopay-an-interaction-protocol-between-wallet-application-and-dapp)
  - [Contents](#contents)
  - [Description](#description)
  - [Background And Motivation](#background-and-motivation)
  - [ErgoPay Interaction Protocol](#ergopay-interaction-protocol)
  - [Data Formats](#data-formats)
    - [Option 1: dApp providing URL request to fetch ErgoPaySigningRequest information from](#option-1-dapp-providing-url-request-to-fetch-ergopaysigningrequest-information-from)
      - [Response: ErgoPaySigningRequest](#response-ergopaysigningrequest)
    - [Option 2: dApp providing URI scheme containing ReducedTransaction](#option-2-dapp-providing-uri-scheme-containing-reducedtransaction)
  - [Implementation in Wallet app](#implementation-in-wallet-app)
  - [Implementation in dApp](#implementation-in-dapp)
  - [Benefits for dApps](#benefits-for-dapps)
  - [Benefits for Wallets](#benefits-for-wallets)

## Description 
This EIP defines a standard for cross-platform interaction between an online dApp
and a wallet app for creating, signing and sending Ergo transactions.

## Background And Motivation

Cryptocurrency wallets, (like [Ergo Android Wallet](https://github.com/ergoplatform/ergo-wallet-android)) typically
support scanning QR codes of [Payment
Request](https://explorer.ergoplatform.com/payment-request?address=9er9hxmVcL8S4bNypCyJHpPEEkAfEd9CLq5gNrHN6s2pYomp55N&amount=0&description=) (on mobile devices) or intercepting a link with a special [URI scheme](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki).
After parsing a payment request the wallet can build a new transaction, sign it
using a secret key stored locally on the device and then send to the blockchain.

However, in general, this only can be done for _simple_ transactions like
transferring ERGs and assets between Pay-To-Public-Key addresses or transactions
which only spend boxes from P2PK addresses.

The reason for this is simple. In Ergo's eUTXO model a box can be protected by
an arbitrarily complex contract (aka spending condition) and any spending
transaction should satisfy that condition by adding required context variables,
creating expected number of outputs with specific registers depending on the
contract. There is no way for a universal wallet application to know all
the specific details of all the possible contracts.

On the contrary, every Ergo dApp (think of website) is usually built on top of
specific contracts. Business logic of the dApp includes spending of the boxes
protected by those application-specific contracts.
In other words, any dApp is in full control of creating transactions and thus
can spend its own contracts without above mentioned problems. However, the dApp
cannot sign those transactions, because signing requires knowledge of private keys
which are not stored on the dApp, but instead stored on the wallet app.

Thus, interaction between a dApp and a wallet is required such that:
1) the dApp builds a transaction and makes it available for the wallet application
2) wallet app shows a confirmation screen to the user, displaying the inboxes and 
outboxes it is going to sign
3) when user confirms, wallet app signs the transaction and submits it to the blockchain or
returns back to the dApp
4) dApp monitors the transaction on the blockchain and upon confirmations
proceeds with its business logic.

One possible implementation of such an interaction scheme is described in
[EIP-0012](https://github.com/ergoplatform/eips/pull/23). It, however, only
suitable for in-browser wallets, which can interact with dApps via javascript
code injected into a web context.

Luckily, the design of Ergo contracts allows for a simple and universal
implementation suitable for mobile or desktop wallets. This EIP describes the interaction
protocol and the reference implementation.

In this 3 step process the main question is: 
"How the transaction built in dApp can be transferred to the wallet
application so that it can be signed there?".

## ErgoPay Interaction Protocol

An ErgoPay interaction between Wallet, and dApp is driven by dApp's
user and proceeds as the following:

1) The user enters the necessary information in the dApp's UI and proceeds to a
payment screen. Optionally, dApp can request user's P2PK address with an extra 
step using a signing request with a placeholder URL (see below).

2) The payment screen shows transaction details and a QR code as well as a clickable 
link.

3) User clicks the link to hand the information over to a wallet application on the 
same device, or scans the QR code using a wallet application.

4) The Hot Wallet application parses the QR code data and obtains either
`ErgoPaySigningRequest` or `ReducedTransaction` data (see [Data Formats](#data-formats) section). In the former case the 
QR code data contains an url to download `ErgoPaySigningRequest` from the dApp.

5) When `ErgoPaySigningRequest` or `ReducedTransaction` is obtained, it is shown as a payment screen on the
wallet app containing the same transaction details as the dApp screen.

6) The user compares the dApp's screen, the Wallet's screen and the
transaction details and confirms the payment by using a "Sign" button. If the Hot
Wallet also supports [EIP-0019](eip19.md) and private keys are not available,
then the sign button behaves like a "Cold Sign" button according to EIP-0019. 

7) The wallet application signs the transaction either using local private keys
or using Cold Wallet and EIP-0019 protocol. The result of signing is
`SignedTransaction` data.

8) The Hot Wallet obtains the transaction id and sends it to the dApp using
`ErgoPayTransactionSent` API post message to `replyToUrl` if the url is provided, if
successful the wallet then submits `SignedTransaction` to the blockchain. If the url is
not provided, then the wallet submits the transaction to blockchain without notifying the
dApp. The dApp can monitor the blockchain by txId which it can learn from
UnsignedTransaction. This is based on the fact that `UnsignedTransaction.id ==
SignedTransaction.id` holds for all transactions.

10) dApp monitors the transaction by id and proceeds with its business logic upon
receiving enough confirmations. This concludes the ErgoPay protocol.

Additional requirements: 
- many instances of ErgoPay protocol MUST be able to run simultaneously within
the dApp
- the Hot Wallet MAY implement "one-at-a-time" signing. When the signing
is interrupted for some reason it can be started from the step 3) (i.e. scanning
the QR)
- when dApp receives the transaction id, but the `SignedTransaction` is not
accepted to the blockchain then the dApp SHOULD detect this situation and stop
monitoring that transaction.

## Data Formats

The data formats of this EIP are based on a new binary data structure and
serialization format called `ReducedTransaction` which is described in
[EIP-0019](eip19.md).

Below we describe the data formats specified by this EIP and refer to the
formats defined in EIP-0019.

Wallet apps should be able to initiate ErgoPay both by using URI schemes 
(clickable links) or QR codes.

### Option 1: dApp providing URL request to fetch ErgoPaySigningRequest information from

`ergopay://<URL>`

URL is provided without the https prefix. http communication is not allowed except for IP addresses 
(in order to test within a local network).

The provided URL can contain a `#P2PK_ADDRESS#` placeholder. The wallet application will replace this 
placeholder with an actual P2PK address.

Examples:
* `ergopay://sigmausd.io/signingRequest/2001-16b8-66c4-b800-6e52-8ce4` will make the wallet app request
`https://sigmausd.io/signingRequest/2001-16b8-66c4-b800-6e52-8ce4`
* `ergopay://192.168.0.1/html` will make the wallet app request
`http://192.168.0.1/html`
* `ergopay://auctionhouse.org/bid/#P2PK_ADDRESS#` will make the wallet app request `ergopay://auctionhouse.org/bid/3Wx7Z8FywaL4ofunDCV1NaTJX5CpjTubMrjpCkEhnNBAgJLGfRcD`

#### Response: ErgoPaySigningRequest

The wallet application should request URL and obtain the following data (json format)

```
ErgoPaySigningRequest:
  - transaction: ReducedTransaction (optional*)
  - address: String (optional)
  - message: String (optional*)
  - messageSeverity: String (optional) "INFORMATION", "WARNING", "ERROR"
  - replyToUrl: String (optional)
```

Either a **transaction** or a **message** must be provided, otherwise the request is invalid.

The wallet application should show the **message** and display the **messageSeverity** in a suitable 
way, if provided.

If **transaction** details are obtained, a payment screen opens for the user to confirm 
signing the transaction. Transaction details CAN BE OPTIONALLY shown on the payment
screen of the wallet application. 

If **address** is provided by the dApp, the wallet can preselect the key the user
needs to sign the transaction.

After signing is performed and the `SignedTransaction` data is obtained, the 
wallet can POST the following data to the dApp using **replyToUrl** from the
signing request (json format) if it is provided. 

```
ErgoPayTransactionId:
  - txId: String
```

dApps should not rely on this request to be made. It could happen that the transaction was submitted to a node, but the reply couldn't be executed. dApps know the transaction id and should monitor the mempool and blockchain on their own.

In case no **transaction** was provided, the wallet app displays the **message** that should inform the user about further steps needed.


### Option 2: dApp providing URI scheme containing ReducedTransaction

When dapp does not need to use an extra request, the `ReducedTransaction` could 
also be encoded in the QR code or link:

`ergopay:<ReducedTransaction, base 64 url safe encoded>`

It is not possible to provide **description**, **address**, **message** and **replyToUrl** in this simpler interchange format.

dApp developers should keep in mind that there are length restrictions for URI schemes
and QR codes. Both should be able handle up to 2900 chars, but QR codes with a lot of 
content need to be shown bigger to be read without problems.

We recommend to use an URL request if the payload exceeds 400 chars.


## Implementation in Wallet app
Ergo Wallet App 1.6 or higher https://github.com/ergoplatform/ergo-wallet-app

## Implementation in dApp
ErgoPay showcase example https://github.com/MrStahlfelge/ergopay-server-example

## Benefits for dApps
ErgoPay provides a fast, easy, and secure way for users to buy goods and
services in a dApp or on a website. When supported, ErgoPay can substantially
increase checkout conversion rates, user loyalty and purchase frequency, and
reduced checkout time. 

- dApp or website don’t need to handle user's secrets (mnemonic/private keys).
Instead, once the user has signed the transaction to confirm purchase intent,
your app or website receives a transaction id to monitor payment status on the
blockchain.
- dApp's users don't need to worry about security of their private keys as the
 wallet app guarantees they never leave the device
- ErgoPay EIP is compatible with Cold Wallet EIP, thus users can use Cold Wallet
devices to sign transaction within ErgoPay signing process.
- adding ErgoPay to product detail pages, the cart, checkout page, in payment
settings, or anywhere else a user can choose ErgoPay as the payment method or
initiate a purchase.
- The payment screen can be presented immediately after the user taps the Ergo
Pay button, without any interim screens or pop-ups except to prompt for
necessary product details, such as size or quantity.
- ErgoPay is simple and universal. It supports all smart contracts and offers
the flexibility to implement simple to complex dApps.

## Benefits for Wallets

Any wallet application team should consider supporting ErgoPay in their wallet along
with basic wallet features. Users can participate in Ergo dApps and
wallet team can receive service fees from those transactions. 

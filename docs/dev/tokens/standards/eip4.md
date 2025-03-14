# EIP-0004: Asset Standard

> 🔗 From [EIP-0004:](https://github.com/ergoplatform/eips/blob/master/eip-0004.md)


This standard provides a uniform way to issue Ergo tokens.

A standard interface allows any tokens on Ergo to be re-used by other applications: from wallets to decentralized exchanges.


## Ergo tokens background

Ergo supports **custom tokens as first-class citizens.**

- Namely, a special register (R2) of a box contains (tokenId -> amount) pairs.
- A box can contain at most 255 secondary tokens. 

There are also indirect limits: 

- A box can be no larger than 4 kilobytes
- Tokens add to the computational cost of the transaction.

A transaction can create *out-of-thin-air* tokens in its outputs if the token identifier is equal to the identifier of the first input box of the transaction.

- As the box identifier is cryptographically unique, there is no chance of having the second token with the same identifier while the hash function is collision-resistant.
- This rule also means that only one new token per transaction can be created.
- Unlike ergs, other tokens can be burnt: the total amount for a token in transaction inputs should be no less than the amount of the outputs.

[Storage rent](rent.md) component allows a miner to collect (or burn) all the tokens inside a box if it is expired, and there are not enough Ergs in the box to pay the storage rent fee.

## Ergo tokens standard

Though this is not required by the protocol, we propose the following structure for the box that issues a token:

| Register        | Description                                     | Example                      |Encoded                      |
| --------------- |:-----------------------------------------------:| ----------------------------:|----------------------------:|
| R2              | Token id and amount pair                        | [("7d...09", 100000)]        |                             |
| R4              | Token verbose name (UTF-8 representation)       | "USD"                        | "0e03555344"                |
| R5              | Token description (UTF-8 representation)        | "Nothing backed USD token"   | "0e184e6f7468696e67206261636b65642055534420746f6b656e"  |
| R6              | Number of decimals                              | "2"                          | "0e0132"                         |

Note, that additional registers (R4-R6) are encoded as **Coll[Byte]** type of ErgoScript and their encoded representation is received as `'\x0e' + intToVlq(byteArray.length) + byteArray` where `byteArray` is UTF-8 representation of the original string.
The example above issues one thousand tokens called "USD" with two decimals each.
The transaction that issues such a token was included in block 98288 and may be found in block [explorer](https://explorer.ergoplatform.com/en/transactions/5c131f8ae9fa68dab1bac654aa66d364bc7da12107f337a0c9d3d80d8951ee41))

## Ergo asset types

This standard is an extension of [token standard](#ergo-tokens-standard):

In the asset type standard, R7 is a required two-byte value encoded as Coll[Byte]. The first byte represents the asset category, for example, _NFT_ or _Share Tokens_. The second byte specifies the exact subcategory, for example, _Picture Artwork NFT_ or _ErgoMixer's Share Tokens_. The second byte can be omited so that the issuance only specifies the catagory and not the subcatagory, for example, _Share Tokens_.
Also, the remaining R8 and R9 registers can be used by each individual asset types based on their needs.

The standardization of various asset types can be found below:

| Asset type        | R7                                     | R8                      |R9                      |
| --------------- |:-----------------------------------------------:| ----------------------------:|----------------------------:|
| NFT - picture artwork              | [0x01, 0x01] - i.e., "0e020101"                        | SHA256 hash of the picture    | Optional - link to the artwork (UTF-8 representation) |
| NFT - audio artwork              | [0x01, 0x02] - i.e., "0e020102"                        | SHA256 hash of the audio    | Optional - link to the audio encoded as Coll[Byte] or (link to the audio, link to the image cover) encoded as (Coll[Byte], Coll[Byte]) (UTF-8 representation) |
| NFT - video artwork              | [0x01, 0x03] - i.e., "0e020103"                        | SHA256 hash of the video    | Optional - link to the video (UTF-8 representation) |
| [Membership token - threshold signature](https://www.ergoforum.org/t/a-simpler-collective-spending-approach-for-everyone/476)              | [0x02, 0x01] - i.e., "0e020201"                        | Number of required signatures (Integer) - i.e., 4 in case of 4-of-10 threshold signature   | Deposit address of the funds controlled by the threshold signature (Ergo tree byte array) |

The above registers (R7-R9) are also encoded as Coll[Byte] type unless stated otherwise.

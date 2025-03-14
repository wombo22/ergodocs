# EIP-0024: Artwork Contract

> 🔗 From [EIP-0024:](https://github.com/ergoplatform/eips/blob/master/eip-0024.md)


## Motivation 
With the discovery that we can easily use spent boxes in contracts ([see here](https://www.ergoforum.org/t/ergoscript-design-patterns/222/23?u=anon_real)), some new features are introduced for artworks and can be extended further in the future. This EIP is an extension of [Asset [standard](eip4.md) and tends to standardize artwork issuance in particular.

## Design V1

There are two important boxes in the issuance process of an artwork.

- The issuance box, which follows [Asset standard](eip-0004.md) depending on the type of artwork.
- The issuer box, which is the first input of the transaction that is issuing the artwork. In particular, the box with the same ID as the artwork's token ID.

Now we will discuss why the issuer box is important and define a standard for it.

The issuer box is important because it has the same ID as the artwork's token ID and we can include it (although it is spent) in our transactions either in registers or as context data. This means that we can prove/verify certain features that an artwork may have using this box. The following is the v1 standard for this box - which may be extended in the future (refer to Design V2 part).

- `R4` of this box is an `Int`, showing 1000 * royalty percentage of the artwork. e.g, 20 for 2% royalty. If `R4` of the issuer box of an artowrk is empty or non-positive, then royalty is considered to be 0%.
- In the current design of the [Auction contranct](eip-0022.md), proposition bytes of this box is the contract that the royalty percentage will be sent to. This is a suboptimal design because of AOT costing mechanism which is supposed to be replaced with JIT costing with a soft-fork that will happen with v5.0 of [ergo node](https://github.com/ergoplatform/ergo).
  
As mentioned, royalty amount will go to the propositiona bytes of the issuer box. In the case of a simple proxy contract (proposition bytes of the issuer box), this means that the artist will receive the royalty share. However, the proxy contract may enforce the royalty to go to any other complex contract - e.g., 20% to the artist, and 80% to some charity.

## Design V2
Version 2 of this proposal is created to both extend this proposal to include collections and fix some poor design choices that were caused by the AOT costing issues. The following is the v2 standard for the issuer box.

- `R4` is an integer showing the artwork standard version. If This register is empty, the type is something other than `Int`, or the number is greater or equal to 100 then it should be considered as part of V1 design. Otherwise, this register shows the version number for the standard, i.e., 2 for Design V2.
- `R5` is of type `Coll[(Coll[Byte], Int)]` that encodes information about royalty recipients and percentages. In particular, a list of royalty recipients and their respective percentage. The `Coll[Byte]` part is the ergo tree of one of the royalty recipients and the `Int` part shows 1000 * royalty percentage of this recipient (e.g. 50 if the receipient receives 5% of the sale). The total royalty percentage of the artwork is the sum of percentages of this list. If the list is empty, the artwork's royalty is 0%.
Royalty amount must go to royalty recipients based on the explained share of recipients. An important note here is that it is the responsibility of the auction contract to follow the appropriate version of this standard and consequently send the royalty amount to appropriate recipient(s). 

- `R6` contains three types of traits:
  - Properties: These are textual traits such as specifying `sex` as `male`.
  - Levels: These are numerical traits that encode some sort of level-like information such as specifying `speed` as `60 out of 100`.
  - Stats: These are numerical traits that encode any numerical information about the NFT such as specifying `age` as `25 out of 50`.

Although the structure of levels and stats are similar, they can be considered different in marketplaces both for UI/UX purposes and also for filtering purposes.

The following specifies the structure for traits.
```scala
( // properties
  Coll[(  
    Coll[Byte],  // key
    Coll[Byte]   // value
  )],
  ( // levels
    Coll[(
      Coll[Byte],  // key
      (Int, Int)   // value, max
    )],
    // stats
    Coll[(
      Coll[Byte],  // key
      (Int, Int)   // value, max
    )]
  )
)
```
If a list is empty(e.g., there is no `levels`) then the corresponding `Coll` should be empty. Keys are case-insensitive which means for example, marketplaces should treat "KeY" the same as "key". Moreover, keys should be consistent across different artworks of a specific collection. For example, if "key1" is a property trait, it should always be used as a property trait in all artworks of a specific collection.
All `Coll[Byte]` types in the above structure are UTF-8 text encoded as `Coll[Byte]`.

- `R7` of this box contains token ID of the collection as `Coll[Byte]` otherwise it is an empty `Coll[Byte]`. This is needed for two cases:
  - if the issuer box contains more than one collection token, this will specify which one is the collection that the artist intended to put the artwork in.
  - if the issue box contains some collection tokens but the artist does not intend to put the artwork in any collections. In this case, this register must be an empty `Coll[Byte]`.
- `R8` of this box is `Coll[(Coll[Byte], Coll[Byte])]` specifying other additional information about the artwork. the following pieces of information currently is intended to be in this register.
  - Whether the artwork is explicit content or not. The key is "explicit" and the value is 0x00 if it is not explicit and 0x01 if it is. If this key is not present the artwork is assumed to not be explicit content.

The order of these information should not be important as it is intended to represent a json with the following format:

```json
{
  "key1": "value1",
  "key2": "value2",
  "...": "..."
}
```
This field can be extended in the future based on the needs of Auction Houses around.


## Artist Identity
Artist identity is very important for any auction house to eliminate NFT copies and scams for the end-users.

The identity of the artist can not necessarily be determined with the issuance box (unless it is a P2PK box) since its correctness can not be verified, e.g., one can impersonate an artist.

Hence, the artist's identity and address are determined with the first P2PK input in the chain of transactions leading to the artwork issuance. This means that in the case of using proxy contracts, the first input of the transaction that is sending the funds to the proxy contract is going to determine the artist's identity.
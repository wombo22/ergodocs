---
tags:
  - Data Model
---
# Blocks

The Ergo block interval is 2 minutes, and each block will release 75 Ergs to be shared between the miners and the Treasury for the first two years. However, starting at year 2, the emission rate will fall by 3.0 Ergs and, after that, further decline every three months by an additional 3.0 Ergs, which originally resulted in an end to emission eight years after launch. With EIP-27, this has been extended to ~2045. 


## Extension Section

In Ergo, blocks are broken into sections, just like Bitcoin, Ethereum, and other blockchains. In Bitcoin, there is simply a block header and the transactions themselves. However, in Ergo, we have some extra sections that enable new functionality:

* Header
* Transactions
* Extensions
* Proofs of UTXO transformation

The 'extension' section contains certain mandatory fields (including links for NIPoPoWs, once per 1,024 block epoch) and parameters for miner voting, such as current block size. It can also contain arbitrary fields.

**What this means in practice is that different types of nodes and clients can download only those sections of the blocks they need – reducing the demands for storage, bandwidth, and CPU cycles.**


## Resources

- [Superblock Clients](log_space.md) are also possible on Ergo. 



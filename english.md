The Design and Implementation of Inter-blockchain Communication Between EOSIO Architecture Blockchains
---------
(BOSCORE TEAM)

This paper introduces the technical principle of IBC, the contracts and ibc_plugin developed by boscore team.

In order to realize the inter-blockchain transfer of token between two EOSIO blockchains, first, we need to 
solve two problems: 1. How to realize the lightweight client, 2. How to ensure the integrity and reliability 
of inter-blockchain transactions, how to prevent double flower and replay attacks.

The EOS mainnet and BOS mainnet are illustrated below. However, this document applicable for
any two EOSIO architecture blockchains.


### Terminology
- BOSIBC  
  IBC contracts and ibc_plugin developed by the BOSCORE technical team.
  
### Key concepts and data structures
- Simple Payment Verification (SPV)  
  Simple Payment Verification was first proposed in Bitcoin White Paper of Satoshi (https://bitcoin.org/bitcoin.pdf) 
  to verify that a transaction exists in a blockchain.`SPV client` stores contiguous block headers, but no blocks' body,
  so it only takes up a small amount of storage space. When receive a transaction and the `merkle path` of the transaction,
  `SPV client` can used to verify whether this transaction exist in the blockchain or not.

- Lightwight client (lwc)  
  also called `SPV client', is a lightweight chain composed of block headers.

- Merkle path  
  In order to verify whether a transaction exists in a blockchain, only the original data of the transaction 
  and the transaction's `merkle path` is needed, instead of the whole block. then calculate the merkle root and
  compaire with the merkle root of block header, if equal, indicates that the transaction exists in that blockchain.
  `merkle path` also known as `merkle branch`.

- Block Producer Schedule  
  The `BP Schedule` is a EOSIO architecture blockchain technology, which used to determine the rights that which 
  producer can produce blocks. The new version `BP Schedule` comes into effect after passing the validation of
  the last batch of `BP Schedule`. In order to ensure strict handover of BP rights, it is a core technology of 
  IBC system logic, that the lightweight client must follow the corresponding `BP Schedule` with it's mainnet.

- forkdb  
  When the EOSIO node runs, there are two underlying DBs used to store block information. 
  One is `blog`, i.e. `block log`, which is used to store irreversible blocks, and the other is `forkdb`, 
  which is used to store reversible blocks. Forkdb stores part of the block information at the top of the 
  current blockchain. A block must be accepted by forkdb before it finally enters the irreversible block. 
  The lightweight client mainly refers to the logic of forkdb.
  
### 3 Lightweight Client
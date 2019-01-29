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

In order to solve the cross-chain problem, the first problem to be solved is how to realize the lightweight client.
1. At where should the lightweight client runs? In the contract or out of the contract, for example, in plugin layer?
2. If running in contract, is it to synchronize all block header data of the opposite blockchain in real time, 
   or to synchronize part of block header data to verify transactions according to need, because synchronizing
   all block header data will consume a lot of CPU resources of two chains.
3. If running in contract, how to ensure the credibility of the lightweight client, how to prevent malicious attacks, 
   how to achieve complete decentralization, does not depend on the trust of any relay node.


#### 3.1 Should the Lightweight Client Run in the Contract
Bitcoin's light client was originally run on a single node (such as a personal computer or a decentralized 
Bitcoin mobile wallet) to verify the existence of transactions and to see the depth of that block. Specific 
implementation can be referred to [breadwallet](https://github.com/breadwallet/breadwallet-core).

IBC and Decentralized Wallet have different requirements for light clients. Decentralized wallets generally run
on personal mobile apps, providing transaction verification services for individual users, while IBC systems need 
light clients that are open, accessible and trustworthy to everyone. From this point of view, a light client that
can gain public trust can only run in the contract, because only the contract data is globally consistent and 
can not be tampered with. It is impossible to implement a trusted light client outside the contract, so BOSIBC 
run the light client in the contract. See source code of [ibc.chain](https://github.com/boscore/ibc_contracts/tree/master/ibc.chain).

#### 3.2 Whether to synchronize all block headers
In the light client of Bitcoin and ETH, a starting block and some subsequent verification block will be provided, 
and the light client will synchronize the full block headers after that starting block. Bitcoin produces only 4 Mb 
of block headers per year. According to the current storage capacity of mobile devices, it can be fully accommodated, 
and synchronization of these blocks will not consume a lot of computing resources of mobile devices. However, 
the situation of EOSIO is quite different. EOSIO contract consumes 0.5 milliseconds CPU add a block header to the contract 
and 0.2 milliseconds to deleted a block header, so 0.7 ms of CPU is needed for every block header procession.
Assuming that we want to synchronize all the block headers of the other EOSIO chain, according to the total CPU 
time of each block, now is 200ms, that is to say, `0.7ms/200ms = 0.35%` of the whole chain is needed to achieve 
real-time full synchronization. According to the actual total amount of delgated token of the whole network is about 
400 million, in order to make the IBC system works when the CPU is busy, the account which push block headers and ibc 
transactions needs to be delgate `400 million * 0.35% = 1.4 million` token, which is a large amount. Because EOSIO 
advocates multi side-chain ecosystem, let's assumed that there will be multiple side chains and EOSIO main network
to achieve cross-chain in the future, and between side chains and side chains, and it also will realizes one-to-many
cross-chains operation, assume to one-to-ten situation, each chain needs to maintain 10 light clients, 
in order to maintain these light clients, it needs to consume 3.5% of the whole network CPU of a single chain,
this proportion is too high, therefore, we need to find a more reasonable solution.

The process of designing inter-blockchain communication is a process of finding credible evidence. 
Is there a scheme that does not need to synchronize the whole block information, but also can guarantee the credibility
of light clients? The bottom of EOSIO source code has been prepared for this purpose.
Let's assume that if BP schedule never change, then at any time, when the ibc.chain contract obtains a series of
block headerss passed by signature verification, such as `n ~ n+336`, and more than two-thirds of the active BP
is producing blocks, we can sure that the nth block is irreversible, and can be used to verify cross-chain transactions.
Later, we need to consider the situation of BP schedule replacement. When BP schedule replacement occurs, 
we will not accept transaction verification until the replacement is completed. The relatively complex process
of BP replacement is dealt with, which will be described in more detail bellow.
Therefore, using this scheme can greatly reduce the amount of block headers that need to be synchronized,
and only when the BP list is updated or cross-chain transactions occured, that it's needed synchronized block headers.

In order to achieve this goal, the concept of `section` is introduced in ibc.chain. 
A section records a batch of continuous block headers. Section does not store block headers, instead, 
it records the first block number (first) and the last block number (last) of the block header, block headers are
stored in `chaindb`. Each section has a valid bool value. When there is no BP schedule replacement,
As long as two-thirds of the active BP is producing block and `last - first > lib_depth`, then 
the block of `first to last - lib_depth` is considered irreversible and can be used to verify cross-chain transactions.
When BP schedule is replaced, the 'valid' of that section becomes false and no transaction validation is accepted 
until schedule replacement finished and 'valid' become true, then continue cross-chain transaction validation.


#### 3.3 How to Ensure the Reliability of Light Clients
**3.3.1 forkdb**  
*1. How to append a new block to forkdb*  
A running eosio node maintains two underlying data structures 
[blog](https://github.com/EOSIO/eos/blob/master/libraries/chain/include/eosio/chain/block_log.hpp)
and [forkdb](https://github.com/EOSIO/eos/blob/master/libraries/chain/include/eosio/chain/fork_database.hpp),
blog is used to store irreversible block information, its stored data is serialized `signed_block`, 
forkdb is used to store reversible block information, and its stored data is `block_state`.
`block_state` contains more block-related information than `signed_block`. A block must first be appended to forkdb 
before it can eventually become irreversible and remove from forkdb into blog.
How can the `block_state` information of a block be obtained? It is not thart BP of the production block transfers 
`block_state` to other BP nodes and full-nodes, P2P network eos eosio pass only 
[signed_block](https://github.com/EOSIO/eos/blob/master/plugins/net_plugin/include/eosio/net_plugin/protocol.hpp#L142),
When a node receives a `signed_block` over P2P network, it uses this `signed_block` to construct `block_state` 
then verify the BP signature. [Relevance function](https://github.com/EOSIO/eos/blob/master/libraries/chain/block_header_state.cpp#L144),
The key points should be clarified are: 1. `blockroot_merkle`, 2. `get_scheduled_producer()`, 3. `verify_signee()`.

The signature verification related function are:
```
  digest_type   block_header_state::sig_digest()const {
     auto header_bmroot = digest_type::hash( std::make_pair( header.digest(), blockroot_merkle.get_root() ) );
     return digest_type::hash( std::make_pair(header_bmroot, pending_schedule_hash) );
  }

  public_key_type block_header_state::signee()const {
    return fc::crypto::public_key( header.producer_signature, sig_digest(), true );
  }

  void block_header_state::verify_signee( const public_key_type& signee )const {
     EOS_ASSERT( block_signing_key == signee, wrong_signing_key, "block not signed by expected key",
                 ("block_signing_key", block_signing_key)( "signee", signee ) );
  }
```

The first step in verifying signature is to obtain signature digest, i.e. `sig_digest()`, 
which uses `header.digest()`, `blockroot_merkle.get_root()` and `pending_schedule_hash`.
The second step is to obtain the signature public key, i.e. `signee()`, which compute the BP public key through
`producer_signature ` and `sig_digest()`.
The third step is to verify whether the public key is correct, i.e. `verify_signee()`, which is called in 
`block_header_state:: next()`, after validation, a block is added to forkdb main branch.

So every block which added to forkdb has undergone a very rigorous and comprehensive verification.
The core logic contains `blockroot_merkle`, `get_scheduled_producer()` and `verify_signee()`.
The ibc.chain contract fully inherits the rigorous validity of forkdb.




















































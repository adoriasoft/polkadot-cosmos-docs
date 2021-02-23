# Project description
 
The ultimate goal of this project is to create tools that allow building Polkadot parachains with the Cosmos SDK.
 
## Table of Contents
[1. How it works?](#How-it-works?)<br>
[2. ABCI](#ABCI)<br>
[3. Polkadot-Cosmos integration](#Polkadot-Cosmos-integration)<br>
[4. Cosmos CLI and RPC](#Cosmos-CLI-and-RPC)<br> 
[5. ABCI calls in Substrate](#ABCI-calls-in-Substrate)<br>
[6. Token](#Token)<br>
[7. Validators](#Validators)<br>
[8. Pallet subscription](#Pallet-subscription)<br>
[9. Demo application](#Demo-application)<br>
[10. Versions](#Versions)<br>
[11. User-guides](#User-guides)<br>
 
 
## How it works?
In Cosmos SDK, Tendermint core encapsulates the Network and Consensus layers of a node. It interacts with the Application layer, which defines the system's business logic, using ABCI interface. The communication between customer application and Tendermint core is similar to 'client-server,’ customer application is the server, and Tendermint core is the client.
![Figure 1 - The interaction of two Cosmos nodes](https://raw.githubusercontent.com/adoriasoft/polkadot-cosmos-docs/master/img/ABCI-Detailed-explanation.png)
Figure 1 - The interaction of two Cosmos nodes
 
## ABCI
ABCI contains a set of methods including the following:
- CheckTx - check and set transaction priority before adding to the transaction pool
- BeginBlock - start block execution
- DeliverTx - execute a transaction from the block
- EndBlock - finish bock execution
- Commit - approve changes after block execution
- InitChain - load initial state from JSON file
- Query - query for data from the application
- And other.
 
Full specification is available [here](https://docs.tendermint.com/master/spec/abci/abci.html).
We've already finished the implementation of all ABCI methods, excluding the set of Snapshot methods.
 
## Polkadot-Cosmos integration
 
To minimize changes in Substrate and Cosmos cores, we add a new pallet - `cosmos-abci` - that contains one type of extrinsics - `abci_transaction` - that is used as a wrapper for regular Cosmos transactions.
 
We analyzed several possible solutions for integration:
- Runtime interfaces
- Off-chain workers;
- Common database.
 
### Runtime interfaces
Runtime interfaces allow us to use external libraries and create network connections directly from runtime. Initially, we used HTTP protocol that was natively supported by Substrate and HTTP-GRPC gateway on the application (Cosmos) side. Then we add [Tonic library](https://github.com/hyperium/tonic) that provides GRPC protocol to Substrate node, and now we send GRPC requests directly to Cosmos node.
 
![Figure 2 - Transaction and block processing in Substrate-Cosmos app](https://raw.githubusercontent.com/adoriasoft/polkadot-cosmos-docs/master/img/Architecture-direct.png)
 
Figure 2 - Transaction and block processing in Substrate-Cosmos app
 
1) New block or transaction is imported, Substrate runtime sends a GRPC request with data to Cosmos node.
2) Cosmos runtime updates its state.
3) Cosmos runtime processes data and sends  GRPC response to Substrate runtime.
4) Substrate runtime receives a response and updates its state.
 
Cosmos-abci [pallet](https://github.com/adoriasoft/polkadot_cosmos_integration/tree/master/substrate/pallets/cosmos-abci) contains logic of calls to main ABCI methods (checkTx, deliverTx, beginBlock, endBlock, commit, initChain). These methods are called during chain initialization, transaction validation, and block execution. Tonic library, necessary proto files, and GRPC calls are located [here](https://github.com/adoriasoft/polkadot_cosmos_integration/tree/master/substrate/pallets/cosmos-abci/abci).
 
**Problem with on_initialize**
 
In general, the approach based on runtime interfaces seems to be the simplest and the most convenient. But we found a strange behavior that `on_initialize` can be called multiple times on the same block height and even after `on_finalize` on this height. Also, it was impossible to get a hash of the current block during `on_initialize`. According to the [reply](https://github.com/paritytech/subport/issues/43) of support:
1) `on_initialize` is called any time that a runtime function is called from the client, so it is normal to see multiple calls to `on_initialize`.
2) There is no guarantee that `on_finalize` will execute after `on_initialize`.
3) `on_initialize` and `on_finalize` are called before the block hash and extrinsic root are calculated, so it is expected that you would not be able to access them in either function; these values are calculated immediately after calling `on_finalize`.
 
That's why block processing (beginBlock, deliverTx, endBlock, commit) can't be done using these methods because it doesn't guarantee the order of their execution.
Runtime interfaces are used for transaction checking.
 
### Off-chain workers
 
Off-chain workers for each block are called when block processing is finished. During extrinsic processing, `abci_transaction` extrinsics are saved into storage. The off-chain worker calls `beginBlock`, gets all necessary extrinsics from storage, calls `deliverTx` for each one, removes them from storage, and calls `endBlock` and `commit`.
 
![Figure 3](https://raw.githubusercontent.com/adoriasoft/polkadot-cosmos-docs/master/img/Off-chain-workers.png)
 
Figure 3 - Off-chain worker
1) Runtime saves data from new block to storage.
2) Block processing is finished, runtime calls off-chain worker.
3) Off-chain worker gets data from the block from storage.
4) Off-chain worker sends GRPC request with data to Cosmos node
5) Cosmos runtime processes data and sends GRPC response to Off-chain worker.
6) Off-chain worker removes executed transactions from storage.
 
 
### Common database
Off-chain workers cannot write to the primary storage of the node, but they have additional local storage that can be accessed both from runtime and off-chain workers. Also, this storage is not synchronized between nodes. In fact, storage is a Rocks database, so potentially other processes on the same device can get access to this DB.
The idea is:
- Off-chain workers or runtime itself writes data (requests) that must be processed by Cosmos application (transactions, blocks, etc.) to the local storage.
- Cosmos app is continuously monitoring the DB, and when it finds new appropriate data, it takes it and processes it. Then Cosmos removes requests from DB and writes responses.
- During the processing of a new block/transaction Substrate finds responses in DB, performs necessary actions, and removes the responses.
 
RocksDB does not support full interaction with the database by two processes, it only allows the "main" process to read/write, and the "secondary" process only to read, which categorically doesn’t suit us.
 
It seems to be a bad idea to implement data exchange through a shared file in this context because Cosmos will have to constantly read some sections in the database, looking for the requests. It will be slow and difficult to implement. The substrate can't "trigger" Cosmos when writing to the DB, unlike the HTTP-requests approach, where Cosmos starts acting after a Substrate request. Theoretically, we can change the DB  layer but it seems to be unnecessary work with many potential problems.
 
Moreover, RocksDB doesn't support simultaneous work with two processes when both can write to it.
https://github.com/facebook/rocksdb/wiki/Secondary-instance
https://github.com/facebook/rocksdb/issues/908
https://github.com/facebook/rocksdb/issues/118
 
 
## Cosmos CLI and RPC
A user can interact with its Cosmos node using CLI. This CLI doesn't connect directly to the Cosmos node but to the Tendermint core that redirects requests from the CLI to the node using ABCI and then returns responses to the CLI. It allows developers to use the same CLI for different applications, just connecting Tendermint to them.
 
For interactions with CLI, Tendermint contains an additional json-rpc server, a bit similar to the server used Substrate for interaction with web interface.
We implemented an [rpc server](https://github.com/adoriasoft/polkadot_cosmos_integration/tree/master/substrate/node/src/cosmos_rpc) as a component of Substrate node to allow interactions between CLI and Cosmos.
 
![Figure 4](https://raw.githubusercontent.com/adoriasoft/polkadot-cosmos-docs/master/img/CLI-scheme.png)
 
Figure 4 - Interaction between Cosmos CLI and Cosmos node using Substrate RPC
 
1) User sends a new request using Cosmos CLI.  Cosmos CLI sends it to Substrate.
2) Substrate RPC receives the request and calls an appropriate ABCI method in ABCI pallet.
3) ABCI pallet sends a GRPC request to Cosmos runtime.
4) Cosmos runtime receives the request and may get some necessary information from storage.
5) Cosmos runtime sends a GRPC response to Substrate runtime.
6) Substrate runtime gets the response and
   - a) may update its state according to the response.
   - b) may add a new transaction to mempool.
7) Substrate runtime returns a response to Substrate RPC.
8) Substrate RPC sends a response to Cosmos CLI.
 
 
Tendermint server provides the following [API](https://docs.tendermint.com/master/rpc/). All methods are divided into several groups:
- *Tx* - these methods creates new transactions - (done)
 
   `broadcast_tx_async`- Returns right away, with no response; does not wait for CheckTx nor DeliverTx results.
  
   `broadcast_tx_sync` - Returns with the response from CheckTx; does not wait for DeliverTx result.
  
   `broadcast_tx_commit` - Now acts the same way as `broadcast_tx_sync`. According to the original docs, this call should return with the responses from CheckTx and DeliverTx, but in our case, when blocks are created by Substrate runtime and then executed by an off-chain worker, it's a challenging task to track the inclusion of specific transaction into a block.
  
   `check_tx` - Checks the transaction without executing it.
  
- *ABCI* - calls to ABCI Info and Query methods - (done)
- *Unsafe API*
    We analyzed the use cases of `dial_seeds` and `dial_peers` RPCs and came to the conclusion that Substrate doesn't need it.
 
   As in Tendermint docs specified: 
DialSeeds: Dial a peer, this route in under unsafe, and has to manually enabled to use;
DialPeers: Set a persistent peer, this route in under unsafe, and has to manually enabled to use.
Substrate already has pretty much the same API inside Node CLI (--bootnodes or --reserved-nodes) and p2p protocol that connects all nodes.
 
   We could add them as dummy methods that do nothing but return a response with success, but we think this will be more unclear to those who will use it.
- *Websocket* - (will be done later)
- *Info* - (will be done later)
 
 
## ABCI calls in Substrate
 
As described above, different ABCI methods are called in different places. Full specification of standard requests and responses is [here](https://github.com/adoriasoft/polkadot_cosmos_integration/blob/develop/pallets/cosmos-abci/abci/proto/types.proto).
 
| \# | Method | Substrate method |
| :------------: | :------------: | :------------: |
| 1 | initChain | load_spec |
| 2 | checkTx | validate_transaction |
| 3 | beginBlock | cosmos_abci::offchain_worker |
| 4 | deliverTx | cosmos_abci::offchain_worker |
| 5 | endBlock | cosmos_abci::offchain_worker |
| 6 | commit | cosmos_abci::offchain_worker  |
| 7 | query | json-rpc server |
| 8 | info | json-rpc server |
| 9 | setOption | json-rpc server |
| 10 | echo | json-rpc server |
| 11 | flush | json-rpc server |
 
## Token
Both Substrate and Cosmos SDK contain many pallets\modules useful for blockchain developers. Some of them use native tokens, above all consensus-related (staking, validators' election and slashing), and governance modules. The security of any stake-based decentralized system is based on the economic motivation of participants. In our case, Cosmos SDK is responsible for all application logic, so the Cosmos token is the main one.
 
Cosmos native consensus - Tendermint - knows nothing about Cosmos token logic; it gets the minimum necessary information (for example, the list of validators) through ABCI and uses it.
 
There are two ways to allow Substrate modules to use all the benefits of token: associate Substrate tokens with Cosmos ones or ignore Substrate tokens and make Substrate blindly trust Cosmos solutions. We use the second approach because it's nearer to Cosmos-Tendermint relations, and Cosmos apps can be launched on top of Substrate without any changes. At the same time, this way may require small changes in core pallets related to validators.
 
## Validators
Both Substrate and Cosmos use DPoS BFT consensuses but with some differences.  Substrate is based on a hybrid consensus (BABE or AURA) + GRANDPA; Cosmos works with Tendermint. Staking during validators elections is one of the main token use cases. In our case, Substrate is responsible for the consensus layer, but token and stacking logic are defined in Cosmos, so we need to elect Substrate validators according to the Cosmos stacking system. For this purpose, we need to match Cosmos and Substrate validator addresses so that they can be elected in Cosmos and participate in consensus in Substrate.
 
In Substrate consensuses, all validators are equal because of the NPoS election, but in Cosmos, validators' weights in consensus depend on their stakes, so we need to use weighted voting in BABE and GRANDPA.
 
### Matching validators
 
The abci-pallet's storage contains a few maps to provide matching Cosmos and Substrate accounts. `CosmosAccounts` maps Cosmos keys to Substrate keys, `SubstrateAccounts` does vice versa. Any Substrate user can register their Cosmos account using `insert_cosmos_account` extrinsic and remove an existing account using `remove_cosmos_account` extrinsic.
 
The current version of `insert_cosmos_account` extrinsic doesn't verify that a Substrate user really controls a corresponding Cosmos account. It will be fixed later by adding a digital signature made by the Cosmos key to the extrinsic.
 
### Validator update
Cosmos informs Tendermint/ABCI pallet about changes in the list of validators using the `validator_updates` field in the response of `EndBlock`. Each validator in the list is represented by `pub_key` and `power` (`power` is an integer directly proportional to the stake).
 
An off-chain worker, where `EndBlock` is called, can't write updates directly to Substrate storage; also, the off-chain worker's storage doesn't guarantee data safety after restarts. That's why an additional instance of RocksDB was added to the Substrate node, and it can be accessed from both off-chain worker and Substrate runtime. This DB also allowed solving the problem of node crashes after restarts - Substrate calls `InitChain` each time that contradicts the requirement of Cosmos where `InitChain` must be called only once. When `InitChain` is called first, a special flag is set in DB that blocks repeated calls.
 
Validator updates in Substrate are implemented through the trait [`SessionManager`](https://substrate.dev/rustdocs/v2.0.0/pallet_session/trait.SessionManager.html) and its method `new_session` that returns the list of `ValidatorId`. The pallets that use this validator list - Aura, BABE, GRANDPA - implement [`OneSessionHandler`](https://substrate.dev/rustdocs/v2.0.0/pallet_session/trait.OneSessionHandler.html) and process the received list in `on_new_session`.
The current version of Substrate `session` pallet [doesn't support](https://github.com/paritytech/subport/issues/86) weighted voting (`new_session` returns only the list of validators without weights). At the same time, BABE and GRANDPA operate with authorities' lists with weights, and the equal weights of all authorities from `new_session` are hardcoded in `on_new_session` methods.
 
In ABCI pallet, `new_session` gets the list of Cosmos validators for a corresponding block from DB using runtime interfaces and matches them with corresponding Substrate accounts registered through `insert_cosmos_account` extrinsic. The validators' weights received from Cosmos are accessible in the pallet but can't be used by consensus pallets because of the limitations described above. This logic is completely correct for Aura, which doesn't work with weights at all.
 
To add a fully functional weighted voting to Substrate, we need to modify several pallets: session, aura, grandpa, babe. It may be done in the later versions.
 
Substrate has limits for a maximum frequency of validator updates (minimum session length) relates to the logic of [session change](https://github.com/paritytech/substrate/discussions/7801). That's why we can't update validators on each block in contrast to Cosmos and Tendermint, but only every two blocks.
 
 
### Validator rewards
Tendermint informs Cosmos about validators who signed a current block using `last_commit_info` field from `BeginBlock` request. To use the same approach with Substrate is more complicated because GRANDPA finalizes not each block but a chain of blocks at once. That's why it's impossible to get the list of signers for each block. A possible solution - limit the maximum chain length in GRANDPA to one block. It will remove an important advantage of GRANDPA and allow Substrate to interact with Cosmos more similar to the way the Tendermint does. It will also solve the fork resolution problem that Cosmos SDK doesn’t support.
 
The current version of ABCI pallet deals only with the ideal case when all validators work honestly and sign each block: `last_commit_info` contains the list of ALL current validators.
 
`last_commit_info` should contain Cosmos validators' addresses that are [calculated](https://docs.tendermint.com/master/spec/abci/abci.html#validator) using SHA256 hash function, which is not natively supported by Substrate. The implementation of SHA256 was imported to the ABCI pallet.
 
### Consensus parameters update
 
Tendermint can update a set of system parameters according to `EndBlock` response. These [parameters](https://docs.tendermint.com/master/spec/core/data_structures.html#validatorparams) include:
 
**BlockParams**
- `max_bytes` - an equivalent of Substrate [MaximumBlockLength](https://substrate.dev/rustdocs/v2.0.0/frame_system/trait.Trait.html#associatedtype.MaximumBlockLength)
- `max_gas` - an equivalent of Substrate [MaximumBlockWeight](https://substrate.dev/rustdocs/v2.0.0/frame_system/trait.Trait.html#associatedtype.MaximumBlockWeight)
- `time_iota_ms` - an equivalent of Substrate [Slot duration](https://substrate.dev/rustdocs/v2.0.0/sc_consensus_slots/struct.SlotInfo.html#structfield.duration)
 
All these parameters are parts of Substrate runtime and can't be easily updated from the off-chain worker that can't write directly to Substrate storage. These parameters are critical for system stability, and Substrate doesn't provide mechanisms to update them easily. There is a possible solution - add a new extrinsic type that allows such updates. This extrinsic will require a more complex verification and connection with Substrate governance mechanisms. This feature is not extremely important at the current stage and will be implemented later.
 
**EvidenceParams**
- `max_age_num_blocks`, `max_age_duration`,
`max_bytes` - these parameters are related to slashing mechanism and will be analyzed later.
 
**ValidatorParams**
 
- `pub_key_types` - unnecessary for Substrate at this stage.
 
**VersionParams**
 
- `app_version` - unnecessary for Substrate at this stage.
 
 
### BABE and AURA
The current version of the node supports both BABE and AURA consensuses, but only one can be used by one node. The consensus is chosen according to the feature (`--features "std aura"` or `--features "std babe"`) used during node initialization.
 
## Pallet subscription
The subscription mechanism allows other pallets to expand the `check_tx` and `deliver_tx` logic. This approach is similar to the `SessionManager` and `OneSessionHandler` interactions. The trait `SubscriptionManager` contains two methods: `on_check_tx` and `on_deliver_tx` that are called after the corresponding methods of the abci pallet.
 
## Demo application
To demonstrate the correct work of our module, we chose a simple Cosmos SDK-based application, "Nameservice": [tutorial](https://tutorials.cosmos.network/nameservice/tutorial/00-intro.html), [source code](https://github.com/cosmos/sdk-tutorials/tree/master/nameservice).
Small changes have been made in the code to launch the application with a newer version of Cosmos SDK, [updated version](https://github.com/adoriasoft/cosmos-sdk/tree/feature/add_nameservice).
 
## Versions
Substrate - v2.
Cosmos SDK - master branch on commit [89097a0](https://github.com/adoriasoft/cosmos-sdk/commit/89097a00d7f6d6339c377f6c87bea8fa5068d125).
 
 
## User guides
[Launch Substrate](https://github.com/adoriasoft/polkadot_cosmos_integration/blob/master/README.md)
 
[Launch Cosmos and send transactions](https://github.com/adoriasoft/cosmos-sdk/blob/feature/add_nameservice/simapp/README.md)

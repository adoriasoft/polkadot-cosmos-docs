# Roadmap
 
### Phase 1
 
The first phase included initial research of Substrate and Cosmos SDK and the implementation of basic ABCI methods for their interactions.
 
**Milestone 1 (Complete)**
 
- Create a pallet with basic ABCI methods: InitChain, BeginBlock, EndBlock, DeliverTx, CheckTX;
- Add GRPC to Substrate (initially, using HTTP-GRPC gateway);
- Implement a simple ERC20-like token that works similarly to Cosmos SDK app.
 
**Milestone 2 (Complete)**
 
-  Add GRPC to Substrate (without gateway);
- Update ABCI methods (InitChain, BeginBlock, EndBlock, DeliverTx, CheckTX) for usage standard data formats used in ABCI;
- Launch a simple Cosmos app (Nameservice) using Substrate instead of Tendermint;
- Add  ABCI Query and RPC into Substrate for interactions with Comos CLI.
 
### Phase 2
 
During phase 1, we've implemented the main methods of ABCI (checkTx, deliverTx, beginBlock, endBlock, commit, initChain). Still, there is a set of other useful ABCI methods that can be implemented (Info, SetOption, Query, Flush, Echo). Tendermint acts as an intermediary between Cosmos CLI and application; the same should be done by Substrate. Some of these calls are just redirected by Tendermint, but others may be processed in a more complex way. Also, Tendermint reacts on Cosmos ABCI responses, and one of the most important cases is validators list and consensus parameters updates according to EndBlock response.
 
**Milestone 3 (Complete)**
 
- Implement transactions broadcast APIs and ABCI in Substrate RPC APIs similar to https://docs.tendermint.com/master/rpc/#/ ;
- Add ABCI methods Info, SetOption, Query, Flush, Echo in Substrate RPC and ABCI_pallet so that they can be used for interaction between Cosmos CLI and Cosmos node;
- Process ABCI responses in ABCI pallet similar to their processing in Tendermint (excluding validator list and consensus parameters updating in EndBock, it will be done during next milestone);
- Cover Cosmos transactions with Substrate unsigned transactions instead of signed.
 
**Milestone 4 (In progress)**
 
*Validators election and consensus*
 
- Modify ABCI pallet so that it will be able to update the list of validators with their weights and consensus parameters according to EndBlock responses.
- Connect BABE and GRANDPA pallets to Substrate node and use weighted voting in these consensuses.
- Configure Substrate and Cosmos so that validators participating in Substrate consensus will get rewards in Cosmos token.
 
*Nodes synchronization*
 
- Modify Cosmos launcher so that it will be able to start Substrate node automatically, possibly as a daemon, using special flags.
- Cosmos nodes will be able to monitor substrate logs and stop both nodes in the event of a crash.
 
*Pallet subscription*
 
- Provide other Substrate pallets functionality to subscribe on abci pallet like it's done in session pallet.
 
### Phase 3
 
**Nodes synchronization**
 
Unsynchronization can be caused by a number of reasons: one of the nodes can crash, blockchain or state storage may be damaged.  In the event of unsynchronization between Cosmos and Substrate, chain and state should be returned to a shorter chain, which is related to the fork resolution problem, described in the next subsection.
 
*Planned deliverables:*
- Synchronize nodes in case of asynchronization (choose the shorter chain and request other blocks from the network);
- Simulate an attack that causes asynchronity.
 
**Fork resolution**
 
In Tendermint, block producing and finalization are a single process, so forks can happen only in the event of the validator misbehavior. Substrate consensus divides this process into two parts: BABE\AURA for block producing and GRANDPA for block finalization; this approach is faster but creates orphan blocks. So, Cosmos-Substrate nodes should detect forks, roll-back their state, and choose finalized branches. 
 
*Planned deliverables:*
- Connect Cosmos rollback mechanisms with Substrate;
- Simulate an attack that causes fork.
  
**Governance + Runtime upgrade**
 
Both systems have government mechanisms that allow to change system parameters and even the source code in the case of Substrate. Governance mechanisms are controlled by referendum system, where each change must be approved by token-based voting. Cosmos internal governance can work despite used consensus. We want to allow Cosmos users to manage both Cosmos and Substrate parts of thor nodes. A special case of governance is runtime upgrade that needs additional research.
 
*Planned deliverables:*
- Allow Substrate parameters updates using Cosmos governance;
- Allow Substrate runtime updates using Cosmos governance.
 
**Slashing**
 
Slashing mechanisms prevent or, at least, minimize validator misbehavior in both Substrate and Cosmos systems. Each validator has a deposit that can be slashed in case of malicious actions. Fishermen in Substrate or Hackers in Cosmos can send claims about misbehavior.
 
High-level misbehavior like validator collusion is difficult to detect automatically, and it may require claims from users and voting for decisions. We think this case doesn't depend on consensus and will work in Cosmos app with no changes. Simpler cases like equivocating can be detected and claimed automatically. We think about using off-chain workers for this purpose.
 
*Planned deliverables:*
- Connect Substrate equivocating detection with Cosmos slashing.
 
**Transaction priority**
 
Transactions in the transaction pool are ordered according to some priorities. A validator may want to prioritize transactions according to a custom criterion.
Substrate method `validate_transaction` returns `TransactionValidity` struct, its field `priority` determines the transaction priority in the transaction pool. We add simple logic that increases the priority of transactions according to Cosmos respond, which is as follows:
 
`priority += GasWanted - GasUsed`
 
We want to analyze and make the priority system more customizable.
 
*Planned deliverables:*
- More elaborate, possibly customizable transaction priority system.
 
### Phase 4
**ABCI Snapshots**
 
Cosmos ABCI contains a set of Snapshots methods that allow creating and sharing the state on specific height without the whole chain of previous blocks.
- `ListSnapshots` - used during state sync to discover available snapshots on peers.
- `LoadSnapshotChunk` - used during state sync to retrieve snapshot chunks from peers.
- `OfferSnapshot` - OfferSnapshot is called when bootstrapping a node using state sync. The application may accept or reject snapshots as appropriate. Upon accepting, Tendermint will retrieve and apply snapshot chunks via ApplySnapshotChunk. The application may also choose to reject a snapshot in the chunk response, in which case it should be prepared to accept further OfferSnapshot calls.
- `ApplySnapshotChunk` - the application can choose to refetch chunks and/or ban P2P peers as appropriate. Tendermint will not do this unless instructed by the application.
 
*Planned deliverables:*
- Implement Snapshot methods in Substrate RPC;
- Probably, extend Cosmos snapshots with Substrate state.
 
**Substrate RPC**
 
There is a set of additional APIs in Tendermint RPC (Info_rpcs and Websocket_rpcs https://docs.tendermint.com/master/rpc/#/), they are not critical for the work of Substrate-Cosmos node but make user interface more convenient and usable.
 
*Planned deliverables:*
- Implement all methods from Tendermint RPC in Substrate RPC.
 
### Phase 5
Our project’s primary goal is to allow blockchains based on our Substrate-Cosmos SDK to interact in both Polkadot and Cosmos systems. It’s difficult to describe the details of this phase because both systems are still under development, and many aspects may change, and therefore deliverables.
 
*Planned deliverables:*
- Add IBC module to Cosmos;
- Connect node to Cosmos Hub;
- Add Polkadot consensus and XCMP to Substrate part;
- Implement Cosmos module for using XCMP;
- Connect node to Polkadot Relay Chain (probably to a test relay chain, because the mainnet will have a rather complex procedure of registration of new parachains).
 

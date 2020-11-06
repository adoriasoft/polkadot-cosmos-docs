# Cosmos SDK Parachain development kit

This repository contains documentation for the [project](https://github.com/adoriasoft/polkadot_cosmos_integration). The project is being developed, some parts of documentation may be out-of-date. 

## Project Overview
Polkadot and Cosmos are two projects with a similar aim: integrate independent blockchain technologies in a single infrastructure supporting communication between them. 

The core idea of our project is to combine Polkadot shared security with the flexibility of Cosmos SDK, to allow developers to create parachains based on Cosmos SDK. We are building functionality that allows anyone who has built a chain with the Cosmos SDK to turn that chain into a full Polkadot parachain; so any sovereign chain created with the Cosmos SDK could be connected with Polkadot and not be responsible for its own security.

## Current stage
By this moment, we have implemented a basic integration between Substrate and Cosmos SDK that allows transaction and block producing and validation.  Tendermint consensus and network layers used in Cosmos have been replaced with Substrate-based “sub-node". We have implemented a new Substrate pallet that acts like Tendermint. Changes in Cosmos SDK were minimized to allow Cosmos developers to change consensus and network layers of their applications as easy as possible. To run Cosmos SDK with the Substrate a user need only enable a few flags during the node start (disable Tendermint consensus and enable grpc). 

A more detailed description of the finished steps and future plans can be found in the [roadmap](https://github.com/adoriasoft/polkadot-cosmos-docs/blob/master/roadmap.md).

The details of solution architecture are described [here](https://github.com/adoriasoft/polkadot-cosmos-docs/blob/master/description.md).

## Links
 - [User guide](https://github.com/adoriasoft/polkadot_cosmos_integration/blob/master/README.md)
 - [Polkadot](https://polkadot.network/)
 - [Cosmos](https://cosmos.network/ )

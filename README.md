# Kadena Chainweb Indexer

## Introduction

This tool automatically retrieves events from a Chainweb Node:
  - In real-time using block streams (with a minimum of 2 confirmations)
  - In the block history (backward indexing), to fill missing non indexed blocks

The tool manages missed blocks and events according to the configuration.

## How to use

Make sure you have
  - a running Kadena node (see config)
  - a running MongODB (see config)
  - a YAML config file

If you installed through wheel:
```sh
python -m kadena_indexer config.yaml
```

Otherwise you can run the PEX file:
```sh
./kadena_indexer_v1.0.0.pex config.yaml
```


## Configuration

The tool is configured using a YAML file:

Three fields are mandatory:

- `db` : name of the MongoDB database
- `node`: URL of the Chainweb node service endpoint
- `event`: Array of events to be handled by this indexer.

Each event is defined by:
 - `name`: FQN of the event: namespace + module + event
 - `chains`: Array of chains from which the events must be indexed
 - `height`: 2-element array defining the indexed range.
    - Array can be `null` to indicate an unbounded indexing.
    - A `null` for one of both bounds indicate an infinite minimum or infinite maximum


**Example:**

```yaml
db: kadena_events
node: http://localhost:1848
events:
  - name: coin.TRANSFER
    height: ~
    chains: ["1","2"]

  - name: coin.TRANSFER
    height: [4000000, 4600000]
    chains: ["3","4","5","6","7","8","9","10","11","12","13"]

  - name: n_582fed11af00dc626812cd7890bb88e72067f28c.bro.TRANSFER
    height: [4000000, ~]
    chains: ["1","2"]

  - name: n_e47192edb40ff014ab9c82ae42972d09bbad4316.bro-lottery.TICKET-BOUGHT
    height: [4000000, ~]
    chains: ["2"]

  - name: n_e47192edb40ff014ab9c82ae42972d09bbad4316.bro-lottery.LOTTERY-ROUND
    height: [4000000, ~]
    chains: ["2"]

  - name: n_e47192edb40ff014ab9c82ae42972d09bbad4316.bro-lottery.SETTLED
    height: [4900000, ~]
    chains: ["2"]

  - name: kaddex.exchange.UPDATE
    height: [3800000, ~]
    chains: ["2"]

  - name: kaddex.exchange.SWAP
    height: [3800000, ~]
    chains: ["2"]
```

## Chainweb node Configuration

The node must expose its service endpoint.
The node must be configured with `headerStream: true`


## MongoDB configuration and layout

Since the indexers make use of MogoDB transactions, **MongoDB must be configured as a Replica Set**

The indexer automatically creates:
  - A *technical* collection called `coordinator`
  - A collection per event (ie: `coin.TRANSFER`)

Inside an events collection, the indexer creates 1 document per event:
```js
{
  name: "" //Event name
  params: [], // Aray: Parameters of the event
  reqKey: "", // Request key of the transaction
  chain: "", // Chain
  block: "", // Block hash containing the transaction
  rank: 0, // Event rank inside the block. Rank 0 is always the coinbase transfer event
  height: 0, // Block height of the event
  ts: ISODate('2022-07-15T00:35:57.399Z'); // Creation date of the block, differs from Pact data
}
```
- Decimal parameters are automatically converted to Mongo `Decimal128`
- Integers parameters less than 64 bits are automatically converted to Mongo integers.

The indexer creates its own indexes. But the user is encouraged to create his own indexes depending on his needs and event types.

## Future improvements

- Support a MongoDB from another host
- Better manage the case when a event/chain is completely removed from the config
- Devnet support
- Improve backward indexing performance
- Add a command line to manually prune an event/chain/range to force re-indexing

# Hobbits

**This document follows [RFC-2119](https://tools.ietf.org/html/rfc2119).**

## Table of Contents

1. [Abstract](#abstract)
2. [Flow](#flow)
3. [Wire Protocol](#wire-protocol)
    1. [Messages](#messages)
4. [Protocols](#protocols)
    1. [RPC](#rpc)
    2. [GOSSIP](#gossip)
    3. [PING](#ping)
5. [Implementations](#implementations)

## Abstract

Hobbits is a modular wire protocol which allows implementers to experiment with various application logic over a practical network in an agnostic and expedient manner.

Hobbits wire protocol was developed so that Eth2.0 clients could begin network-level testing without requiring libp2p.

## Flow

We layout a network where the nodes are peered in the following manner:

![nodes](./nodes.png)

The following flow dictates how blocks are shared amongst peers:

![sequence diagram](./diagram.png)

1. Node 1 receives a new block `0xCAB` and gossips `hash(0xCAB)` to Node 2 and Node 3
2. Node 2 requests the block bodies for block `0xCAB` from Node 1 in an RPC Command query
3. Node 1 responds to Node 2 with the block bodies for block `0xCAB` in an RPC Command response
4. Node 2 receives a new block `0xCAB` and gossips it to Node 1 and Node 3

## Wire Protocol

### Messages

The `message` format looks as follows:

```
<preamble><version><protocol><header-length><body-length><header><body>
```

| Field | Definition |
| ----- | ---------- |
| preamble | utf-8 encoded `EWP` |
| version | 4-byte encoded `uint32` that represents the hobbits version. (Current Version: 3) |
| protocol | 1-byte encoded `uint8` that represents the protocol (0 - RPC, 1 - GOSSIP, 2 - PING) |
| header-length | 4-byte encoded `uint32` that represents the header length. |
| body-length | 4-byte encoded `uint32` that represents the body length. |
| header | byte encoded header |
| body | byte encoded body |

**NOTE:** 
 - Message `body`s along with `header` fields are BSON encoded.
 - All `int` types are big endian encoded.

#### Fields

Every hobbit message contains the following fields: 

| Field | Definition | Validity |
|:------:|----------|:----:|
| `version` | Defines the EWP version number e.g. `3`. | `uint32` |
| `protocol` | Defines the [protocol](#protocols). | `uint8` |
| `header` | Defines the header | payload |
| `body` | Defines the body | payload |

## Protocols

Hobbits defines 3 protocols that dictate how messages are interpreted and responded to.

### RPC

The RPC protocol defines the interaction between two peers exchanging messages to coordinate their actions.

The messages range from communicating status and metadata to exchanging block data to synchronize chains.

#### Envelope

```python
{ 
  'method_id': 'uint16' ## byte representing the method
  'id': 'uint64' ## id of the request
}
{
  'body': 'bytes' ## body of the request itself
}
```

The `body` field contains an [SSZ](https://github.com/ethereum/eth2.0-specs/blob/dev/specs/simple-serialize.md) encoded byte array. 

#### RPC Methods

There are two types of RPC methods: one that requests data and one that responds with data.

* Peers MAY request blocks and headers from other peers, possibly in response to a `GOSSIP` message. 
* Peers MAY request blocks repeatedly from the same peers.
* Other peers MAY respond on a best effort basis with header and block data.
* There is no SLA for responding. 

##### Handshake

###### `0` HELLO

Upon discovering each other, nodes MAY exchange `HELLO` messages.

Nodes MAY send `HELLO` to other peers when they exchange messages for the first time or when their state changes to let them know new blocks are available.

Nodes can send `HELLO` messages to each other to exchange information on their status, with the following information contained in the body of the message:

```python
{
  'network_id': 'uint8' ## the ID of the network (1 for mainnet, and some predefined number for a testnet)
  'chain_id': 'uint8' ## the ID of the chain (1 for ETH)
  'latest_finalized_root': 'bytes32' ## the hash of the latest finalized root
  'latest_finalized_epoch': 'uint64' ## the number of the latest finalized epoch
  'best_root': 'bytes32' ## the hash of the best root this node can offer
  'best_slot': 'uint64' ## the number of the best slot this node can offer
}
```

Upon receiving a `HELLO` message, the node SHOULD reply with a `HELLO` message.

###### `1` GOODBYE

Nodes MAY signal to other nodes that they are going away by sending a `GOODBYE` message, with the following information contained in the body of the message:

```python
{
  'reason': 'uint64' ## an optional reason code up to the client
}
```

The reason given is optional. Reason codes are up to each client and SHOULD NOT be trusted.

Upon receiving a `GOODBYE` message, no response is necessary.

##### `2` GET_STATUS

Nodes MAY exchange metadata information using a `GET_STATUS` message.
<!--is this necessary?-->
A `GET_STATUS` request MAY be sent in response to receiving a `GOSSIP` message, with the following information contained in the body of the message:

```python
{
  'user_agent': 'bytes' ## the human readable name of the client, optionally with its version and other metadata
  'timestamp': 'uint64' ## the current time of the node in milliseconds since epoch
}
```

Any peer MAY provide information about their status and metadata to any other peer. Other peers MAY respond on a best effort basis, if at all.

##### Block Headers

###### `10` GET_BLOCK_HEADERS

Nodes MAY request block headers from other nodes using the `GET_BLOCK_HEADERS` message, with the following information contained in the body of the message:

<!--is this necessary?-->
```python
{
  'start_root': 'bytes32' ## the root hash to start querying from OR
  'start_slot': 'uint64' ## the slot number to start querying from
  'max': 'uint64' ## the max number of elements to return
  'skip': 'uint64' ## the number of elements apart to pick from
  'direction': 'uint8' ## 1 is ascending, 0 is descending direction to query elements
}
```

A `GET_BLOCK_HEADERS` request MAY be sent in response to receiving a `GOSSIP` message.

###### `11` BLOCK_HEADERS

Nodes MAY provide block roots to other nodes using the `BLOCK_HEADERS` message, usually in response to a `GET_BLOCK_HEADERS` message, with the following information contained in the body of the message:

```python
{
  'headers': '[]BeaconBlockHeader'
}
```

##### Block Bodies

###### `12` GET_BLOCK_BODIES

Nodes MAY request block bodies from other nodes using the `GET_BLOCK_BODIES` message, with the following information contained in the body of the message:

```python
{
  'start_root': 'bytes32' ## the root hash to start querying from OR
  'start_slot': 'uint64' ## the slot number to start querying from
  'max': 'uint64' ## the max number of elements to return
  'skip': 'uint64' ## the number of elements apart to pick from
  'direction': 'uint8' ## `0x01` is ascending, `0x00` is descending direction to query elements
}
```

###### `13` BLOCK_BODIES

Nodes MAY provide block roots to other nodes using the `BLOCK_BODIES` message, usually in response to a `GET_BLOCK_BODIES` message, with the following information contained in the body of the message:

```python
{
  'bodies': '[]BeaconBlock'  #SSZ encoded
}
```

##### Attestations

###### `14` GET_ATTESTATION

Nodes MAY request an attestation from other nodes using the `GET_ATTESTATION` message, with the following information contained in the body of the message:

```python
{
    'hash' : 'bytes'  #hash tree root of the attestation object
}
```

###### `15` ATTESTATION

```python
{
    'attestation' : 'Attestation'  #SSZ encoded
}
```

##### Beacon States

###### `16` GET_BEACON_STATES

Nodes MAY request beacon states from other nodes using the `GET_BEACON_STATES` message, with the following information contained in the body of the message:

```python
{
    'hashes' : '[]bytes32'  
}
```

###### `17` BEACON_STATES
Nodes MAY provide beacon states to other nodes using the BEACON_STATES message, usually in response to a GET_BEACON_STATES message, with the following information contained in the body of the message:

```python
{
    'states' : '[]BeaconState'     #SSZ encoded
}
```

### GOSSIP

The gossip protocol allows the propagation of a message to all peers by gossiping the message or its attestation.

For the scope of this specification, only the GOSSIP message is defined, so all peers receive a full copy of the message.

#### Envelope

The message MUST contain the following headers:

```python
{ 
  'method_id': 'uint16' ## the method used in this exchange, as described below
  'topic': 'string' ## the type of message being exchanged
  'timestamp': 'uint64' ## timestamp when the message was sent
  'message_hash': 'bytes32' ## a hash uniquely representing the message contents, with a hash function up to the application
  'hash': 'bytes32' ## hash identifying the data being sent
}
```

The message MAY contain additional headers specified by the application layer (metadata for monitoring and metrics collection, such as the number of hops).

#### GOSSIP Methods

##### `0` GOSSIP

Nodes use `GOSSIP` methods to send data to other nodes in the network.

The `message_hash` header value MUST match the hash of the contents of the body according to a predefined hash function defined by the application.

The `hash` is the [`hash_tree_root`](https://github.com/ethereum/eth2.0-specs/blob/v0.8.1/specs/core/0_beacon-chain.md#hash_tree_root) of the object. The type of object is defined by `topic`, which MUST be `BLOCK` or `ATTESTATION`.

```
EWP 3 1 222 0
{
  "method_id": 0,
  "topic": "BLOCK",
  "timestamp": 1560471980,
  "message_hash": "0x9D686F6262697473206172652074776F20616E6420666F75722066656574",
  "hash": "0x0000000009A4672656E63682070656F706C6520617265207468652062657374"   #hash tree root of BLOCK or ATTESTATION
}
```

#### Other gossip methods

Other gossip methods are explicitly out of scope for this specification at this time.
`IHAVE`, `PRUNE`, `GRAFT` and `IWANT` will be defined if necessary.

### PING

The ping/pong protocol is used to test connections between two peers and ensure the Hobbits implementation is passing conformance tests.

When a `PING` message is received, the node MUST respond with the body of the `PING` message as a `PONG` message.

#### PING methods

##### Ping

```
EWP 3 2 4 32
ping<body bytes>
```

Headers: `ping` as UTF-8 bytes

Body: `random 32 bytes`

##### Pong

```
EWP 3 2 4 32
pong<body bytes>
```

Headers: `pong` as UTF-8 bytes

Body: `32 bytes sent by the ping packet`

## Implementations

As a reference, the following implementations exist:
 - [go-hobbits](https://github.com/renaynay/go-hobbits)
 - [Apache Tuweni](https://github.com/apache/incubator-tuweni)
 - [Typescript](https://github.com/chainsafe/hobbits-ts)

## Acknowlegments

This project would not exist without the dedication of the following individuals. 

* Zak Cole
* Dean Eigenmann
* Matt Elder
* Rene Nayman
* Jonny Rhea
* Preston Van Loon

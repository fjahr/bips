```
  BIP: ?
  Layer: Peer Services
  Title: P2P UTXO Set Sharing
  Authors: Fabian Jahr <fjahr@protonmail.com>
  Status: Draft
  Type: Specification
  Assigned: ?
  License: BSD-2-Clause
  Discussion: 2026-05-06: https://groups.google.com/g/bitcoindev/c/rThmyI8ZN3Q
  Version: 0.6.0
  Requires: 434
```

## Abstract

This BIP defines a P2P protocol extension for sharing full UTXO sets between peers. It introduces
a new service bit `NODE_UTXO_SET` advertising a deterministic schedule of served heights, four new
P2P messages (`getutxotree`, `utxotree`, `getutxoset`, `utxoset`) negotiated via a BIP 434 feature,
and a chunk-hash list anchored to a Merkle root known to the requesting node, enabling per-chunk
verification. This allows bootstrapping nodes to leapfrog to a recent height by obtaining the
required UTXO set directly from the P2P network via mechanisms such as assumeutxo.

## Motivation

The assumeutxo feature (implemented in Bitcoin Core) allows nodes to begin operating from a serialized
UTXO set while validating
historical blocks in the background. However, there is currently no canonical source for obtaining this
data. Users must either generate one themselves from a fully synced node (using `dumptxoutset` in 
Bitcoin Core), or download one from a third party.

By enabling UTXO set sharing over the P2P network, new nodes can obtain the data directly from
peers, removing the dependency on external infrastructure.

## Specification

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be
interpreted as described in RFC 2119.

### Service Bit

| Name | Bit | Description |
|------|-----|-------------|
| `NODE_UTXO_SET` | 14 (0x4000) | The node serves complete UTXO set data for the scheduled heights (see [Scheduled UTXO Set Heights](#scheduled-utxo-set-heights)). |

A node MUST NOT set this bit unless it can serve the UTXO sets at the scheduled heights defined below.
A node signaling `NODE_UTXO_SET` MUST be capable of responding to `getutxotree` and `getutxoset`
requests for the scheduled heights `H1` and `H2`, including the full chunk-hash list and every chunk of
those sets. It MAY additionally serve UTXO sets at other heights.

### Scheduled UTXO Set Heights

To make the `NODE_UTXO_SET` service bit meaningful for peer discovery, advertising nodes serve UTXO
sets at a deterministic schedule of block heights derived from the current tip. All heights refer to
blocks on the active most-work chain.

Let:

* `N` = height of the current chain tip
* `M = N - 2016` (the height as of approximately two weeks ago)
* `K = 14112` (7 difficulty adjustment periods, approximately three months)
* `H1 = M - (M mod K)` (the most recent multiple of `K` that is buried by at least 2016 blocks)
* `H2 = H1 - K` (the preceding multiple of `K`)

A node advertising `NODE_UTXO_SET` MUST be able to serve the UTXO sets at heights `H1` and `H2` as
computed from the current tip. Serving two consecutive scheduled heights guarantees an overlap
window: when a new height becomes `H1`, the previous one remains available as `H2`, so an in-progress
download is not interrupted.

The 2016-block offset ensures a height becomes scheduled only once it is buried by approximately two
weeks, which should make it safe from reorganization and gives serving nodes time to produce the
snapshot before it is requested.

### Feature Negotiation

Support for the messages in this document is negotiated per connection via the BIP 434 `feature`
message, using `featureid` `BIPXXXX` (TODO) and empty `featuredata`. A node implementing these
messages advertises this feature and MUST NOT send any of them to a peer that has not.

Advertising the feature indicates only that a node implements the protocol while `NODE_UTXO_SET`
indicates it additionally serves the scheduled heights. A node setting `NODE_UTXO_SET` MUST also
advertise the feature.

### Data Structures

#### Serialized UTXO Set

This proposal uses version 3 of the `dumptxoutset` serialization. All fixed-width integers are
unsigned and little-endian. `compact_size` is Bitcoin's minimally encoded CompactSize: a one-byte
value below 253, otherwise a marker byte 253, 254, or 255 followed by a little-endian 2-, 4-, or
8-byte value, respectively. Fixed-length byte fields have no prefix. Variable-length arrays and byte
strings are prefixed with their CompactSize element or byte count.

**Header (51 bytes):**

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `magic` | `bytes` | 5 | `0x7574786fff` (ASCII `utxo` + `0xff`). |
| `version` | `uint16_t` | 2 | Format version, MUST be 3. |
| `network_magic` | `bytes` | 4 | Network message start bytes. |
| `base_blockhash` | `uint256` | 32 | Block hash of the UTXO set. |
| `coins_count` | `uint64_t` | 8 | Total number of coins (UTXOs) in the set. |

The base height is obtained by looking up `base_blockhash` in the block-header chain; it is not
serialized separately. This retains the header layout of version 2 of `dumptxoutset`.

**Body (framed coin data):**

The header is followed by zero or more data chunks. Each data chunk is serialized as
`compact_size(payload_length) || payload`, with `1 <= payload_length <= 262144` (256 KiB).
The prefix is part of the chunk bytes transferred and hashed. No coin or group header crosses a
chunk boundary. An empty UTXO set has no data chunks.

Each payload contains one or more transaction group fragments. Each fragment is entirely contained
in one chunk; its `num_coins` counts only the coins in that fragment, not coins in other fragments
with the same txid:

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `txid` | `bytes` | 32 | Transaction hash in its serialized byte order. |
| `num_coins` | `compact_size` | 1–9 | Positive number of coins in this group fragment. |

For each coin in the fragment:

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `vout_index` | `compact_size` | 1–9 | Output index, at most `0xfffffffe`. |
| `coin` | `Coin` | variable | Serialized coin (varint-encoded code for height/coinbase, then compressed txout), as in version 2 of `dumptxoutset`. |

The complete sequence of coins MUST be strictly increasing by outpoint: compare the 32 serialized
txid bytes lexicographically, treating each byte as unsigned; for equal txids compare output indices
numerically. This is not the order of displayed txid hex strings, nor a comparison of encoded index
bytes. Duplicates are forbidden, including across chunk boundaries. Exactly `coins_count` coins
MUST be present, with no trailing bytes.

To obtain deterministic chunk boundaries, writers MUST pack coins greedily in this order. For each
txid, put the largest positive number of its remaining coins that fits in the current payload,
including the txid, group count, output indices and serialized coins. If none fits, finish the current
chunk and start a new one. Continue a transaction in a new chunk by repeating its txid and writing
the count for the new fragment. After finishing a txid, try the next txid in the same chunk. Finish the last
chunk after the last coin. A single serialized coin with its group header MUST fit in 256 KiB.

Each data chunk can consequently be decoded without downloading any other data chunk. Obtaining
all unspent outputs for a transaction may require several chunks, but decoding and inserting any
individual coin does not. The chunk's first and last outpoints bound its contents in the global
order, allowing a future implementation to locate an outpoint by binary search over authenticated chunks. No seek index or just-in-time validation
mechanism is specified here.

#### Chunk Merkle Tree

Chunk 0 consists of the 51-byte snapshot header. Chunks 1 onwards are the framed data chunks above,
in file order. Concatenating all chunk bytes reconstructs the snapshot file exactly.

Each chunk descriptor consists of its `hash = SHA256d(chunk_bytes)` and its byte `size` as a
`uint32_t`. The leaf hash is `SHA256d(0x00 || uint32_le(size) || hash)`. Interior nodes are
`SHA256d(0x01 || left_child || right_child)`; when a level has an odd number of nodes, its last node
is promoted unchanged. A one-leaf tree has that leaf as its root. Hashes are concatenated in their
raw 32-byte digest representation, without reversing bytes. The domain prefixes distinguish leaves
from interior nodes; committing to each size authenticates the file offsets derived from the list.

The ordered descriptors are delivered in a single `utxotree` response. A node verifies the list by
recomputing its root and comparing against its known root for this snapshot and format version.
`SHA256d(x)` denotes `SHA256(SHA256(x))`.

### Messages

#### `getutxotree`

Sent to request the chunk-hash list for a specific UTXO set.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `block_hash` | `uint256` | 32 | Block hash identifying the requested UTXO set. |

A node that has advertised `NODE_UTXO_SET` and can serve the requested UTXO set MUST respond with
`utxotree`. If the serving node cannot fulfill the request, it MUST NOT respond. The requesting
node SHOULD apply a reasonable timeout and try another peer.

#### `utxotree`

Sent in response to `getutxotree`, delivering the full ordered chunk-descriptor list.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `block_hash` | `uint256` | 32 | Block hash this data corresponds to. |
| `version` | `uint16_t` | 2 | Format version of the serialized UTXO set. |
| `chunks` | `chunk_descriptor[]` | 1–5 + 36 × N | CompactSize-prefixed list of N descriptors, each a 32-byte `hash` followed by a 4-byte `size`. |

There MUST be between 1 and 100,000 descriptors. The first size MUST be 51. Subsequent sizes MUST
include the CompactSize payload-length prefix and MUST match the framed-chunk format above
(at most 262149 bytes). The sum of all sizes is the snapshot file length; a chunk's file offset is
the sum of the preceding sizes. A receiver MUST reject unsupported format versions before
requesting chunks. The header version MUST agree with this message's `version`.

Upon receiving a `utxotree` message, the requesting node MUST recompute the Merkle root from
`chunks` and compare it against the Merkle root it knows for the corresponding UTXO set. If
the roots do not match, the node MUST discard the response and MUST disconnect the peer.

#### `getutxoset`

Sent to request a single chunk of UTXO set data. The requesting node MUST have received a `utxotree`
for the corresponding UTXO set (from any peer) before sending this message.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `block_hash` | `uint256` | 32 | Block hash identifying the requested UTXO set. |
| `chunk_index` | `uint32_t` | 4 | Zero-based index of the requested chunk. |

If the serving node cannot fulfill the request, it MUST NOT respond. The requesting node SHOULD apply
a reasonable timeout and try another peer.

#### `utxoset`

Sent in response to `getutxoset`, delivering one chunk.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `block_hash` | `uint256` | 32 | Block hash this data corresponds to. |
| `chunk_index` | `uint32_t` | 4 | Zero-based index of this chunk. |
| `data` | `bytes` | variable | CompactSize-prefixed chunk bytes, of the size in `chunks[chunk_index]`. |

The transfer is receiver-driven: the requesting node sends one `getutxoset` per chunk. Chunks MAY be
requested in any order and from different peers.

Upon receiving a `utxoset` message, the node MUST compute `SHA256d(data)` and compare it against
`chunks[chunk_index].hash` from the accepted `utxotree`. The byte length MUST also equal
`chunks[chunk_index].size`. If either check fails, the node MUST discard the chunk and MUST
disconnect the peer. The message byte-vector prefix is excluded from the chunk bytes, hash, and size.
A node SHOULD also disconnect
a peer that sends a `utxoset` message with fields (`chunk_index`, `block_hash`) that do not match
the outstanding request.

After all chunks have been received, the node SHOULD parse the reassembled UTXO set against the
serialized UTXO set format to confirm it is well-formed.

### Protocol Flow

1. The requesting node identifies peers advertising `NODE_UTXO_SET`.
2. The requesting node sends `getutxotree` for the desired block hash to one of these peers, or to
   several peers to corroborate the Merkle root by agreement if no trusted root is available.
3. The peer or peers respond with `utxotree`. The requesting node verifies each response by
   recomputing the Merkle root and comparing it against a value it knows for the given UTXO set,
   either from a trusted source or from agreement among multiple peers. A single accepted `utxotree`
   can be used as the basis for all subsequent chunk requests for this UTXO set, regardless of
   which peer those chunks are fetched from.
4. The requesting node downloads chunks via `getutxoset`/`utxoset` exchanges, verifying each chunk
   against its entry in the accepted `utxotree` on receipt. On verification failure the peer is
   disconnected and download continues from another peer without losing already-verified chunks.
5. After all chunks are received, the node parses the reassembled UTXO set against the serialized
   UTXO set format to confirm that it is well-formed.

Serving nodes are free to limit the number of concurrent and repeated transfers per peer at their own
discretion to manage resource consumption.

## Rationale

**Usage of service bit 14:** Service bits allow selective peer discovery through
DNS seeds and addr relay. Bit 14 is chosen because bits 12 and 13 are reserved by the
Utreexo proposal (BIP 183 draft).

**Direct request model:** Peers signal availability of UTXO sets via the `NODE_UTXO_SET`
service bit; the requesting node identifies the desired UTXO set by block hash when sending
`getutxotree`. The serving node responds only if it can serve that specific UTXO set.

**Per-chunk verification:** The chunk-hash list returned in `utxotree` enables each chunk to be verified
by direct lookup against the accepted list as it arrives, allowing immediate detection of corrupt data,
peer switching without data loss, and parallel download from multiple peers. Deterministic
serialization lets honest nodes agree on the Merkle root.

**256 KiB payload bound:** Smaller chunks reduce how much unrelated data must be fetched to decode
a particular coin. Splitting large transaction groups, while repeating only the txid and count,
keeps each chunk independently decodable without repeating a txid for every coin. Greedy packing
preserves a single canonical serialization. The size list is sufficient to reconstruct file offsets;
no separate seek index is required.

**One descriptor-list message:** At 100,000 descriptors the response is 3,600,039 bytes, below the
4,000,000-byte P2P message bound. For a 9 GB snapshot with nearly full chunks the list is about
1.24 MB. This version limits the number of chunks, and thus the size of snapshots it can serve;
larger future sets require a protocol revision, such as paginated descriptors.

**Snapshot version 3:** Framing and splitting transaction groups changes the file body, so version 2
files and their chunk commitments cannot be reused. The header layout and coin encoding are
retained. Numeric output-index ordering is explicit and independent of a database's encoded-key
iteration order.

**Relationship to BIP 64:** BIP 64 defined a protocol for querying individual UTXOs by outpoint and is
now closed. This BIP addresses a different use case: bulk transfer of the entire UTXO set for node
bootstrapping.

## Backwards Compatibility

The P2P extension is backward compatible. Peers that do not implement it ignore the new service bit
and never issue the new messages. Feature negotiation follows BIP 434, so peers that do not
recognize the feature ignore it. Snapshot files use a new version; version 2 readers must reject
version 3 files. Chunk commitments must be regenerated from version 3 snapshots.

## Reference Implementation

[Bitcoin Core implementation pull request](https://github.com/bitcoin/bitcoin/pull/35054)

## Acknowledgements

Thanks to Anthony Towns for suggesting that the requesting node fetch the full chunk-hash list up front
via the `getutxotree`/`utxotree` exchange rather than per-chunk Merkle proofs, using the Merkle root as
the sole trust anchor in place of a separate serialized hash, dropping the redundant `num_chunks` field,
and the deterministic schedule of served heights.

Thanks also to Murch for catching the service-bit collision with the Utreexo proposal, raising
the Merkle tree malleability concern behind promoting odd nodes unchanged rather than duplicating them,
and prompting the Backwards Compatibility section; stickies-v for raising the data-availability concern
that motivated a fixed schedule of served heights; Luke Dashjr for arguing against a separate discovery
step on privacy grounds and suggesting the serialization format version be carried; and Daniela Brozzoni
for helping make the peer-disconnection rules consistent.

## Copyright

This BIP is made available under the terms of the 2-clause BSD license. See
https://opensource.org/license/BSD-2-Clause for more information.

## Changelog

* __0.6.0__ (2026-10-17):
    * Defined version 3 snapshots with 256 KiB, coin-aligned, independently decodable chunks
    * Specified global outpoint ordering and deterministic packing, including split transaction groups
    * Added authenticated chunk sizes and domain-separated Merkle hashes
    * Bounded the single descriptor-list response to 100,000 chunks
* __0.5.0__ (2026-06-03):
    * Defined a deterministic schedule of served heights for the `NODE_UTXO_SET` service bit
    * Added BIP 434 feature negotiation to signal support for the protocol messages
    * Added Acknowledgements section
* __0.4.0__ (2026-05-18):
    * Removed `num_chunks` from `utxotree`
* __0.3.0__ (2026-05-17):
    * Moved service bit from 12 to 14 to avoid collision with the Utreexo proposal (BIP 183 draft)
    * Changed Merkle tree construction: odd nodes are promoted unchanged rather than duplicated
* __0.2.0__ (2026-05-04):
    * Dropped discovery before download approach, instead request the chunk-hash list via `getutxotree`/`utxotree`
    * Dropped per-chunk Merkle proofs; chunks verified directly against the chunk-hash list
    * Dropped `height` from requests (`block_hash` is the sole identifier); added format `version` to `utxotree`
    * Dropped references to the serialized hash; the Merkle root is the sole integrity check
* __0.1.0__ (2026-04-10):
    * Initial draft

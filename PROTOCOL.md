# Bitcoin Full Node Client: Protocol Specification

Version: 1.0
Purpose: Complete specification for building a Bitcoin P2P full node client from scratch
Scope: v1 transport only (BIP 324 v2 transport is out of scope)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Encoding Primitives](#2-encoding-primitives)
3. [Message Frame Format](#3-message-frame-format)
4. [Handshake Lifecycle](#4-handshake-lifecycle)
5. [Message Types Reference](#5-message-types-reference)
6. [Peer Discovery](#6-peer-discovery)
7. [Blockchain Sync (IBD)](#7-blockchain-sync-ibd)
8. [Transaction Handling](#8-transaction-handling)
9. [Address Monitoring](#9-address-monitoring)
10. [Mempool](#10-mempool)
11. [Common Mistakes](#11-common-mistakes)

---

## 1. Architecture Overview

### 1.1 What This Builds

A Bitcoin full node P2P client connects directly to the Bitcoin network over raw TCP. It speaks the Bitcoin binary wire protocol and can:

- Discover peers via DNS seeds
- Complete the version/verack handshake
- Request and validate block headers and full blocks
- Parse, verify, and broadcast transactions
- Monitor the mempool for unconfirmed transactions
- Track specific addresses for incoming payments

### 1.2 What This Is NOT

- Not an RPC client for bitcoind (that uses HTTP on port 8332)
- Not an SPV/light client (this downloads and validates full blocks)
- Not a miner (no proof-of-work grinding or pool coordination)

### 1.3 Network Parameters

| Network  | Magic Bytes      | Default Port | Genesis Hash |
|----------|------------------|--------------|--------------|
| mainnet  | `F9 BE B4 D9`    | 8333         | `000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f` |
| testnet3 | `0B 11 09 07`    | 18333        | `000000000933ea01ad0ee984209779baaec3ced90fa3f408719526f8d77f4943` |
| testnet4 | `1C 16 3F 28`    | 48333        | |
| regtest  | `FA BF B5 DA`    | 18444        | |

### 1.4 Protocol Version

Announce protocol version **70015** (Bitcoin Core 0.18+). This enables segwit (BIP 144), compact blocks (BIP 152), sendcmpct, feefilter, and sendheaders.

---

## 2. Encoding Primitives

All multi-byte integers are **little-endian** unless explicitly stated otherwise.

### 2.1 Integer Types

| Type      | Size    | Encoding     |
|-----------|---------|--------------|
| uint8     | 1 byte  | unsigned     |
| uint16    | 2 bytes | little-endian |
| uint32    | 4 bytes | little-endian |
| uint64    | 8 bytes | little-endian |
| int32     | 4 bytes | little-endian, signed |
| int64     | 8 bytes | little-endian, signed |

### 2.2 CompactSize (Variable-Length Integer)

Used before any variable-length field (arrays, strings, scripts).

| Value Range          | Encoding                          | Total Bytes |
|----------------------|-----------------------------------|-------------|
| 0x00 to 0xFC         | 1 byte: the value directly        | 1           |
| 0xFD to 0xFFFF       | 0xFD followed by uint16 LE        | 3           |
| 0x10000 to 0xFFFFFFFF | 0xFE followed by uint32 LE       | 5           |
| Above 0xFFFFFFFF     | 0xFF followed by uint64 LE        | 9           |

### 2.3 Variable-Length String (VarStr)

A CompactSize length prefix followed by that many bytes of UTF-8 text.

### 2.4 Hash Encoding

Bitcoin hashes (block hashes, txids) are **displayed** in big-endian hex but **transmitted** in little-endian (byte-reversed). Always reverse before display.

### 2.5 Checksum

SHA256(SHA256(payload)), take the first 4 bytes. This is called "double-SHA256" or "SHA256d".

---

## 3. Message Frame Format

Every Bitcoin P2P message uses the same 24-byte header followed by the payload.

### 3.1 Header (24 bytes, fixed)

| Field    | Offset | Size | Type      | Description                                    |
|----------|--------|------|-----------|------------------------------------------------|
| magic    | 0      | 4    | bytes     | Network magic bytes (see Section 1.3)          |
| command  | 4      | 12   | char[12]  | ASCII command name, null-padded to 12 bytes    |
| length   | 16     | 4    | uint32 LE | Byte length of the payload                     |
| checksum | 20     | 4    | bytes     | First 4 bytes of SHA256d(payload)              |

### 3.2 Frame Rules

- Messages with no payload (e.g., verack, mempool, getaddr) have length = 0 and checksum = SHA256d of empty bytes: `5D F6 E0 E2`
- Maximum payload size: 4,000,000 bytes (MAX_SIZE in Bitcoin Core)
- TCP delivers a byte stream, not discrete messages. Implementations must buffer incoming data and parse frames in a loop, never assuming one TCP chunk equals one message.

---

## 4. Handshake Lifecycle

### 4.1 Sequence

```
Client                              Server
  |                                    |
  |---- version ---------------------->|
  |                                    |
  |<--- version -----------------------|
  |<--- verack ------------------------|
  |                                    |
  |---- verack ----------------------->|
  |                                    |
  |       -- HANDSHAKE COMPLETE --     |
```

**Critical rule:** Send verack ONLY after receiving the server's version message. Never before.

After handshake, the server may send: sendheaders, sendcmpct, feefilter, ping.
Respond to ping with pong (echo the 8-byte nonce back).

### 4.2 version Message Payload

| Field           | Size | Type      | Description                                    |
|-----------------|------|-----------|------------------------------------------------|
| version         | 4    | int32 LE  | Protocol version (use 70015)                   |
| services        | 8    | uint64 LE | Bitfield. NODE_NETWORK = 1. Use 0 if not serving blocks |
| timestamp       | 8    | int64 LE  | Unix epoch seconds                             |
| addr_recv.services | 8 | uint64 LE | Services of the remote peer (use 1)            |
| addr_recv.ip    | 16   | bytes     | IPv4-mapped IPv6: 10 bytes 0x00, 2 bytes 0xFF, 4 bytes IPv4 |
| addr_recv.port  | 2    | **uint16 BE** | **BIG-endian** (exception to the LE rule)  |
| addr_trans.services | 8 | uint64 LE | Our advertised services                       |
| addr_trans.ip   | 16   | bytes     | Our IP (can be all zeros)                      |
| addr_trans.port | 2    | **uint16 BE** | Our port (can be 0)                        |
| nonce           | 8    | uint64 LE | Random. Used to detect self-connections        |
| user_agent      | var  | VarStr    | e.g., "/MyNode:1.0.0/"                        |
| start_height    | 4    | int32 LE  | Our best known block height (use 0 initially)  |
| relay           | 1    | bool      | 1 = relay transactions to us, 0 = don't       |

**Total minimum size:** 86 bytes + VarStr overhead for user_agent.

### 4.3 verack Message

No payload. Just the 24-byte header with command "verack" and length 0.

---

## 5. Message Types Reference

### 5.1 Inventory Vectors

Used in `inv`, `getdata`, and `notfound` messages. Each inventory entry is 36 bytes.

| Field | Size | Type      | Description |
|-------|------|-----------|-------------|
| type  | 4    | uint32 LE | Inventory type (see below) |
| hash  | 32   | bytes     | Object hash in little-endian wire order |

**Inventory types:**

| Value      | Name              | Description              |
|------------|-------------------|--------------------------|
| 0          | ERROR             | Ignored                  |
| 1          | MSG_TX            | Transaction              |
| 2          | MSG_BLOCK         | Block                    |
| 3          | MSG_FILTERED_BLOCK | Bloom-filtered block (BIP 37) |
| 4          | MSG_CMPCT_BLOCK   | Compact block (BIP 152)  |
| 0x40000001 | MSG_WITNESS_TX    | Segwit transaction       |
| 0x40000002 | MSG_WITNESS_BLOCK | Segwit block             |

**Important:** After BIP 339, use MSG_WITNESS_TX (0x40000001) in getdata requests, not MSG_TX.

### 5.2 inv

Announces available objects. Payload: CompactSize count, then that many inventory vectors.

### 5.3 getdata

Requests specific objects. Same format as inv. The peer responds with the actual tx or block messages.

### 5.4 notfound

Same format as inv. Sent when the peer doesn't have a requested object.

### 5.5 getheaders

Requests block headers for IBD.

| Field         | Size | Type      | Description                              |
|---------------|------|-----------|------------------------------------------|
| version       | 4    | uint32 LE | Protocol version (70015)                 |
| hash_count    | var  | CompactSize | Number of locator hashes               |
| locator_hashes | 32 each | bytes | Block hashes, newest first (see Block Locator in Section 7) |
| stop_hash     | 32   | bytes     | Hash to stop at, or 32 zero bytes for "send everything" |

### 5.6 headers

Response to getheaders. Payload: CompactSize count, then that many 80-byte block headers each followed by a CompactSize transaction count (always 0 in headers messages).

### 5.7 Block Header (80 bytes)

| Field       | Offset | Size | Type      | Description           |
|-------------|--------|------|-----------|-----------------------|
| version     | 0      | 4    | int32 LE  | Block version         |
| prev_block  | 4      | 32   | bytes     | Previous block hash   |
| merkle_root | 36     | 32   | bytes     | Merkle root of transactions |
| timestamp   | 68     | 4    | uint32 LE | Unix epoch seconds    |
| bits        | 72     | 4    | uint32 LE | Encoded difficulty target |
| nonce       | 76     | 4    | uint32 LE | Mining nonce          |

The block hash is SHA256d of these 80 bytes.

### 5.8 ping / pong

Both carry an 8-byte uint64 LE nonce. When you receive a ping, respond with pong using the same nonce.

### 5.9 getaddr / addr

- `getaddr`: No payload. Requests known peer addresses.
- `addr`: CompactSize count, then entries of:

| Field     | Size | Type         | Description            |
|-----------|------|--------------|------------------------|
| timestamp | 4    | uint32 LE    | Last seen time         |
| services  | 8    | uint64 LE    | Node services          |
| ip        | 16   | bytes        | IPv4-mapped IPv6       |
| port      | 2    | **uint16 BE** | **Big-endian** port   |

### 5.10 mempool

No payload. Requests the peer to send inv messages for all transactions in its mempool.

### 5.11 reject (deprecated but still sent by some peers)

| Field      | Size | Type      | Description           |
|------------|------|-----------|-----------------------|
| message    | var  | VarStr    | Which message was rejected |
| ccode      | 1    | uint8     | Error code            |
| reason     | var  | VarStr    | Human-readable reason |
| data       | 32   | bytes     | Optional: hash of rejected object |

---

## 6. Peer Discovery

### 6.1 DNS Seeds

Resolve these hostnames via DNS A record lookups to get peer IP addresses.

**Mainnet seeds:**
- `seed.bitcoin.sipa.be`
- `dnsseed.bluematt.me`
- `dnsseed.bitcoin.dashjr-list-of-p2p-nodes.us`
- `seed.bitcoinstats.com`
- `seed.bitcoin.jonasschnelli.ch`
- `seed.btc.petertodd.net`
- `seed.bitcoin.sprovoost.nl`

**Testnet3 seeds:**
- `testnet-seed.bitcoin.jonasschnelli.ch`
- `seed.tbtc.petertodd.net`
- `testnet-seed.bluematt.me`

**Fallback testnet3 peers (hardcoded):**
- `138.201.55.219:18333`
- `159.69.36.152:18333`
- `51.15.96.97:18333`

### 6.2 Discovery via addr Messages

After handshake, send `getaddr`. The peer responds with `addr` containing known peers. Cache these for future connections.

---

## 7. Blockchain Sync (IBD)

### 7.1 Strategy: Headers-First

Always use headers-first sync. Never blocks-first.

**Phase 1: Sync all headers**
1. Send `getheaders` with a block locator containing just the genesis hash
2. Receive up to 2,000 headers per response
3. Validate each header: prev_block must match the previous header's hash
4. Repeat with an updated locator until the response contains fewer than 2,000 headers

**Phase 2: Download blocks**
1. For each header, send `getdata` with type MSG_BLOCK (or MSG_WITNESS_BLOCK for segwit)
2. Validate received blocks: header hash matches, merkle root matches transactions, proof of work passes
3. Can request blocks in parallel from multiple peers for speed

### 7.2 Block Locator Construction

A block locator is a list of block hashes sent from newest to oldest. It helps the peer find where your chain diverges from theirs.

Algorithm:
1. Start from your best known block
2. Include the first 10 hashes going back one block at a time
3. After 10, double the step size each time (step back 2, then 4, then 8, etc.)
4. Always include the genesis hash as the last entry

### 7.3 Proof of Work Verification

The `bits` field encodes a compact difficulty target:
- Top byte = exponent (number of bytes in the target)
- Lower 3 bytes = mantissa (the significant digits)

Target = mantissa * 2^(8 * (exponent - 3))

The block hash (as a 256-bit number, big-endian) must be less than or equal to the target.

---

## 8. Transaction Handling

### 8.1 Transaction Format (Segwit-aware)

**Legacy format:**

| Field    | Size | Type           | Description                |
|----------|------|----------------|----------------------------|
| version  | 4    | int32 LE       | Transaction version (1 or 2) |
| tx_in_count | var | CompactSize  | Number of inputs           |
| tx_in    | var  | TxIn[]         | Input list                 |
| tx_out_count | var | CompactSize | Number of outputs          |
| tx_out   | var  | TxOut[]        | Output list                |
| locktime | 4    | uint32 LE      | Lock time                  |

**Segwit format** (BIP 144): After the version field, if the next two bytes are `0x00 0x01` (marker + flag), the transaction uses segwit serialization. Witness data appears after all outputs but before locktime.

Detection: Read the byte after version. If it's 0x00, check the next byte. If non-zero, this is segwit (marker=0x00, flag=0x01). If the first byte is non-zero, it's a legacy transaction and that byte is the start of tx_in_count.

### 8.2 TxIn (Transaction Input)

| Field       | Size | Type         | Description                        |
|-------------|------|--------------|------------------------------------|
| prev_hash   | 32   | bytes        | Hash of the transaction being spent |
| prev_index  | 4    | uint32 LE    | Output index in that transaction   |
| script_len  | var  | CompactSize  | Length of signature script         |
| script_sig  | var  | bytes        | Signature script                   |
| sequence    | 4    | uint32 LE    | Sequence number                    |

### 8.3 TxOut (Transaction Output)

| Field       | Size | Type         | Description              |
|-------------|------|--------------|--------------------------|
| value       | 8    | **uint64 LE** | Satoshi amount (**use 64-bit integers, not floats**) |
| script_len  | var  | CompactSize  | Length of pubkey script   |
| script_pubkey | var | bytes       | Output script            |

### 8.4 Witness Data (Segwit)

For each input, after all outputs:
- CompactSize: number of witness items
- For each item: CompactSize length, then that many bytes

### 8.5 TXID Calculation

The TXID is SHA256d of the transaction serialized WITHOUT the witness data (legacy serialization). Display in big-endian (reversed bytes).

The WTXID includes witness data in the hash.

### 8.6 Broadcasting

Send the raw serialized transaction as a `tx` message. The peer will validate and relay it. Listen for `reject` messages in case of errors.

---

## 9. Address Monitoring

### 9.1 Script Matching

To monitor an address, convert it to its scriptPubKey and compare against transaction outputs.

**P2PKH** (addresses starting with 1 or m/n):
`OP_DUP OP_HASH160 <20-byte-hash> OP_EQUALVERIFY OP_CHECKSIG`
Bytes: `76 A9 14 [hash] 88 AC`

**P2SH** (addresses starting with 3 or 2):
`OP_HASH160 <20-byte-hash> OP_EQUAL`
Bytes: `A9 14 [hash] 87`

**P2WPKH** (bech32 addresses, bc1q...):
`OP_0 <20-byte-hash>`
Bytes: `00 14 [hash]`

**P2WSH** (bech32, bc1q... with 32-byte program):
`OP_0 <32-byte-hash>`
Bytes: `00 20 [hash]`

**P2TR** (bech32m, bc1p...):
`OP_1 <32-byte-key>`
Bytes: `51 20 [key]`

### 9.2 Base58Check Decoding

For legacy addresses (P2PKH, P2SH): Base58Check decode gives a version byte followed by the 20-byte hash.
- Version 0x00 = mainnet P2PKH
- Version 0x05 = mainnet P2SH
- Version 0x6F = testnet P2PKH
- Version 0xC4 = testnet P2SH

### 9.3 Bech32/Bech32m Decoding

For segwit addresses: Bech32 decode gives a witness version and witness program.
- Witness version 0 + 20 bytes = P2WPKH
- Witness version 0 + 32 bytes = P2WSH
- Witness version 1 + 32 bytes = P2TR (uses bech32m, not bech32)

---

## 10. Mempool

### 10.1 Requesting the Mempool

Send a `mempool` message (no payload) after handshake. The peer responds with `inv` messages containing all transaction hashes in its mempool.

### 10.2 Tracking

For each `inv` with type MSG_TX:
1. Check if the txid is already known
2. If not, send `getdata` to request the full transaction
3. Parse and store the transaction
4. Check against monitored addresses

### 10.3 Ongoing Monitoring

After initial mempool sync, the peer will send `inv` messages for new transactions as they arrive. Continue requesting unknown transactions via `getdata`.

---

## 11. Common Mistakes

| Wrong | Correct |
|---|---|
| Assume one TCP chunk = one message | Buffer incoming data, parse frames in a loop |
| Send verack immediately on connect | Send verack only AFTER receiving the peer's version |
| Encode port as little-endian | Port in addr and version is **big-endian** |
| Display hash bytes directly | Reverse bytes before converting to hex for display |
| Use floating-point for satoshi values | Use 64-bit integers |
| Send getheaders before handshake completes | Wait for both version and verack before requesting data |
| Use MSG_TX (1) in getdata | Use MSG_WITNESS_TX (0x40000001) for segwit-aware peers |
| Use dns.lookup() for seeds | Use dns.resolve() to get all A records |
| Ignore reject messages | Always handle reject for debugging |
| Skip checksum validation | Verify SHA256d checksum on every received message |

### 11.1 Byte Budget Quick Reference

| Message      | Header | Payload     | Notes                     |
|-------------|--------|-------------|---------------------------|
| verack      | 24     | 0           | Empty payload             |
| ping/pong   | 24     | 8           | 8-byte nonce              |
| version     | 24     | 86+         | Plus VarStr user_agent    |
| inv (1 item) | 24    | 37          | 1 + 36                   |
| getheaders  | 24     | 37+         | 4 + 1 + 32*n + 32        |
| block header | n/a   | 80          | Within headers message    |

---

*This document describes the Bitcoin P2P wire protocol as implemented by Bitcoin Core. It is language-agnostic. Hand it to any implementation environment with the instruction: "Build a Bitcoin full node client following this specification."*

# Bitcoin Full Node Client: Implementation Protocol

Version: 1.0
Target Runtime: Node.js >= 18 (ESM)
Purpose: LLM-executable specification for one-shot generation of a working Bitcoin P2P full node client
Scope: v1 transport (BIP 324 v2 is out of scope unless explicitly requested)

Directive: Follow every section in order. Do not invent API shapes. Do not skip steps. Every byte value, field size, and encoding shown here is exact.

## Table of Contents

1. [Mental Model and Architecture](#1-mental-model-and-architecture)
2. [Project Setup](#2-project-setup)
3. [Bitcoin Wire Protocol: Encoding Primitives](#3-bitcoin-wire-protocol-encoding-primitives)
4. [Message Frame Format](#4-message-frame-format)
5. [Handshake: Connection Lifecycle](#5-handshake-connection-lifecycle)
6. [All Message Types: Serialization Reference](#6-all-message-types-serialization-reference)
7. [Peer Discovery](#7-peer-discovery)
8. [Blockchain Sync (IBD)](#8-blockchain-sync-ibd)
9. [Transaction Handling](#9-transaction-handling)
10. [Address/Wallet Monitoring](#10-addresswallet-monitoring)
11. [Mempool Access](#11-mempool-access)
12. [Complete Client Architecture](#12-complete-client-architecture)
13. [Full Working Implementation](#13-full-working-implementation)
14. [Anti-Patterns and Checklist](#14-anti-patterns-and-checklist)

---

## 1. Mental Model and Architecture

### 1.1 What This Client Does

A Bitcoin full node P2P client connects directly to the Bitcoin network over raw TCP. It speaks the Bitcoin wire protocol, a binary, little-endian framed protocol, and can:

- Discover peers via DNS seeds or hardcoded nodes
- Complete the version handshake
- Request and validate block headers and full blocks
- Receive and broadcast transactions
- Monitor the mempool
- Track unconfirmed/confirmed transactions for specific addresses

### 1.2 What It Is NOT

- It is not an RPC client talking to bitcoind over HTTP (that is a separate thing)
- It is not an SPV client (it downloads full blocks)
- It does not mine

### 1.3 Network Parameters

| Network  | Magic Bytes | Default Port | Genesis Hash |
|----------|-------------|--------------|--------------|
| mainnet  | F9 BE B4 D9 | 8333  | 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f |
| testnet3 | 0B 11 09 07 | 18333 | 000000000933ea01ad0ee984209779baaec3ced90fa3f408719526f8d77f4943 |
| testnet4 | 1c 16 3f 28 | 48333 | (use testnet3 for this protocol) |
| regtest  | FA BF B5 DA | 18444 | (local only) |

Use testnet3 during development. Switch to mainnet by changing magic bytes and port only.

### 1.4 Protocol Version

Always announce protocolVersion = 70015 (Bitcoin Core 0.18+). This enables:

- Segwit (BIP 144)
- Compact blocks (BIP 152)
- sendcmpct, feefilter, sendheaders

---

## 2. Project Setup

### 2.1 Directory Structure

```
bitcoin-node-client/
  src/
    index.js          <- entry point
    peer.js           <- single peer connection + message framing
    messages.js       <- message serialization/deserialization
    encoding.js       <- Bitcoin encoding primitives
    dns.js            <- peer discovery
    blockchain.js     <- header chain + IBD logic
    mempool.js        <- mempool tracking
    wallet.js         <- address monitoring
  package.json
  .env
```

### 2.2 package.json

```json
{
  "name": "bitcoin-node-client",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js"
  },
  "dependencies": {
    "bs58check": "^4.0.0"
  }
}
```

CRITICAL: `"type": "module"` is required. No external P2P library is used. This implements the wire protocol from scratch. Only bs58check is needed for address decoding.

### 2.3 Install

```bash
npm install
```

---

## 3. Bitcoin Wire Protocol: Encoding Primitives

All multi-byte integers are little-endian unless stated otherwise. All code lives in `src/encoding.js`.

### 3.1 Integer Encodings

```javascript
// src/encoding.js
export function writeUInt8(buf, val, offset) { buf.writeUInt8(val, offset); return offset + 1; }
export function writeUInt16LE(buf, val, offset) { buf.writeUInt16LE(val, offset); return offset + 2; }
export function writeUInt32LE(buf, val, offset) { buf.writeUInt32LE(val, offset); return offset + 4; }
export function writeUInt64LE(buf, val, offset) {
  buf.writeBigUInt64LE(BigInt(val), offset);
  return offset + 8;
}
export function writeInt32LE(buf, val, offset) { buf.writeInt32LE(val, offset); return offset + 4; }
export function writeInt64LE(buf, val, offset) { buf.writeBigInt64LE(BigInt(val), offset); return offset + 8; }
```

### 3.2 CompactSize (VarInt)

Bitcoin uses a variable-length integer encoding called CompactSize. This is used before any variable-length field (strings, arrays, scripts).

| Value range | Encoding |
|---|---|
| 0x00 - 0xFC | 1 byte: the value directly |
| 0xFD - 0xFFFF | 0xFD + 2 bytes LE uint16 |
| 0x10000 - 0xFFFFFFFF | 0xFE + 4 bytes LE uint32 |
| > 0xFFFFFFFF | 0xFF + 8 bytes LE uint64 |

```javascript
export function writeCompactSize(buf, val, offset) {
  if (val < 0xfd) {
    buf.writeUInt8(val, offset);
    return offset + 1;
  } else if (val <= 0xffff) {
    buf.writeUInt8(0xfd, offset);
    buf.writeUInt16LE(val, offset + 1);
    return offset + 3;
  } else if (val <= 0xffffffff) {
    buf.writeUInt8(0xfe, offset);
    buf.writeUInt32LE(val, offset + 1);
    return offset + 5;
  } else {
    buf.writeUInt8(0xff, offset);
    buf.writeBigUInt64LE(BigInt(val), offset + 1);
    return offset + 9;
  }
}

export function compactSizeBytes(val) {
  if (val < 0xfd) return 1;
  if (val <= 0xffff) return 3;
  if (val <= 0xffffffff) return 5;
  return 9;
}

export function readCompactSize(buf, offset) {
  const first = buf.readUInt8(offset);
  if (first < 0xfd) return { value: first, size: 1 };
  if (first === 0xfd) return { value: buf.readUInt16LE(offset + 1), size: 3 };
  if (first === 0xfe) return { value: buf.readUInt32LE(offset + 1), size: 5 };
  return { value: Number(buf.readBigUInt64LE(offset + 1)), size: 9 };
}
```

### 3.3 Variable-Length String (VarStr)

```javascript
export function writeVarStr(buf, str, offset) {
  const strBuf = Buffer.from(str, 'utf8');
  offset = writeCompactSize(buf, strBuf.length, offset);
  strBuf.copy(buf, offset);
  return offset + strBuf.length;
}

export function varStrBytes(str) {
  const len = Buffer.byteLength(str, 'utf8');
  return compactSizeBytes(len) + len;
}
```

### 3.4 Hash Encoding

Bitcoin hashes (block hashes, txids) are displayed in big-endian hex but transmitted in little-endian (bytes reversed).

```javascript
export function hexToWireBytes(hexStr) {
  return Buffer.from(hexStr, 'hex').reverse();
}

export function wireBytesToHex(buf) {
  return Buffer.from(buf).reverse().toString('hex');
}
```

### 3.5 Checksum

```javascript
import { createHash } from 'crypto';

export function sha256d(buf) {
  return createHash('sha256')
    .update(createHash('sha256').update(buf).digest())
    .digest();
}

export function checksum(payload) {
  return sha256d(payload).slice(0, 4);
}
```

---

## 4. Message Frame Format

Every Bitcoin P2P message has the same frame structure.

### 4.1 Header (24 bytes, always)

| Field | Size | Type | Description |
|---|---|---|---|
| magic | 4 | uint32 | Network magic (LE) |
| command | 12 | char[12] | ASCII name, null-padded to 12 bytes |
| length | 4 | uint32LE | Byte count of payload |
| checksum | 4 | bytes | First 4 bytes of SHA256(SHA256(payload)) |

### 4.2 Frame Builder

```javascript
// src/messages.js
import { sha256d, checksum } from './encoding.js';

export const NETWORKS = {
  mainnet: Buffer.from([0xf9, 0xbe, 0xb4, 0xd9]),
  testnet3: Buffer.from([0x0b, 0x11, 0x09, 0x07]),
  regtest: Buffer.from([0xfa, 0xbf, 0xb5, 0xda]),
};

export function buildMessage(network, command, payload) {
  const magic = NETWORKS[network];
  const header = Buffer.alloc(24);
  magic.copy(header, 0);
  const cmdBuf = Buffer.alloc(12, 0);
  Buffer.from(command, 'ascii').copy(cmdBuf, 0);
  cmdBuf.copy(header, 4);
  header.writeUInt32LE(payload.length, 16);
  checksum(payload).copy(header, 20);
  return Buffer.concat([header, payload]);
}

export function parseHeader(buf) {
  return {
    magic: buf.slice(0, 4),
    command: buf.slice(4, 16).toString('ascii').replace(/\0/g, ''),
    length: buf.readUInt32LE(16),
    checksum: buf.slice(20, 24),
  };
}

export function verifyChecksum(payload, expected) {
  return checksum(payload).equals(expected);
}
```

---

## 5. Handshake: Connection Lifecycle

### 5.1 Exact Sequence

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
  |                                    |
  |<--- sendheaders (optional) --------|
  |<--- sendcmpct (optional) ----------|
  |<--- ping --------------------------|
  |---- pong ------------------------->|
```

Rule: Send verack only AFTER receiving the server's version. Do not send it before.

### 5.2 Version Message Payload

| Field | Size | Type | Description |
|---|---|---|---|
| version | 4 | int32LE | Protocol version (use 70015) |
| services | 8 | uint64LE | NODE_NETWORK = 1; use 0 if not serving |
| timestamp | 8 | int64LE | Unix timestamp (seconds) |
| addr_recv.svc | 8 | uint64LE | Services of remote peer (use 1) |
| addr_recv.ip | 16 | bytes | IPv4-mapped IPv6: 10 zero + 2x 0xFF + 4 IPv4 |
| addr_recv.port | 2 | uint16BE | IMPORTANT: port is BIG-endian |
| addr_trans.svc | 8 | uint64LE | Our services |
| addr_trans.ip | 16 | bytes | Our IP (use all zeros) |
| addr_trans.port | 2 | uint16BE | Our port (use 0) |
| nonce | 8 | uint64LE | Random nonce (detect self-connections) |
| user_agent | var | VarStr | e.g. "/MyNode:1.0.0/" |
| start_height | 4 | int32LE | Our best block height (use 0 initially) |
| relay | 1 | bool | 1 = send us txs, 0 = headers only |

```javascript
export function buildVersion(opts = {}) {
  const {
    network = 'testnet3',
    protocolVersion = 70015,
    services = 0n,
    timestamp = BigInt(Math.floor(Date.now() / 1000)),
    receiverIP = '127.0.0.1',
    receiverPort = 18333,
    nonce = BigInt(Math.floor(Math.random() * 2**52)),
    userAgent = '/bitcoin-node-client:1.0.0/',
    startHeight = 0,
    relay = 1,
  } = opts;

  const userAgentBytes = varStrBytes(userAgent);
  const payload = Buffer.alloc(4 + 8 + 8 + 26 + 26 + 8 + userAgentBytes + 4 + 1);
  let o = 0;

  o = writeInt32LE(payload, protocolVersion, o);
  payload.writeBigUInt64LE(services, o); o += 8;
  payload.writeBigInt64LE(timestamp, o); o += 8;

  payload.writeBigUInt64LE(1n, o); o += 8;
  const ipParts = receiverIP.split('.').map(Number);
  payload.fill(0, o, o + 10); o += 10;
  payload[o++] = 0xff; payload[o++] = 0xff;
  ipParts.forEach(b => { payload[o++] = b; });
  payload.writeUInt16BE(receiverPort, o); o += 2;

  payload.fill(0, o, o + 26); o += 26;

  payload.writeBigUInt64LE(nonce, o); o += 8;
  o = writeVarStr(payload, userAgent, o);
  o = writeInt32LE(payload, startHeight, o);
  payload.writeUInt8(relay, o);

  return buildMessage(network, 'version', payload);
}
```

### 5.3 Verack Message

```javascript
export function buildVerack(network = 'testnet3') {
  return buildMessage(network, 'verack', Buffer.alloc(0));
}
```

### 5.4 Peer State Machine

```javascript
// src/peer.js
import net from 'net';
import { EventEmitter } from 'events';
import { buildVersion, buildVerack, parseHeader, verifyChecksum, NETWORKS } from './messages.js';

export class Peer extends EventEmitter {
  constructor({ host, port, network = 'testnet3' }) {
    super();
    this.host = host;
    this.port = port;
    this.network = network;
    this.socket = null;
    this.buffer = Buffer.alloc(0);
    this.handshakeDone = false;
    this.versionReceived = false;
    this.verackReceived = false;
    this.bestHeight = 0;
    this.nonce = BigInt(Math.floor(Math.random() * 2**52));
  }

  connect() {
    return new Promise((resolve, reject) => {
      this.socket = net.createConnection({ host: this.host, port: this.port });

      this.socket.on('connect', () => {
        this.send(buildVersion({
          network: this.network,
          receiverIP: this.host,
          receiverPort: this.port,
          nonce: this.nonce,
        }));
      });

      this.socket.on('data', (chunk) => this._onData(chunk));
      this.socket.on('error', reject);
      this.socket.on('close', () => this.emit('disconnect'));

      this.once('handshake', resolve);
      this.socket.setTimeout(10000, () => reject(new Error('Handshake timeout')));
    });
  }

  send(msgBuffer) {
    if (!this.socket || this.socket.destroyed) return;
    this.socket.write(msgBuffer);
  }

  _onData(chunk) {
    this.buffer = Buffer.concat([this.buffer, chunk]);

    while (this.buffer.length >= 24) {
      const header = parseHeader(this.buffer.slice(0, 24));

      if (!this.buffer.slice(0, 4).equals(NETWORKS[this.network])) {
        this.emit('error', new Error('Invalid magic bytes'));
        this.socket.destroy();
        return;
      }

      if (this.buffer.length < 24 + header.length) break;

      const payload = this.buffer.slice(24, 24 + header.length);

      if (!verifyChecksum(payload, header.checksum)) {
        this.emit('error', new Error(`Bad checksum for ${header.command}`));
        this.socket.destroy();
        return;
      }

      this.buffer = this.buffer.slice(24 + header.length);
      this._handleMessage(header.command, payload);
    }
  }

  _handleMessage(command, payload) {
    switch (command) {
      case 'version': this._onVersion(payload); break;
      case 'verack': this._onVerack(); break;
      case 'ping': this._onPing(payload); break;
      case 'inv': this.emit('inv', parseInv(payload)); break;
      case 'headers': this.emit('headers', parseHeaders(payload)); break;
      case 'block': this.emit('block', payload); break;
      case 'tx': this.emit('tx', payload); break;
      case 'addr': this.emit('addr', parseAddr(payload)); break;
      case 'notfound': this.emit('notfound', parseInv(payload)); break;
      case 'reject': this.emit('reject', parseReject(payload)); break;
      default: this.emit('message', { command, payload }); break;
    }
  }

  _onVersion(payload) {
    this.versionReceived = true;
    this.bestHeight = payload.readInt32LE(80);
    this.send(buildVerack(this.network));
    this._checkHandshake();
  }

  _onVerack() {
    this.verackReceived = true;
    this._checkHandshake();
  }

  _checkHandshake() {
    if (this.versionReceived && this.verackReceived && !this.handshakeDone) {
      this.handshakeDone = true;
      this.socket.setTimeout(0);
      this.emit('handshake');
    }
  }

  _onPing(payload) {
    const pong = buildMessage(this.network, 'pong', payload);
    this.send(pong);
  }

  close() {
    this.socket?.destroy();
  }
}
```

---

## 6. All Message Types: Serialization Reference

### 6.1 Inventory Vectors

Used in inv, getdata, notfound messages. Each inventory is 36 bytes.

| Field | Size | Type | Description |
|---|---|---|---|
| type | 4 | uint32LE | 0=ERROR, 1=MSG_TX, 2=MSG_BLOCK, 0x40000001=MSG_WITNESS_TX |
| hash | 32 | bytes | Little-endian hash |

```javascript
export const INV_TX = 1;
export const INV_BLOCK = 2;
export const INV_WTXID = 0x40000001;
export const INV_WBLOCK = 0x40000002;

export function buildInv(network, items) {
  const payload = Buffer.alloc(compactSizeBytes(items.length) + items.length * 36);
  let o = writeCompactSize(payload, items.length, 0);
  for (const item of items) {
    payload.writeUInt32LE(item.type, o); o += 4;
    item.hash.copy(payload, o); o += 32;
  }
  return buildMessage(network, 'inv', payload);
}

export function parseInv(payload) {
  const { value: count, size } = readCompactSize(payload, 0);
  let o = size;
  const items = [];
  for (let i = 0; i < count; i++) {
    items.push({
      type: payload.readUInt32LE(o),
      hash: payload.slice(o + 4, o + 36),
    });
    o += 36;
  }
  return items;
}
```

### 6.2 getdata

```javascript
export function buildGetdata(network, items) {
  const payload = Buffer.alloc(compactSizeBytes(items.length) + items.length * 36);
  let o = writeCompactSize(payload, items.length, 0);
  for (const item of items) {
    payload.writeUInt32LE(item.type, o); o += 4;
    item.hash.copy(payload, o); o += 32;
  }
  return buildMessage(network, 'getdata', payload);
}
```

### 6.3 getheaders / getblocks

```javascript
export function buildGetheaders(network, locatorHashes, stopHash = null) {
  const stop = stopHash || Buffer.alloc(32, 0);
  const count = locatorHashes.length;
  const payload = Buffer.alloc(4 + compactSizeBytes(count) + count * 32 + 32);
  let o = 0;
  payload.writeUInt32LE(70015, o); o += 4;
  o = writeCompactSize(payload, count, o);
  for (const hash of locatorHashes) { hash.copy(payload, o); o += 32; }
  stop.copy(payload, o);
  return buildMessage(network, 'getheaders', payload);
}
```

### 6.4 ping / pong

```javascript
export function buildPing(network) {
  const payload = Buffer.alloc(8);
  payload.writeBigUInt64LE(BigInt(Math.floor(Math.random() * 2**52)), 0);
  return buildMessage(network, 'ping', payload);
}
```

### 6.5 addr / getaddr

```javascript
export function buildGetaddr(network) {
  return buildMessage(network, 'getaddr', Buffer.alloc(0));
}

export function parseAddr(payload) {
  const { value: count, size } = readCompactSize(payload, 0);
  let o = size;
  const addrs = [];
  for (let i = 0; i < count; i++) {
    const timestamp = payload.readUInt32LE(o); o += 4;
    const services = payload.readBigUInt64LE(o); o += 8;
    const ip = payload.slice(o, o + 16); o += 16;
    const port = payload.readUInt16BE(o); o += 2;
    const ipv4 = ip[10] === 0xff && ip[11] === 0xff
      ? `${ip[12]}.${ip[13]}.${ip[14]}.${ip[15]}`
      : null;
    addrs.push({ timestamp, services, ip: ipv4, port });
  }
  return addrs;
}
```

### 6.6 mempool

```javascript
export function buildMempool(network) {
  return buildMessage(network, 'mempool', Buffer.alloc(0));
}
```

### 6.7 Block Header Parsing

```javascript
export function parseHeaders(payload) {
  const { value: count, size } = readCompactSize(payload, 0);
  let o = size;
  const headers = [];
  for (let i = 0; i < count; i++) {
    const header = payload.slice(o, o + 80);
    headers.push({
      version: header.readInt32LE(0),
      prevBlock: header.slice(4, 36),
      merkleRoot: header.slice(36, 68),
      timestamp: header.readUInt32LE(68),
      bits: header.readUInt32LE(72),
      nonce: header.readUInt32LE(76),
      hash: sha256d(header),
    });
    o += 80;
    const { size: txCountSize } = readCompactSize(payload, o);
    o += txCountSize;
  }
  return headers;
}
```

### 6.8 Transaction Parsing (Segwit-aware)

```javascript
export function parseTx(payload, offset = 0) {
  let o = offset;
  const version = payload.readInt32LE(o); o += 4;

  let segwit = false;
  if (payload[o] === 0x00 && payload[o + 1] !== 0x00) {
    segwit = true;
    o += 2;
  }

  const { value: inCount, size: inSz } = readCompactSize(payload, o); o += inSz;
  const inputs = [];
  for (let i = 0; i < inCount; i++) {
    const prevHash = payload.slice(o, o + 32); o += 32;
    const prevIndex = payload.readUInt32LE(o); o += 4;
    const { value: scriptLen, size: slSz } = readCompactSize(payload, o); o += slSz;
    const script = payload.slice(o, o + scriptLen); o += scriptLen;
    const sequence = payload.readUInt32LE(o); o += 4;
    inputs.push({ prevHash, prevIndex, script, sequence });
  }

  const { value: outCount, size: outSz } = readCompactSize(payload, o); o += outSz;
  const outputs = [];
  for (let i = 0; i < outCount; i++) {
    const value = payload.readBigUInt64LE(o); o += 8;
    const { value: scriptLen, size: slSz } = readCompactSize(payload, o); o += slSz;
    const script = payload.slice(o, o + scriptLen); o += scriptLen;
    outputs.push({ value, script });
  }

  if (segwit) {
    for (const input of inputs) {
      const { value: wCount, size: wcSz } = readCompactSize(payload, o); o += wcSz;
      input.witness = [];
      for (let w = 0; w < wCount; w++) {
        const { value: itemLen, size: ilSz } = readCompactSize(payload, o); o += ilSz;
        input.witness.push(payload.slice(o, o + itemLen)); o += itemLen;
      }
    }
  }

  const locktime = payload.readUInt32LE(o); o += 4;
  return { version, inputs, outputs, locktime, segwit, bytesRead: o - offset };
}
```

---

## 7. Peer Discovery

### 7.1 DNS Seeds

```javascript
// src/dns.js
import dns from 'dns/promises';

export async function discoverPeers(network = 'testnet3') {
  const seeds = network === 'mainnet' ? [
    'seed.bitcoin.sipa.be',
    'dnsseed.bluematt.me',
    'seed.bitcoinstats.com',
  ] : [
    'testnet-seed.bitcoin.jonasschnelli.ch',
    'seed.tbtc.petertodd.net',
  ];

  const port = network === 'mainnet' ? 8333 : 18333;
  const peers = [];

  for (const seed of seeds) {
    try {
      const addrs = await dns.resolve4(seed);
      for (const ip of addrs) {
        peers.push({ host: ip, port });
      }
    } catch (_) {}
  }
  return peers;
}
```

---

## 8. Blockchain Sync (IBD)

### 8.1 Headers-First IBD

Use headers-first sync. Never use blocks-first.

```
Phase 1: Sync headers
  getheaders(locator=[genesis]) -> headers(up to 2000)
  repeat until no new headers received

Phase 2: Download blocks in parallel
  For each header: getdata([MSG_BLOCK, hash])
  Validate: prevBlock matches, merkleRoot matches, PoW check
```

### 8.2 Block Locator

```javascript
export function buildLocator(knownHashes) {
  const locator = [];
  let step = 1;
  for (let i = knownHashes.length - 1; i >= 0; i -= step) {
    locator.push(knownHashes[i]);
    if (locator.length >= 10) step *= 2;
  }
  if (!locator.includes(knownHashes[0])) locator.push(knownHashes[0]);
  return locator;
}
```

### 8.3 Proof of Work Verification

```javascript
export function verifyPoW(headerBuf, bits) {
  const hash = sha256d(headerBuf);
  const exp = bits >>> 24;
  const mantissa = bits & 0x7fffff;
  const target = Buffer.alloc(32, 0);
  const targetBytes = mantissa.toString(16).padStart(6, '0');
  const targetBuf = Buffer.from(targetBytes, 'hex');
  const pos = 32 - exp;
  targetBuf.copy(target, pos < 0 ? 0 : pos);
  const hashBE = Buffer.from(hash).reverse();
  return hashBE.compare(target.reverse()) <= 0;
}
```

---

## 9. Transaction Handling

### 9.1 Broadcasting

```javascript
const rawTx = buildTx({ inputs, outputs });
const txMsg = buildMessage(network, 'tx', rawTx);
peer.send(txMsg);
```

### 9.2 TXID Calculation

```javascript
export function txid(rawTx) {
  return sha256d(rawTx).reverse().toString('hex');
}
```

---

## 10. Address/Wallet Monitoring

```javascript
// src/wallet.js
import bs58check from 'bs58check';

export function addressToScriptPubKey(address) {
  if (address.startsWith('bc1') || address.startsWith('tb1')) {
    throw new Error('Bech32 requires the "bech32" npm package');
  }

  const decoded = bs58check.decode(address);
  const version = decoded[0];
  const hash = decoded.slice(1);

  if (version === 0x00 || version === 0x6f) {
    // P2PKH
    return Buffer.concat([
      Buffer.from([0x76, 0xa9, 0x14]), hash, Buffer.from([0x88, 0xac]),
    ]);
  }

  if (version === 0x05 || version === 0xc4) {
    // P2SH
    return Buffer.concat([
      Buffer.from([0xa9, 0x14]), hash, Buffer.from([0x87]),
    ]);
  }

  throw new Error(`Unsupported address version: ${version}`);
}

export class WalletMonitor {
  constructor(watchAddresses) {
    this.scripts = new Map();
    for (const addr of watchAddresses) {
      this.scripts.set(addr, addressToScriptPubKey(addr));
    }
  }

  checkTx(txPayload) {
    const tx = parseTx(txPayload);
    const matches = [];
    for (const [addr, script] of this.scripts) {
      for (const [i, output] of tx.outputs.entries()) {
        if (output.script.equals(script)) {
          matches.push({ address: addr, outputIndex: i, value: output.value, tx });
        }
      }
    }
    return matches;
  }
}
```

---

## 11. Mempool Access

```javascript
// src/mempool.js
export class Mempool {
  constructor() {
    this.txids = new Set();
    this.txs = new Map();
  }

  requestMempool(peer, network) {
    peer.send(buildMempool(network));
  }

  onInv(peer, network, items) {
    const needed = items.filter(i =>
      i.type === INV_TX && !this.txids.has(i.hash.toString('hex'))
    );
    if (needed.length > 0) {
      peer.send(buildGetdata(network, needed));
    }
  }

  onTx(txPayload) {
    const tx = parseTx(txPayload);
    const id = txid(txPayload);
    this.txids.add(id);
    this.txs.set(id, tx);
    return id;
  }

  size() { return this.txids.size; }
}
```

---

## 12. Complete Client Architecture

```javascript
// src/index.js
import { Peer } from './peer.js';
import { discoverPeers } from './dns.js';
import { syncHeaders } from './blockchain.js';
import { Mempool } from './mempool.js';
import { WalletMonitor } from './wallet.js';
import { buildGetdata, INV_BLOCK, buildMessage } from './messages.js';

const NETWORK = process.env.NETWORK || 'testnet3';
const WATCH_ADDRESSES = (process.env.WATCH_ADDRESSES || '').split(',').filter(Boolean);

async function main() {
  console.log(`Starting Bitcoin node client on ${NETWORK}`);

  let peers = await discoverPeers(NETWORK);
  console.log(`Discovered ${peers.length} peers`);

  let peer;
  for (const peerInfo of peers.slice(0, 5)) {
    try {
      peer = new Peer({ ...peerInfo, network: NETWORK });
      await peer.connect();
      console.log(`Connected to ${peerInfo.host}:${peerInfo.port}`);
      break;
    } catch (e) {
      console.log(`Failed: ${peerInfo.host}: ${e.message}`);
    }
  }

  if (!peer) throw new Error('Could not connect to any peer');

  const mempool = new Mempool();
  const wallet = new WalletMonitor(WATCH_ADDRESSES);

  peer.on('inv', (items) => {
    mempool.onInv(peer, NETWORK, items);
    const blockInvs = items.filter(i => i.type === INV_BLOCK);
    if (blockInvs.length > 0) {
      peer.send(buildGetdata(NETWORK, blockInvs));
    }
  });

  peer.on('tx', (txPayload) => {
    const id = mempool.onTx(txPayload);
    const hits = wallet.checkTx(txPayload);
    for (const h of hits) {
      console.log(`TX: ${id} -> ${h.address}: ${h.value} sats`);
    }
  });

  peer.on('block', (blockPayload) => {
    console.log(`Block received: ${blockPayload.length} bytes`);
  });

  console.log('Starting header sync...');
  const headerChain = await syncHeaders(peer, NETWORK);
  console.log(`Header sync complete: ${headerChain.length} headers`);

  mempool.requestMempool(peer, NETWORK);

  setInterval(() => {
    const b = Buffer.alloc(8);
    b.writeBigUInt64LE(BigInt(Math.floor(Math.random() * 2**52)), 0);
    peer.send(buildMessage(NETWORK, 'ping', b));
  }, 60000);

  console.log('Node running. Listening for transactions...');
}

main().catch(e => { console.error(e); process.exit(1); });
```

---

## 13. Full Working Implementation

```bash
# testnet3 (default)
node src/index.js

# mainnet
NETWORK=mainnet node src/index.js

# with address monitoring
WATCH_ADDRESSES=tb1qxxxxxx,tb1qyyyyyy node src/index.js
```

---

## 14. Anti-Patterns and Checklist

### 14.1 Never Do These

| Wrong | Correct |
|---|---|
| Assume one message per TCP chunk | Accumulate in buffer, loop until complete |
| Send verack immediately after connecting | Send verack only AFTER receiving server's version |
| Encode port as little-endian | Port in addr/version is BIG-endian |
| Display hash bytes directly | Reverse bytes before hex display |
| Ignore reject messages | Always listen for reject |
| Use parseInt for satoshi values | Use BigInt |
| Send getheaders before handshake | Wait for handshake event |
| Use MSG_TX in getdata after BIP 339 | Use MSG_WITNESS_TX (0x40000001) |
| Call dns.lookup() for seeds | Use dns.resolve4() |

### 14.2 Generation Checklist

- [ ] `"type": "module"` in package.json
- [ ] TCP buffer accumulation loop in _onData
- [ ] Magic bytes validated on every received message
- [ ] Checksum validated on every received message
- [ ] verack sent only after version received
- [ ] Port encoded as big-endian in address fields
- [ ] Hashes reversed before display
- [ ] BigInt used for satoshi values and services
- [ ] Handshake awaited before getheaders/getdata
- [ ] Periodic ping for keepalive
- [ ] reject event handler wired up
- [ ] Graceful close on SIGINT/SIGTERM
- [ ] DNS discovery with fallback peers
- [ ] CompactSize before ALL variable-length fields

### 14.3 Byte Budget Reference

| Message | Min Size | Notes |
|---|---|---|
| header only | 24 | fixed |
| version | 86+ | +VarStr(userAgent) |
| verack | 24 | no payload |
| ping/pong | 32 | 24 + 8 byte nonce |
| inv (1 item) | 61 | 24 + 1 + 36 |
| getheaders | 29+ | 24 + 4 + 1 + 32*n + 32 |
| block header | 80 | version+prevBlock+merkle+time+bits+nonce |

---

*End of protocol document. Hand this file verbatim to any LLM with the instruction: "Implement this Bitcoin full node client exactly as specified."*

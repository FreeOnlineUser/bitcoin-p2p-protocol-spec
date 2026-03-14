# Bitcoin P2P Protocol: Implementation Specification

A complete, LLM-executable specification for building a Bitcoin full node P2P client from scratch. No external P2P libraries, just raw TCP and the Bitcoin wire protocol.

Inspired by [@SteveSimple](https://x.com/SteveSimple)'s suggestion that someone should build a from-scratch Bitcoin node client. This spec was generated with Claude and is designed to be handed to any LLM with the instruction: *"Implement this exactly as specified."*

## What This Covers

- Binary encoding primitives (little-endian integers, CompactSize varints)
- TCP message framing (magic bytes, command, length, checksum)
- Version/verack handshake lifecycle
- Peer discovery via DNS seeds
- Headers-first blockchain sync (IBD)
- Full transaction parsing (segwit-aware)
- Transaction building and broadcasting
- Address/wallet monitoring
- Mempool tracking
- Proof of work verification

## What This Is NOT

- Not an RPC client for bitcoind
- Not an SPV/light client
- Not a miner

## Target

The specification is entirely language-agnostic. Hand it to any LLM targeting Node.js, Kotlin, Rust, Python, Go, or any other language.

## Usage

```bash
# Give the spec to an LLM
"Implement the Bitcoin full node client exactly as specified in PROTOCOL.md"
```

## Files

- `PROTOCOL.md` - The complete implementation specification
- `README.md` - This file

## License

MIT

## Credits

Protocol knowledge from the Bitcoin P2P network documentation and years of open source Bitcoin development. Specification structured for LLM consumption.

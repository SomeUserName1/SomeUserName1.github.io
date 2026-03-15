---
title: "Building a Generic Blockchain in Rust: Beyond Cryptocurrency"
date: 2026-03-15T17:07:00+01:00
summary: "Using Rust's type system to build a blockchain where the transaction payload is a generic parameter — supporting cryptocurrency, voting, and more."
---

Most blockchain tutorials start and end with cryptocurrency. Send coins, receive coins, mine blocks, repeat. But blockchains are a general-purpose data structure — a tamper-evident, distributed ledger that can track *anything*. What if the framework itself reflected that generality?

That's the premise behind **generic-blockchain-rs**, a Rust project by Fabian Klopfer, Felix Mayer, and Stephan Perren that uses Rust's type system to build a blockchain where the transaction payload is a generic parameter. Want a cryptocurrency? Plug in `CryptoPayload`. A voting system? Use `VotePayload`. Distributed version control? There's `CodePayload` for that. The consensus layer, networking, and storage don't care — they work with anything that implements the `Transactional` trait.

## The Core Idea: Generics All the Way Down

At the heart of the project is a single design decision: every major data structure — `Transaction<T>`, `Block<T>`, `Chain<T>`, `Node<T>` — is parameterized over `T: Transactional`. This isn't just an academic exercise. It means the compiler enforces type safety across the entire stack at zero runtime cost. You can't accidentally mix voting transactions into a cryptocurrency chain. The type system won't let you.

The `Transactional` trait requires just a few methods, most notably `genesis()`, which defines how miner reward transactions are created for each payload type. This keeps the block mining logic completely decoupled from any specific application.

```rust
// Three built-in payload types demonstrate the flexibility:
// CryptoPayload — receiver address + amount
// VotePayload   — a vote choice
// CodePayload   — file name, contents, and commit message
```

## Proof-of-Work with Dynamic Difficulty

The mining implementation follows the classic proof-of-work pattern: increment a nonce until the block header's SHA3-512 hash starts with a required number of leading zeros. But it adds a twist — difficulty increases every 100 blocks, and miner rewards scale with difficulty. Blocks are automatically mined when pending transactions exceed a threshold of 20, and each block includes a Merkle root computed from its transactions for efficient verification.

The Merkle tree implementation handles odd-length transaction lists by duplicating the final hash before pairing — the same pragmatic approach used in Bitcoin.

## Async P2P Networking

The networking layer is where Rust's async ecosystem gets a real workout. Built on Tokio, each node operates as a fully asynchronous peer, handling incoming connections, broadcasting transactions, and mining blocks without any operation blocking another.

Nodes communicate over TCP using a JSON-lines protocol (newline-delimited JSON encoded via Serde). The message types are straightforward:

- **Ping/Pong** — Peer discovery and handshake, with the responding node sharing its current chain
- **PeerList** — Gossip protocol that broadcasts known peers every 3 seconds
- **Transaction** — Broadcast a new transaction for inclusion in the next block

Each node gets a UUID, maintains a peer table, and continuously discovers new peers through gossip. When nodes disagree about the state of the chain, a majority consensus mechanism kicks in: nodes track vote counts for competing chain versions, and the chain with the most votes becomes canonical. The alternative chain cache resets every 30 minutes to prevent unbounded memory growth.

## Cryptographic Foundation

The crypto module goes beyond basic hashing. While SHA3-512 handles block and transaction hashing, the project also integrates OpenPGP via the Sequoia library for key generation (RSA 3072-bit), signing, verification, and encryption. The PGP infrastructure is fully implemented and tested but not yet wired into the networking layer — a clear extension point for anyone wanting to add authenticated transactions or encrypted peer communication.

## Pluggable Storage

Storage follows a trait-based design with two backends: an in-memory `HashMap` for testing and prototyping, and a `RocksDB` backend for persistent storage. The RocksDB schema is designed with column families for key pairs, public keys, peer tables, and the blockchain itself — separating concerns cleanly at the storage level.

## What Makes This Worth Studying

This project isn't trying to compete with Ethereum or Solana. Its value is pedagogical and architectural:

1. **Rust generics in practice.** It's one thing to understand `where T: SomeTrait` in isolation. It's another to see it threaded through an entire distributed system — from serialization to networking to consensus — without a single `dyn` or `Any` in sight.

2. **Async networking done right.** The Tokio-based networking layer demonstrates how to structure a real peer-to-peer system with concurrent connection handling, gossip protocols, and non-blocking I/O.

3. **Separation of concerns.** The blockchain core knows nothing about networking. The networking layer knows nothing about storage. The storage layer knows nothing about what's being stored. Each module can evolve independently.

4. **A template for domain-specific blockchains.** Need an auditable voting system? A supply chain tracker? A distributed configuration store? Implement `Transactional` for your payload type and you inherit the entire infrastructure — mining, consensus, networking, persistence — for free.

The codebase is roughly 1,500 lines of Rust across 17 source files. Small enough to read in an afternoon, rich enough to learn from for much longer.

---

*generic-blockchain-rs is an open-source project. If you're interested in Rust, distributed systems, or blockchain architecture beyond the cryptocurrency hype, it's worth a read.*

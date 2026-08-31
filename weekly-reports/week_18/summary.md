# Weekly Progress Report
## Week 18: Reading on Fiber Network and BOLT Specs

After the TypeScript implementation milestone in Week 17, this week was a deliberate step back from coding. With the two Rust crates and the TypeScript derive package in a stable state, and the next implementation work (TypeScript client, nested struct support) not yet started, it made sense to spend a week reading on the broader payment channel space — specifically Fiber Network, which is the CKB-native payment network that LS-IDL could eventually matter to.

---

### Fiber Network

Fiber is a peer-to-peer payment and swap network built on Nervos CKB, conceptually analogous to the Bitcoin Lightning Network. It uses off-chain payment channels backed by CKB cells: two parties lock funds into a channel, exchange signed commitment transactions off-chain, and settle on-chain only when closing. Because CKB is more programmable than Bitcoin, Fiber extends the channel model in directions Lightning was never designed for — multi-asset channels (CKB native tokens and RGB++ assets in the same channel), cross-chain atomic swaps, and eventual interoperability with the Lightning Network itself.

The reference implementation is [`nervosnetwork/fiber`](https://github.com/nervosnetwork/fiber). The protocol is specified in a set of documents under `docs/specs/` in that repository, and those specs are explicitly modelled on the Bitcoin Lightning BOLTs.

---

### The BOLT Specs

BOLT stands for Basis of Lightning Technology. They are the specification documents that define how the Lightning Network works — the protocol between nodes, not any single implementation. There are 11 of them (BOLT 6 was merged away). The ones most relevant to understanding Fiber:

**BOLT 2 — Peer Protocol for Channel Management**
Defines the message exchange between two channel peers: how a channel is opened (`open_channel` / `accept_channel`), how commitment transactions are constructed and exchanged, how HTLCs are added and resolved, and how a channel is cooperatively or unilaterally closed. Fiber's `p2p-message.md` spec describes itself explicitly as an adaptation and simplification of BOLT 2 for CKB's transaction structure.

**BOLT 3 — Bitcoin Transaction and Script Formats**
Defines the exact transaction shapes for funding, commitment, and HTLC transactions. On Lightning, this is Bitcoin script; on Fiber, the equivalent is CKB lock scripts and cell structures. Reading BOLT 3 makes it clear why Fiber needed to redesign this layer — CKB's cell model and script system are different enough that a direct port is not possible, but the logical structure (funding output, revocation keys, HTLC outputs) maps across.

**BOLT 4 — Onion Routing Protocol**
Defines how payments are routed across the network without intermediate nodes learning the full path. Fiber inherits this design. Reading it is useful context for understanding why the witness format of a channel-related transaction may include routing metadata that the lock script itself does not interpret — it is consumed by a higher layer.

**BOLT 7 — P2P Node and Channel Discovery**
Defines the gossip protocol: how nodes announce themselves, how channels are announced after funding, and how the network graph is built. Relevant because any registry for lock script IDLs would operate in a similar space — nodes advertising capabilities that other nodes can discover.

**BOLT 11 — Invoice Protocol**
The payment request format — what a payee sends to a payer to request a specific amount along a specific route. This is the human-facing layer of Lightning/Fiber. Less directly relevant to LS-IDL, but useful context for how structured data (amount, hash, expiry, routing hints) is encoded and transmitted out-of-band.

---

### Relevance to LS-IDL

Fiber is not a direct user of LS-IDL today, but the connection is real. Every Fiber channel involves lock scripts — funding locks, HTLC locks, revocation locks. All of those have witness formats. Currently those formats are known only to the Fiber node implementation. If Fiber scripts were IDL-annotated, external tooling (wallets, watchtowers, channel explorers) could structurally validate witnesses before submitting channel transactions without having hard-coded knowledge of the Fiber script internals.

This is speculative — Fiber has its own serialisation layer and the HTLC witness formats are more complex than what the current type registry supports. But the reading week clarified that the long-term scope of LS-IDL is not just simple lock scripts; it is any CKB script that has a describable witness interface.

---

### What Comes Next

Reading week over. The next two active work items are:

1. **TypeScript `ckb-idl-client`** — the tooling-side library in TypeScript: IDL commitment verification and structural witness validation, tested against the same 16 canonical vectors.
2. **Nested struct support** — extending the type registry to accept `CkbInnerWitness`-annotated structs as field types, using a trait-based design.

The CKBuilder issue is open and has received its first community feedback, which will be reviewed next week.

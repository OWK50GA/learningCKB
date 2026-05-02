# RFC: CKB Lock Script Interface Description Language (LS-IDL)

- **RFC Number:** TBD
- **Layer:** Application / Tooling Convention
- **Status:** Draft
- **Author:** TBD
- **Created:** 2026-05-01
- **License:** MIT / Apache 2.0

---

## Table of Contents

1. [Abstract](#abstract)
2. [Motivation](#motivation)
3. [Specification](#specification)
   - [IDL Format](#idl-format)
   - [Blessed Type Registry](#blessed-type-registry)
   - [On-Chain Commitment](#on-chain-commitment)
   - [Off-Chain Storage and Indexing](#off-chain-storage-and-indexing)
   - [Verification Protocol](#verification-protocol)
   - [Compiler Integration](#compiler-integration)
4. [Extensibility](#extensibility)
   - [Versioning](#versioning)
   - [Custom Types](#custom-types)
   - [Upgradeable Lock Scripts](#upgradeable-lock-scripts)
5. [Security Considerations](#security-considerations)
6. [Use Cases](#use-cases)
7. [Backwards Compatibility](#backwards-compatibility)
8. [Open Questions](#open-questions)
9. [Acknowledgements](#acknowledgements)

---

## Abstract

This RFC proposes a standard **Interface Description Language (LS-IDL)** for CKB lock scripts. An LS-IDL is a structured JSON document that describes the witness requirements of a lock script — what fields are required, their types, their order, and the signing algorithm expected. The IDL is authored or generated at script development time, stored off-chain, and anchored on-chain by committing its Blake2b hash into the script's code cell data. This enables wallets, hardware signers, dApp frontends, block explorers, and any other tooling to interact with any IDL-compliant lock script without requiring custom, script-specific integrations.

---

## Motivation

CKB's programmable lock scripts are one of its most powerful features. Unlike Bitcoin, where spending conditions are constrained to a finite set of known script templates, CKB allows arbitrary logic to govern how a cell is unlocked. This power creates a critical tooling gap:

**There is currently no protocol-level standard for expressing what witness data a lock script requires.**

In practice this means:

- Hardware wallets only support a small set of hardcoded, well-known lock scripts.
- Wallets must maintain custom integrations per lock script — a maintenance burden that scales poorly.
- Cells locked by obscure or deprecated custom lock scripts can become permanently inaccessible if witness requirements are not documented externally.
- Block explorers display witness fields as raw hex blobs for any non-standard lock script.
- dApp frontends hardcode witness construction logic per script type.

This is analogous to the pre-ABI era on EVM chains, where calling a smart contract required knowing its bytecode interface through out-of-band documentation. The introduction of the Ethereum ABI standard enabled universal tooling across the entire smart contract ecosystem. This RFC proposes the equivalent primitive for CKB lock scripts.

The LS-IDL is not a constraint on lock script authors — it is a **communication mechanism** between script authors and all downstream consumers of those scripts.

---

## Specification

### IDL Format

An LS-IDL is a UTF-8 encoded JSON document. The top-level object has the following structure:

```json
{
  "idl_version": "1",
  "name": "my_lock_script",
  "description": "A brief human-readable description of the lock script",
  "script_version": "1",
  "signing": {
    "algorithm": "secp256k1_ecdsa",
    "message": "ckb_transaction_hash",
    "hasher": "blake2b_256"
  },
  "key_derivation": {
    "standard": "bip44",
    "coin_type": 309
  },
  "witness": [
    {
      "name": "signature",
      "type": "secp256k1_sig",
      "required": true,
      "description": "65-byte ECDSA signature (r, s, v)"
    },
    {
      "name": "pubkey",
      "type": "bytes33",
      "required": false,
      "description": "Optional 33-byte compressed public key"
    }
  ],
  "partial_witness": {
    "accumulating_fields": ["signature"],
    "threshold": 2,
    "total": 3
  }
}
```

#### Top-Level Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `idl_version` | string | yes | Version of the LS-IDL specification itself |
| `name` | string | yes | Human-readable name for the lock script |
| `description` | string | no | Human-readable description |
| `script_version` | string | yes | Version of this specific lock script's interface |
| `signing` | object | yes | Describes the signing algorithm and message construction |
| `key_derivation` | object | no | Key derivation hints for HD wallet signers |
| `witness` | array | yes | Ordered list of witness fields |
| `partial_witness` | object | no | Describes how partial witnesses accumulate (multisig, threshold) |

#### The `signing` Object

| Field | Type | Required | Description |
|---|---|---|---|
| `algorithm` | string | yes | Signing algorithm identifier from the blessed registry |
| `message` | string | yes | What is signed — `"ckb_transaction_hash"` or `"custom"` |
| `hasher` | string | no | Hash function applied before signing, if any |
| `custom_message` | object | no | If `message` is `"custom"`, describes what fields are hashed |

#### The `witness` Array

Each element describes one field of the witness:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Field name |
| `type` | string | yes | Type identifier from the blessed registry or a custom type |
| `required` | boolean | yes | Whether this field must be present for the witness to be valid |
| `description` | string | no | Human-readable field description |

The order of elements in `witness` matches the byte layout order of the actual witness structure.

#### The `partial_witness` Object

This object is present only for scripts that support accumulating partial witnesses — such as multisig locks. It describes how a witness is constructed incrementally as signatures are collected.

| Field | Type | Required | Description |
|---|---|---|---|
| `accumulating_fields` | array | yes | Which `witness` field names accumulate across multiple signers |
| `threshold` | integer | no | Minimum number of entries needed in accumulating fields |
| `total` | integer | no | Total possible number of entries |

---

### Blessed Type Registry

The following types are defined by this RFC and must be understood by all LS-IDL-compliant implementations:

#### Primitive Types

| Type Identifier | Description | Size |
|---|---|---|
| `uint8` | Unsigned 8-bit integer, little-endian | 1 byte |
| `uint16` | Unsigned 16-bit integer, little-endian | 2 bytes |
| `uint32` | Unsigned 32-bit integer, little-endian | 4 bytes |
| `uint64` | Unsigned 64-bit integer, little-endian | 8 bytes |
| `bytes` | Variable-length byte array | variable |
| `bytes20` | Fixed 20-byte array | 20 bytes |
| `bytes32` | Fixed 32-byte array | 32 bytes |
| `bytes33` | Fixed 33-byte array (compressed pubkey) | 33 bytes |
| `bytes64` | Fixed 64-byte array | 64 bytes |
| `bytes65` | Fixed 65-byte array | 65 bytes |

#### Cryptographic Types

| Type Identifier | Description | Size |
|---|---|---|
| `secp256k1_sig` | secp256k1 ECDSA signature (r, s, v) | 65 bytes |
| `secp256k1_pubkey` | Compressed secp256k1 public key | 33 bytes |
| `schnorr_sig` | Schnorr signature (R, s) | 64 bytes |
| `bls12_381_sig` | BLS12-381 signature | 96 bytes |
| `bls12_381_pubkey` | BLS12-381 public key | 48 bytes |
| `blake2b_hash` | Blake2b-256 hash | 32 bytes |

#### Signing Algorithm Identifiers

| Algorithm Identifier | Description |
|---|---|
| `secp256k1_ecdsa` | secp256k1 ECDSA signing |
| `secp256k1_schnorr` | secp256k1 Schnorr signing |
| `bls12_381` | BLS12-381 aggregate signing |
| `custom` | Custom signing algorithm — full description must be provided |

Types not listed here are `custom` and must include a full byte-layout description in the IDL.

---

### On-Chain Commitment

A lock script's code cell data field carries the Blake2b-256 hash of the LS-IDL:

```
code_cell.data = [script_bytecode] ++ [blake2b_256(idl_json_utf8)]
```

The IDL hash occupies the **last 32 bytes** of the code cell's data field. The preceding bytes are the script bytecode itself.

**A code cell that does not carry an IDL hash is simply non-IDL-compliant.** This RFC imposes no requirement on existing or future scripts to adopt LS-IDL. It is a voluntary convention.

#### Governing Type Script (Optional but Recommended)

Script authors who wish to make IDL compliance enforceable at the protocol level may deploy their code cell under a governing type script that enforces:

1. The last 32 bytes of cell data must be a valid 32-byte IDL hash.
2. Any update to the cell data must preserve a valid IDL hash.
3. The IDL hash must not be the zero hash.

This is not required by this RFC, but is strongly recommended for scripts intended for broad ecosystem use.

---

### Off-Chain Storage and Indexing

The full LS-IDL JSON document is stored off-chain. Any compliant indexer must:

1. Monitor code cells whose data fields end in a 32-byte value.
2. Accept IDL submissions keyed by `code_hash` (the Blake2b hash of the script bytecode).
3. Verify that `blake2b_256(submitted_idl) == idl_hash` stored in the code cell.
4. Serve IDL documents by `code_hash` lookup.

The indexer's role is **storage and retrieval only** — it does not validate the semantic correctness of the IDL. Trust is established by the on-chain hash, not by the indexer.

Multiple independent indexers may serve the same IDL. Clients should not rely on a single indexer.

---

### Verification Protocol

Any client consuming an LS-IDL — whether a wallet, hardware signer, or dApp frontend — must follow this verification protocol:

```
1. Identify the lock script of the input cell being spent
2. Extract lock.code_hash — this is the unique identifier for the script
3. Fetch the IDL from an indexer: GET /idl/{code_hash}
   OR read the IDL bundled in the PSCT (see PSCT RFC)
4. Compute: blake2b_256(received_idl)
5. Fetch the code cell referenced by code_hash
6. Extract the last 32 bytes of code_cell.data
7. Assert: blake2b_256(received_idl) == code_cell.data[-32:]
8. If assertion passes: IDL is trusted. Proceed to construct witness.
9. If assertion fails: REJECT. Do not sign.
```

Clients must cache verified IDLs locally, keyed by `code_hash`. Subsequent interactions with the same lock script type do not require a repeat indexer fetch.

---

### Compiler Integration

Manually authoring LS-IDL JSON is error-prone and creates a maintenance burden. The recommended approach is **compiler-generated IDLs**.

For Rust-based CKB scripts, the reference implementation uses a procedural macro:

```rust
use ckb_idl::lock_interface;

#[lock_interface]
pub fn verify_witness(witness: MyWitness, tx: &Transaction) -> bool {
    // verification logic
}

#[derive(CkbWitness)]
pub struct MyWitness {
    pub signature: Secp256k1Sig,
    pub pubkey_index: u8,
}
```

The `#[lock_interface]` and `#[derive(CkbWitness)]` macros inspect the `MyWitness` type at compile time and emit a corresponding LS-IDL JSON file as a build artifact. The build toolchain then:

1. Compiles the script to RISC-V bytecode.
2. Generates the IDL JSON.
3. Computes `blake2b_256(idl_json)`.
4. Appends the hash to the bytecode before deploying the code cell.

The IDL JSON file is published alongside the script release and submitted to indexers.

---

## Extensibility

### Versioning

The `idl_version` field in the IDL document allows the specification to evolve. A client encountering an `idl_version` it does not recognize should treat the IDL as uninterpretable and must not proceed with signing.

The `script_version` field allows a lock script's *interface* to version independently of the specification version. When a script's witness structure changes, `script_version` increments. Old and new IDL versions may coexist in indexers.

### Custom Types

Types not in the blessed registry may be used by setting `"type": "custom"` and providing a `"layout"` field describing the byte structure:

```json
{
  "name": "my_proof",
  "type": "custom",
  "layout": {
    "encoding": "little_endian",
    "fields": [
      { "name": "length", "type": "uint32" },
      { "name": "data", "type": "bytes", "length_from": "length" }
    ]
  }
}
```

Frequently used custom types may be proposed for inclusion in the blessed registry through the RFC process.

### Upgradeable Lock Scripts

This RFC intentionally defers a full treatment of upgradeable lock scripts. Scripts deployed under a type script that permits bytecode upgrades present a challenge: the `code_hash` changes when the bytecode changes, making the IDL lookup path straightforward — the new bytecode has a new `code_hash`, and its IDL is looked up independently.

The harder case — scripts whose `lock.code_hash` references a type hash (`hash_type: "type"`) rather than a data hash — is left for a follow-up RFC that addresses upgradeable script patterns more broadly.

---

## Security Considerations

### IDL is Descriptive, Not Enforceable

The LS-IDL describes what a lock script expects. It does not constrain what the lock script accepts at the protocol level. A malicious or buggy lock script may accept witness data that does not match its IDL. Clients must treat the IDL as an authoritative description only when the on-chain hash commitment is verified.

### Hash Commitment is the Trust Root

The `blake2b_256(idl)` stored in the code cell is the only source of trust. Indexers, frontends, and bundled IDLs are all untrusted until verified against this hash. Implementations must not skip the verification step.

### IDL Staleness for Upgradeable Scripts

If a script's bytecode changes while cells locked by the old bytecode still exist, the old `code_hash` remains valid for spending those cells. Indexers should retain IDLs for all historical `code_hash` values, not just the latest version.

### Indexer Availability

Clients must cache verified IDLs locally. Over-reliance on indexer availability creates a liveness dependency for signing. Clients should prefer bundled IDLs (as in the PSCT format) for offline or air-gapped signing scenarios.

---

## Use Cases

1. **Hardware wallet universal support** — hardware wallets can support any IDL-compliant lock script without firmware updates by fetching and verifying the IDL at signing time.
2. **Universal wallet integration** — wallets support new lock scripts on day of deployment with zero wallet-side code changes.
3. **Block explorer witness decoding** — explorers decode and display witness fields in human-readable form for any IDL-compliant lock.
4. **Script archaeology** — cells locked by deprecated or obscure scripts remain spendable as long as their IDL is indexed, even if the original author's documentation is lost.
5. **dApp frontend universality** — frontends use a single IDL-driven witness construction module, eliminating script-specific frontend integrations.
6. **Automated test generation** — testing frameworks generate valid and invalid witness structures for any lock script from its IDL.
7. **Cross-script composition** — meta-lock scripts can understand and validate sub-lock witness requirements through IDL introspection.
8. **Security auditing** — automated tools compare declared IDL witness structure against observed script behaviour to flag mismatches.
9. **Script upgrade communication** — IDL version diffs communicate interface changes clearly to all downstream tooling.
10. **Ecosystem-wide script registry** — a decentralised, verifiable registry of all deployed CKB lock scripts emerges naturally from the indexer network.

---

## Backwards Compatibility

This RFC introduces no breaking changes to the CKB protocol. Existing lock scripts are unaffected. The LS-IDL is a voluntary convention for new scripts. Tooling that does not understand LS-IDL continues to function for scripts it already knows.

Clients encountering a lock script with no IDL hash in its code cell data should fall back to existing behaviour — treating the script as unknown and requiring manual or custom integration.

---

## Open Questions

1. **IDL format for Molecule-based witnesses** — should the IDL support referencing an existing Molecule schema as the witness structure definition, for scripts that already use Molecule encoding?

2. **Type registry governance** — what process governs the addition of new types to the blessed registry? A separate RFC process mirroring CKB's existing RFC conventions is the current assumption.

3. **Upgradeable script IDL binding** — how should IDLs be associated with scripts deployed under `hash_type: "type"`, where the referenced bytecode can change? Full treatment deferred to a follow-up RFC.

4. **IDL composability** — should IDLs support referencing or extending other IDLs, to avoid redundancy in scripts that wrap or extend known lock script patterns?

5. **Minimum indexer API specification** — should this RFC define a minimal HTTP API standard for compliant indexers, or leave that to implementers?

---

## Acknowledgements

This RFC emerged from a broader discussion on adapting Bitcoin's BIP-174 (Partially Signed Bitcoin Transactions) for the CKB cell model, which highlighted the absence of a standard mechanism for describing lock script witness requirements. The EVM ABI and Starknet's `#[abi(embed_v0)]` pattern informed the compiler integration approach.

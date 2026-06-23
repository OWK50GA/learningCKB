# Weekly Progress Report
## Week 9: The Registry Problem and PSCT Foundations

Week 8 ended with a clear plan: implement the `verify` method in `ckb-idl-client`, then test it against the `simple-lock` IDL. That plan was executed, but the implementation surfaced a more fundamental question that consumed most of the week: where does the IDL actually live in production, and what does "fetching" it really mean when you have no off-chain infrastructure?

---

### Vocabulary: Two Hashes, Two Jobs

Before going further it is worth pinning terminology precisely, because this is where the design is easiest to get confused.

There are two distinct hashes in play at all times:

**`code_hash`** — the Blake2b-256 hash of the script's compiled RISC-V **bytecode**. This is a CKB protocol concept. It appears in every on-chain `Script` struct as the `code_hash` field. When a wallet or signing tool encounters an unfamiliar lock script in a transaction, `code_hash` is what it has. It is the primary key for everything — registry lookups, cell dep resolution, script identification.

**`idl_commitment`** — the Blake2b-256 hash of the **IDL JSON bytes**. This is a convention introduced by `ckb-idl-derive`. It is embedded as the last 32 bytes of the script's code cell data at deploy time. It has nothing to do with the bytecode hash. Its sole purpose is to let a verifier confirm that a candidate IDL document received from any source is authentic — the same bytes the script author published when they deployed the script.

The relationship is:

```
code_hash       -> registry lookup    -> receive IDL JSON bytes
idl_commitment  -> verify(idl_bytes)  -> trust that the IDL is authentic
```

The registry is keyed by `code_hash`. The verifier checks the `idl_commitment`. They are independent steps. Conflating them is a category error.

---

### The Verification Gap

The `verify` method itself was straightforward to implement. It takes raw IDL bytes and the code cell's data, extracts the trailing 32 bytes as the `idl_commitment`, hashes the received IDL bytes, and asserts equality. The implementation landed cleanly with a full property-based test suite — hash-matches pass, hash-mismatches fail with `IdlError::HashMismatch`, short data fails with `IdlError::InsufficientData`.

But the moment the implementation was done, the gap became obvious: the `fetch` method makes an HTTP call to `{indexer_url}/idl/{code_hash_hex}`. For this to work in practice, something has to be running at that URL serving IDL JSON keyed by `code_hash`. There is no such thing in the CKB ecosystem today. Every lock script author generates an `idl.json` at build time and has nowhere to put it that a wallet or signing tool can find programmatically.

This is the registry problem, and it turns out to be harder than implementing the verifier.

---

### What a Registry Actually Needs to Do

A registry for CKB IDLs is not just a file server. The requirements, once you think them through carefully, are:

**1. Content addressing, not location addressing.** A wallet looking up an IDL by `code_hash` cannot trust the URL it queries. It needs to know that the bytes it receives are exactly what the script author published. The `idl_commitment` in the code cell provides this anchor — the registry is just the distribution mechanism. The verification step is what makes the IDL trustworthy regardless of where it came from.

**2. Multiple distribution paths.** A registry that is down means signing tools cannot validate witnesses. For something that sits in a critical path of transaction construction, a single centralised server is a liability. The right design has multiple fallback layers: on-chain registry cell, IPFS/content-addressed storage, script author's own server, and local cache — all speaking the same interface so the client does not care which one answered.

**3. `code_hash` as the primary key.** The IDL is tied to a specific bytecode hash, not to a script name or version string. This is the right primary key because it is what a wallet actually has when it encounters an unfamiliar script in a transaction. A registry keyed by anything other than `code_hash` is not solving the right problem.

For now, a minimal HTTP server was built to stand in for a real registry during development. It stores IDLs in memory keyed by `code_hash` hex, exposes `GET /idl/:code_hash` (which is the exact endpoint `IdlClient.fetch` calls) and `POST /idl/:code_hash` (for registration). This is enough to test the full fetch-verify-validate flow end to end without a production registry. It lives in `ckb-idl-client/registry/` as an Express server.

---

### The Structural Validation Layer: `validate_witness_bytes`

With fetch and verify working, the third piece was implementing the structural decoder in `ckb-idl-client`: `validate_witness_bytes`. This is the function that sits between "I have a trusted IDL" and "I have a transaction ready to sign."

Given a slice of `WitnessField` from the IDL and a raw byte buffer, it:

- Decodes each field in declaration order using the same length-prefix wire format as `ckb-idl-derive`'s `from_witness_args`
- Asserts no trailing bytes remain after all fields are consumed
- Returns a `Vec<ValidatedField>` where each entry carries the field name, type, required flag, and the decoded value

The key design point: this is **Tier-1 (structural) validation only**. It does not check whether a secp256k1 signature is cryptographically valid. It does not know what timestamp the block will have. It cannot verify that an extra payload's hash matches a commitment in args. All of those are Tier-2 (semantic) checks that require transaction context.

What Tier-1 validation does catch — and catches immediately, before any transaction is constructed — is: wrong field count, wrong field sizes, missing required fields, and incorrect wire encoding. These are the mistakes that currently manifest as failed VM executions, discovered only after a transaction is submitted to the network.

The error type mirrors `ckb-idl-types`' `WitnessError` but is scoped to the client:

- `FieldTooShort { field, expected, got }` — buffer ran out during a field
- `UnknownType { field, type_ }` — IDL uses a type the client does not know how to decode
- `TrailingBytes { trailing, field_count }` — leftover bytes after all fields decoded

A full property-based test suite was written for `validate_witness_bytes`, covering round-trip fidelity (arbitrary valid wire buffers always decode correctly), truncation invariants (any buffer shorter than required for a field always produces `FieldTooShort`), and trailing-bytes rejection.

---

### PSCT: The Larger Picture

All of this — the registry, the verifier, the structural validator — is infrastructure for a concrete goal: **Partially Signed Cell Transactions (PSCT)**.

The CKB analogue of Bitcoin's PSBT is a signing flow where:
1. A transaction is built with inputs that reference live cells
2. Each input has an associated lock script identified by `code_hash`
3. One or more signers must attach witnesses before the transaction can be submitted
4. In a multi-signer scenario, witnesses are accumulated across parties before any party submits

The failure mode PSCT exists to prevent: a partially-signed transaction is handed off to the next signer, who attaches their witness using a different encoding assumption or wrong field ordering, and the whole transaction fails when it hits the network. By the time this is discovered, UTXOs may have been consumed in other transactions and the state is hard to unwind.

The IDL solves this by giving every party in the signing flow the same machine-readable description of what a valid witness looks like for each input's lock script. The flow becomes:

1. Transaction builder fetches the IDL by `code_hash` from the registry
2. Verifies the IDL against the `idl_commitment` in the code cell data
3. Proposes a witness buffer for each input
4. Calls `validate_witness_bytes` — structural check passes before any crypto
5. Attaches the structurally-valid witness to the PSCT and passes it to the next signer
6. Each subsequent signer repeats steps 1-4 before accepting the partial transaction

The result is that structurally wrong witnesses are rejected at step 4, before they ever reach a signer.

This week, the PSCT flow was demonstrated end-to-end in `ckb_lock_script/tests/` using `ckb-testtool`. Two test suites were written:

- `simple_lock_tests` — exercises the single-field `preimage: Vec<u8>` witness
- `timelock_lock_tests` — exercises the three-field `[u8; 65] + u64 + Vec<u8>` witness

Each test suite contains both pure structural validation tests (no VM) and full flow tests (structural check then VM execution). The tests deliberately demonstrate the PSCT boundary: tests named `test_psct_passes_but_vm_rejects_*` show that structural validation passes for semantically wrong witnesses (zero signature, wrong timestamp, mismatched extra payload hash), because structural validation only checks the wire format. The semantic rejection happens in the VM, which is where it belongs.

---

### What Comes Next

The immediate open problems, in order of urgency:

**1. The registry needs a persistence layer.** The current in-memory Express server loses all data on restart. The minimum viable production registry needs a key-value store keyed by `code_hash`. A SQLite database or a simple file-per-hash layout would both work and keep operational complexity low.

**2. The IDL needs to record its wire encoding.** The `idl.json` currently says `"type": "bytes"` for a `Vec<u8>` field but says nothing about how that field is framed on the wire. A consumer reading the IDL cannot decode a witness buffer without knowing the encoding. Adding `"encoding": "ckb-idl-v1-length-prefix"` at the top level closes this gap.

**3. PSCT needs an `encode` counterpart to `from_witness_args`.** Every PSCT test right now manually encodes the witness buffer before calling `validate_witness_bytes`. This duplicates the encoding logic and introduces drift risk. The `#[derive(CkbWitness)]` macro should generate a `to_witness_bytes` method on the struct. With both `from_witness_args` and `to_witness_bytes` generated, the round-trip becomes testable with a single property: `decode(encode(x)) == x`.

**4. The registry submission step needs to be tied to the build pipeline.** After `make build` generates `idl.json`, the developer still has to manually `POST` it to the registry using the deployed contract's `code_hash`. This should be automatic — either a post-build hook or a `cargo ckb-idl publish` subcommand that reads the generated `idl.json`, computes the `code_hash` of the deployed binary, and submits them in one step.

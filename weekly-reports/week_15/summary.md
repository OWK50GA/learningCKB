# Weekly Progress Report
## Week 15: Generalisation, Test Vectors, and Protocol Readiness

Week 14 ended with one working script on testnet and a clear next step: show the IDL system generalises beyond `simple-lock`. This week the deployer was refactored to support multiple scripts without duplicating infrastructure, a second script (`timelock-lock`) was wired up end-to-end, and the testing methodology was upgraded from ad-hoc assertions to a formal test vector file — the same approach used in Bitcoin BIPs.

---

### What Was Built

**Generic deployer infrastructure**

The deployer had grown a tight coupling between transaction mechanics and script-specific logic. That was cleaned up. Two generic functions now handle the heavy lifting:

- `create_locked_cell_generic` — takes any pre-built `lock_args` bytes and creates a cell locked by any script. The script-specific part (computing the args) is extracted into a wrapper per script.
- `spend_locked_cell` — handles all three-step transaction construction: balance the fee-payer inputs, inject the locked cell as input[0] with its witness and code dep, then re-sign the secp256k1 fee-payer inputs against the final transaction hash. Any script can use this by providing its pre-encoded witness bytes and any extra cell deps it needs.

A shared `verify_and_validate` helper runs the IDL commitment check and structural witness validation for any script, before any transaction is constructed.

**`timelock-lock` support**

With the generic infrastructure in place, supporting `timelock-lock` required only writing the script-specific parts:

- `create_timelock_cell` — builds the 65-byte args: compressed secp256k1 pubkey at `[0..33]` and an optional 32-byte extra-payload commitment at `[33..65]`.
- `spend_timelock_lock` — encodes the three-field witness (65-byte signature, u64 timestamp, variable-length extra), builds a draft transaction to extract the signing hash, signs it with the provided key, then calls `spend_locked_cell` with the final witness bytes.
- Wire encoding helpers `encode_bytes_field` and `encode_timelock_witness` are now shared and can be reused for future scripts.

New CLI commands: `deploy-timelock-lock`, `create-timelock-cell`, `spend-timelock-lock`.

**Test vectors**

A `test-vectors.json` file was added to the `ckb-idl-client` repository. This is a language-independent canonical specification of the wire format — 16 named cases covering every field type, every error condition, and both scripts. Any reimplementation of the decoder (TypeScript wallet, Python SDK, Go indexer) must produce identical results for every vector.

The vector runner is a standard Rust test that loads the file and runs every case through `validate_witness_bytes`. It runs in CI like any other unit test.

**Integration test suite fixed**

The `ckb_lock_script` test suite had a broken `timelock_lock_tests.rs` — several test calls had stale signatures from an earlier version of the helper functions. All tests were fixed and the suite compiles cleanly.

The test structure now separates two concerns clearly:
- PSCT structural validation tests (no VM): run in `ckb-idl-client`, language-independent, fast
- Full VM execution tests: run in `ckb_lock_script/tests` using `ckb-testtool`, require the compiled contract binary

---

### The PSCT / Semantic Boundary

This week's tests made the design boundary explicit in a way that is worth documenting. The `timelock-lock` tests demonstrate it clearly:

A witness with a zero signature passes structural validation — 65 zero bytes is a valid `secp256k1_sig` field. The VM then rejects it with error code 11 (`SignatureInvalid`). A witness with an unlock time far in the future also passes structural validation — the timestamp is correctly encoded. The VM rejects it with error code 12 (`TimelockNotMet`).

This is correct behaviour. The IDL client checks structure. The VM checks semantics. The point of the IDL system is to catch malformed wire encodings before the transaction is submitted — not to replace the VM. A wallet that sends a structurally valid but semantically wrong transaction has done its job: it correctly encoded the witness. The script then decides whether the conditions are met.

---

### Focus Shift

The `ckb_lock_script` repository is now in a maintenance state. The active development focus shifts to two things:

**`ckb-idl-client`** — the wire format specification, the test vectors, and the client library that wallets and tooling would actually import. That is the artifact that matters for wider adoption, and it has no dependency on any specific lock script.

**`ckb-idl-derive`** — the type registry needs to be treated as a deliberate decision, not just whatever was convenient for the first two scripts. The current set (`u8`, `u32`, `u64`, `[u8; 33]`, `[u8; 64]`, `[u8; 65]`, `Vec<u8>`) covers secp256k1 and simple byte fields. What's missing: `[u8; 32]` for hash fields, `u128` for token amounts, and a plan for Molecule-encoded types. The right approach is to survey what real CKB scripts actually put in their witnesses before extending the registry — adding types that no script uses is noise.

---

### What Comes Next

1. A TypeScript implementation of the wire format decoder, run against the same `test-vectors.json` — this proves the vectors are language-independent and that the spec is complete enough to reimplement from
2. A written specification document (RFC-style) describing the IDL commitment scheme, the wire format, and the PSCT validation contract — the precursor to any formal submission
3. Deciding on the submission venue: a CKB RFC, a mention in the Nervos forum, or direct contact with the ckb-sdk maintainers

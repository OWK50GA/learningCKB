# Weekly Progress Report
## Week 12: Reconnecting with the IDL Work — Brainstorming the Real Demo

No implementation this week. I had to step back from the work around Fiber I was doing (prompted by the Hackathon), back into the project I was already building. The time went into a focused design conversation: stepping back from the accumulation of IDL infrastructure built over the past several weeks, and asking a sharper question — what is the concrete thing I actually want to demonstrate, and what is the shortest path to demonstrating it?

---

### The Confusion That Needed Clearing

After building `ckb-idl-derive`, `ckb-idl-client`, `ckb-idl-types`, the registry server, the PSCT test suites, and the `validate_witness_bytes` implementation, I had drifted away from the original motivation. The pieces were real and working, but the end-to-end connection which is what a person would actually run to see the value — had not been built yet.

The question I came back to was: "I want to write a script, derive its IDL, and use that IDL in another place to verify the script. Does that make sense?"

It does. But "another script" was the ambiguous word. Two things that phrase could mean:

**Option A — Off-chain, at transaction formation time.** You have a lock script deployed. You have its IDL. You are building a transaction that spends a cell locked by it. Before you attach the witness to the transaction, you use the IDL to check that the witness bytes are correctly formed. If the check fails, the transaction is never built, never broadcast, never fails on-chain.

**Option B — On-chain, inside a type script.** You write a separate CKB type script whose job is to enforce that the witness of any transaction spending a cell carrying it conforms to the IDL of the cell's lock script. This is more novel, but it requires the IDL to be embedded or accessible on-chain, which is architecturally harder.

Option A is what I want to build first. It is also what is already partially demonstrated in the `ckb_lock_script/tests/` suites — but those use `ckb-testtool`, a simulated local environment. The real demo runs against testnet, with a transaction that is actually submitted.

---

### Revisiting the CKB Transaction Model

Part of the confusion came from a fuzzy mental model of how lock scripts, type scripts, witnesses, and args relate to each other. The conversation clarified this:

**Lock script** is the padlock. It runs when a cell is being spent. It answers "is this spend authorized?" If it returns 0, the spend is approved.

**Args** is the combination baked into the padlock at cell creation time. For a secp256k1 lock, args is the public key hash. It is set once and never changes. It is part of the cell's identity.

**Witness** is the key the spender provides at spend time. It is not stored in the cell. It is attached to the transaction. The lock script reads the witness and checks it against args. For simple-lock, the witness is a preimage whose hash must match the hash stored in args. For timelock-lock, the witness is a 65-byte signature plus a u64 timestamp plus an optional payload.

**Type script** is a separate script that also runs during the transaction, but it enforces rules about the shape and content of cells — inputs and outputs — regardless of who is spending. It is how tokens (sUDT), NFTs, and other stateful primitives are built on CKB.

The IDL's role in all of this: the IDL is a machine-readable description of what the witness must contain and how it is encoded. It does not run on-chain. It is consumed off-chain — in wallets, SDKs, and transaction builders — to catch malformed witnesses before they ever reach the VM.

---

### What the Real Demo Looks Like

The deployer in `ckb_lock_script/deployer/` already builds and submits real transactions to CKB testnet using `ckb-sdk`. It deploys the sUDT script and mints tokens. The pattern is established.

The demo I want to build is a `spend` binary in that same deployer crate that:

1. Accepts a preimage as input (the "key" for a simple-lock cell)
2. Encodes it as a wire-format witness (`4-byte LE length prefix + payload bytes`)
3. Loads the `simple-lock` IDL from the generated `idl.json`
4. Calls `IdlClient::validate_witness_bytes` — structural validation before anything touches the network
5. If validation fails, prints a clear error and exits — the transaction is never formed
6. If validation passes, wraps the witness in `WitnessArgs`, builds the spend transaction using `ckb-sdk`, and submits it to testnet

The demo then needs to be run twice:
- Once with a correctly-formed witness — it passes validation and the transaction goes through
- Once with a deliberately malformed witness (e.g. raw bytes with no length prefix, or wrong byte count) — it is rejected at step 4, before the network is ever contacted

That second case is the entire point. That is the IDL preventing a malformed transaction at formation time.

---

### The One Missing Piece for the Full Flow

The complete PSCT flow also includes `IdlClient::verify()` — checking that the IDL bytes you loaded match the `idl_commitment` embedded in the deployed script's code cell data. For that to work, the deployment step needs to append the `blake2b_256` of the `idl.json` to the script binary before deploying it.

That is a one-line change to `deploy_script.rs`:

```rust
// Append the IDL hash to the script binary
let idl_hash = blake2b_256(idl_json_bytes);
script_binary.extend_from_slice(&idl_hash);
```

For the initial demo this step will be skipped — the IDL will be loaded directly from disk and `verify()` will not be called. This is explicitly a pragmatic shortcut: the structural validation (`validate_witness_bytes`) is the valuable part to demonstrate, and the verification step can be layered on once the deploy-with-commitment flow is in place.

---

### What Comes Next

1. Add `ckb-idl-client` as a dependency of the `deployer` crate
2. Write a `spend.rs` binary that implements the flow described above
3. Run it against testnet — once correct, once malformed — and record the output
4. That recording is the demonstration of the full IDL-driven witness validation story from derive to deploy to spend

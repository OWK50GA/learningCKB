# Weekly Progress Report
## Week 14: Full IDL-Driven Spend Round-Trip Completed on Testnet

Week 13 ended with one missing step: there was no `simple-lock` cell on testnet to actually spend. The full round-trip was cut off just before the finish line. This week that gap was closed. The complete four-command flow ran successfully on testnet, from key generation through a confirmed spend transaction.

---

### What Was Completed

The goal was to run the full lifecycle of the IDL-driven spend system:

1. Generate a wallet keypair
2. Deploy the `simple-lock` binary (with IDL commitment baked in) to testnet
3. Create a cell locked by `simple-lock` using a known preimage
4. Spend that cell by supplying the preimage, with IDL verification and structural witness validation happening client-side before the transaction is submitted

All four steps succeeded.

---

### The Four Commands and What They Did

**Step 1 — Generate a keypair**

```
cargo run --bin keygen
```

```
─── Save these securely ───────────────────────────────
PRIVATE_KEY=<redacted>
TESTNET_ADDRESS=<redacted>
MAINNET_ADDRESS=<redacted>
───────────────────────────────────────────────────────
Fund your testnet address at:
https://faucet.nervos.org/?address=<testnet-address>

Paste into your .env file:
PRIVATE_KEY=<redacted>
TESTNET_ADDRESS=<redacted>
CKB_RPC=https://testnet.ckb.dev:8114
```

This generates a fresh secp256k1 keypair. `PRIVATE_KEY` is the 32-byte private key in hex. `TESTNET_ADDRESS` is the CKB testnet address derived from it — this is what the wallet sees, and what the cell collector searches for when looking for fee-payer cells. After running this, the address was funded via the testnet faucet and the output was pasted into `.env`.

---

**Step 2 — Deploy `simple-lock` with IDL commitment**

```
cargo run --bin deployer -- deploy-simple-lock
```

```
Deploying from: <redacted>
RPC endpoint:   https://testnet.ckb.dev
Frozen IDL written to: simple-lock-idl.deployed.json
Script size: 55344 bytes
Required capacity: 55444 CKB
Deployment tx hash: 0x654d1b7a618c156b6e867f396a55b1af2549bf9fca39e4c9e83531eaf9bee1b0
Code cell Outpoint: 654d1b7a618c156b6e867f396a55b1af2549bf9fca39e4c9e83531eaf9bee1b0:0x0
Code outpoint tx: 0x654d1b7a618c156b6e867f396a55b1af2549bf9fca39e4c9e83531eaf9bee1b0
```

**`0x654d1b7a618c156b6e867f396a55b1af2549bf9fca39e4c9e83531eaf9bee1b0`** — this is the deployment transaction hash. It uniquely identifies the transaction on-chain that created the code cell. The code cell outpoint is this hash at index `0` — meaning the first output of that transaction. This outpoint is the permanent on-chain address of the `simple-lock` binary.

What was deployed is not just the RISC-V binary. The deployer reads `contracts/simple-lock/idl.json`, computes a SHA-256 hash of those exact bytes, and appends the 32-byte hash to the end of the binary before submitting. The code cell data is therefore `[binary || sha256(idl.json)]`. The frozen copy of the IDL written to `simple-lock-idl.deployed.json` is the exact file whose hash was baked in.

---

**Step 3 — Create a cell locked by `simple-lock`**

```
cargo run --bin deployer -- create-locked-cell \
  654d1b7a618c156b6e867f396a55b1af2549bf9fca39e4c9e83531eaf9bee1b0 \
  68656c6c6f
```

```
Locked cell created.
Outpoint: 0x9a9ad2d9acf03e6ab9a651a6f4a55979a1cd8888cb083ae9316f058f5dcb3ca9:0x0
Lock args (preimage hash): 2da1289373a9f6b7ed21db948f4dc5d942cf4023eaef1d5a2b1a45b9d12d1036
Preimage hex (needed for spend): 68656c6c6f
```

The two arguments here are:
- **`654d1b7a...`** — the code cell tx hash from step 2, used to look up the deployed binary and derive its `code_hash`
- **`68656c6c6f`** — the preimage in hex. Decoded: `hello`

**`0x9a9ad2d9acf03e6ab9a651a6f4a55979a1cd8888cb083ae9316f058f5dcb3ca9`** — the outpoint of the newly created locked cell. This cell's lock script is:

```
code_hash: blake2b_256(code_cell_data)  →  d300d5a32a4c852b7b1dce5d8ebb2ffc19b17e54ff12e6e2fd0ce103c3de9016
hash_type: Data1
args:       blake2b_256("hello")        →  2da1289373a9f6b7ed21db948f4dc5d942cf4023eaef1d5a2b1a45b9d12d1036
```

The script says: "to spend this cell, you must provide a witness whose preimage, when hashed, equals `2da128...`". The only valid preimage is `hello`.

---

**Step 4 — Spend the locked cell**

```
cargo run --bin deployer -- spend-simple-lock \
  654d1b7a618c156b6e867f396a55b1af2549bf9fca39e4c9e83531eaf9bee1b0 \
  68656c6c6f \
  ./simple-lock-idl.deployed.json \
  9a9ad2d9acf03e6ab9a651a6f4a55979a1cd8888cb083ae9316f058f5dcb3ca9
```

```
code_hash: d300d5a32a4c852b7b1dce5d8ebb2ffc19b17e54ff12e6e2fd0ce103c3de9016
IDL commitment verified — IDL is authentic.
Witness structurally valid. Decoded fields:
  preimage (bytes): Bytes([104, 101, 108, 108, 111])
Spend tx hash: 0xe713e2a58dacd3dee99767484ae4f0b35d4be7a7d1e3ca984a666e2cb483f672
Cell spent successfully.
```

The four arguments:
- **`654d1b7a...`** — code cell tx hash (same as before — points to the deployed binary)
- **`68656c6c6f`** — the preimage (`hello`) that unlocks the cell
- **`./simple-lock-idl.deployed.json`** — the frozen IDL file from step 2
- **`9a9ad2d9...`** — the locked cell outpoint from step 3

What happened before the transaction was submitted:

**`code_hash: d300d5a3...`** — derived by fetching the code cell data from chain and running `blake2b_256` over it. This is the identifier the VM uses to find the right bytecode when executing the lock script.

**IDL commitment verified** — the last 32 bytes of the code cell data (`sha256(idl.json)` appended at deploy time) were extracted and compared against `sha256` of the local `simple-lock-idl.deployed.json`. They matched. This proves the IDL file being used is the same one that was committed when the script was deployed — it hasn't been tampered with or regenerated.

**Witness structurally valid** — the proposed witness (`hello` as a length-prefixed byte sequence) was parsed against the IDL's field definitions before any network call. `Bytes([104, 101, 108, 108, 111])` is `hello` in ASCII. The IDL client confirmed the wire encoding is well-formed.

**`0xe713e2a58dacd3dee99767484ae4f0b35d4be7a7d1e3ca984a666e2cb483f672`** — the spend transaction hash. The locked cell at `9a9ad2d9...` is now consumed. The CKB VM ran the `simple-lock` binary on-chain, verified that `blake2b_256(witness.preimage) == args`, and accepted the transaction.

---

### Bugs Fixed This Week

Two transaction construction bugs were caught and fixed during this week's run:

**Bug 1: Missing `cell_dep` for the simple-lock script**

The first error was `ScriptNotFound` with the simple-lock `code_hash`. When the CKB VM executes a lock script, it looks for the bytecode in the transaction's `cell_deps` field. The SDK's `CapacityTransferBuilder` only adds cell deps for inputs it selects itself. Since the locked cell was injected into the transaction manually after `build_unlocked` ran, the builder never triggered the resolver for it — so the code cell dep was never added to the transaction. The fix was to explicitly append `code_cell_dep` to the transaction's cell deps list alongside the input injection.

**Bug 2: Secp256k1 signature over the wrong transaction hash**

The second error was `-31` from the secp256k1 system script (public key hash mismatch). `build_unlocked` collects fee-payer cells and signs them, producing a secp256k1 signature that covers the transaction hash at that moment. The locked cell was then prepended to the input list, changing the transaction structure and therefore the transaction hash. The old signature was now over a hash that no longer existed — the VM correctly rejected it.

The fix restructures the flow into three steps:
1. `build_unlocked` collects and balances fee-payer inputs
2. The locked cell input, its witness (the preimage), and the code cell dep are injected — rebuilding the transaction
3. `unlock_tx` is called on the final transaction to produce a fresh, correct secp256k1 signature

---

### What This Demonstrates

The system does two things that matter:

**Client-side IDL verification** — before submitting a transaction, the client confirms the IDL it is using matches what was committed on-chain at deploy time. A wallet cannot be tricked into encoding a witness for a different version of the script.

**Structural witness validation** — the proposed witness is parsed against the IDL field definitions before the network is contacted. Malformed witnesses are caught locally, not by the VM returning an opaque error code.

Both of these ran successfully on a real CKB testnet transaction. The VM accepted the transaction and the cell was spent.

---

### What Comes Next

This was the proof of concept on `simple-lock` — the simplest possible script. The next phase is to run the same flow on more complex scripts:

- **`timelock-lock`** — a script with multiple witness fields (preimage + timestamp), where structural validation checks that all required fields are present and correctly encoded
- A deliberately malformed spend attempt — submit a witness with the wrong encoding and confirm it is rejected at the structural validation step, before the transaction reaches the network

The goal is to show that the IDL system generalizes beyond the trivial case and provides real safety guarantees for scripts with richer interfaces.

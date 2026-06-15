# Weekly Progress Report
## Week 5: First Real-World Test of `ckb-idl-derive`

This week was about taking `ckb-idl-derive` out of the controlled environment of unit tests and running it against a real CKB lock script. The work split across three areas: setting up the test environment, getting the macro to compile and run on an actual script crate, and identifying what the tool gets wrong in practice.

---

### What We Tested With

The test subject was `simple-lock` — a hash preimage lock script written in Rust using `ckb-std`. The script logic is straightforward: it reads a 32-byte blake2b-256 hash from the script args and a variable-length preimage from the witness, hashes the preimage, and unlocks the cell if the hashes match.

To integrate `ckb-idl-derive`, three additions were made to the script crate:
- `ckb-idl-derive` added as a dependency in `Cargo.toml` (git dependency, no crates.io publish needed)
- An empty `build.rs` at the crate root to ensure Cargo sets `OUT_DIR`
- A `src/witness.rs` file with the `#[derive(CkbWitness)]` struct

The witness struct:

```rust
#[derive(CkbWitness)]
pub struct Witness {
    #[witness(description = "Preimage whose blake2b-256 hash must match the hash in script args")]
    pub preimage: Vec<u8>,
}
```

After resolving several build environment issues (detailed below), the macro ran successfully and produced this IDL:

```json
{
  "witness": [
    {
      "name": "preimage",
      "type": "bytes",
      "required": true,
      "description": "Preimage whose blake2b-256 hash must match the hash in script args"
    }
  ]
}
```

This is the first machine-generated IDL from a real CKB lock script. It is stored in `example-idls/simple-lock/`.

---

### Difficulties Encountered

**1. RISC-V toolchain**
The `make build` command inside a contract directory looks for `scripts/find_clang` relative to the contract folder, not the workspace root. The script doesn't exist at that path. The workaround is `make build CLANG=$(which clang)` from the contract directory. This is a friction point that any developer integrating `ckb-idl-derive` into a new script will hit.

**2. Stale git dependency cache**
When `ckb-idl-derive` was first referenced via `{ git = "..." }`, Cargo had an old checkout cached from before `proc-macro = true` was set in `Cargo.toml`. The build silently ignored the dependency with the warning `ignoring invalid dependency which is missing a lib target`. Required a manual `cargo update` to pull the latest commit.

**3. `no_std` and `Vec`**
The witness struct file is compiled as part of a `no_std` crate. `Vec` is not in scope without `extern crate alloc; use alloc::vec::Vec;`. The macro itself runs on the host and has no issue, but the struct definition in the contract crate needs the explicit alloc import. This is `no_std` boilerplate that is easy to miss.

**4. The workspace Makefile's sim build**
Running `make build CONTRACT=simple-lock` from the workspace root also tries to run `cargo build -p simple-lock-sim`, which doesn't exist. The fix is to run `make build` from inside the contract directory directly.

---

### Flaws Identified in the Current Version

Testing against a real script exposed several gaps:

1. **`"bytes"` has no framing** — `Vec<u8>` maps to `"bytes"` but the IDL says nothing about how the bytes are delimited in the witness. For single-field witnesses like simple-lock this is fine. For multi-field witnesses with more than one variable-length field, the IDL is incomplete.

2. **The const name collides** — `_CKB_WITNESS_IDL_PATH` is always the same name. Multiple structs in the same module scope cause a compile error. The workaround is module isolation, but the const should be named after the struct.

3. **One `idl.json` per crate, last writer wins** — all macro invocations in a crate write to the same file. A crate with multiple witness structs loses all but the last IDL.

4. **No `[u8; 32]` mapping** — the most common fixed-size byte array in CKB (blake2b-256 hashes) is not in the type registry. The simple-lock args are `[u8; 32]`. If that appeared in a witness struct it would fail with an unrecognised type error.

5. **The struct is disconnected from the parsing code** — there is no enforcement that the `#[derive(CkbWitness)]` struct matches what `load_witness_args` actually reads. The IDL is only as correct as the struct definition.

6. **Silent failure without `build.rs`** — if the crate has no `build.rs`, `OUT_DIR` is not set and the macro fails silently (or with a confusing error). The developer has no indication that the IDL was not written.

---

### Scalability Assessment

The core concept is sound and scales well. A proc-macro that runs at compile time with zero runtime cost is the right primitive for this. The `example-idls/` folder pattern — one subfolder per script, machine-generated IDL alongside the witness struct — is a clean convention that any collaborator can follow.

The gaps are real but bounded. Items 2, 3, and 4 above are straightforward to fix in the next iteration. Item 1 (framing) is a harder design problem that needs a `layout` hint attribute. Item 5 is a documentation and convention problem, not a technical one.

The main scalability risk is adoption. The tool is only useful if script authors use it, and the current integration story has too much friction (RISC-V toolchain, `build.rs` requirement, `no_std` boilerplate). Making the integration story one line in `Cargo.toml` and nothing else is the right goal for v2.

---

### Plan Going Forward

**Immediate (before RFC submission):**
- Fix the const name collision (name it after the struct)
- Add `[u8; 32]` → `"bytes32"` to the type registry
- Document the `no_std` + `extern crate alloc` requirement clearly

**Short term (with collaborator):**
- Collect more real script IDLs using the workflow established this week
- For each script: link the source repo, document the witness layout, note any type registry gaps
- Use those gaps to prioritise the next registry additions

**Medium term:**
- Design the `layout` hint attribute for variable-length field framing
- Decide on the per-struct output strategy (named files vs appended JSON)
- Write tests against real scripts, not just synthetic inputs

**Longer term:**
- Extend the IDL format to cover `args` and `cell_data` (type scripts)
- Begin work on the PSCT format once the IDL RFC is accepted by the community
- Build the script registry / package manager that indexes deployed scripts by code hash

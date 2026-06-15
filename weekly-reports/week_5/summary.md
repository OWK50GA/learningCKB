# Weekly Progress Report
## Week 5: First Real-World Test of `ckb-idl-derive`

This week was about taking `ckb-idl-derive` out of the lab and running it against a real CKB lock script. The goal was simple: does the macro actually work when applied to a script that compiles to RISC-V and runs on CKB-VM?

The answer is yes — with some friction along the way.

---

### What We Tested On

The test subject was `simple-lock`, a hash preimage lock script written in Rust using `ckb-std`. The script logic is straightforward: the cell can be spent by anyone who provides a preimage such that `blake2b_256(preimage) == script.args[0..32]`. The expected hash is stored in the script args at deploy time; the preimage is provided in the witness lock field at spend time.

This is a good first test case because:
- It has a real, meaningful witness field (`Vec<u8>` preimage)
- It uses `ckb-std` and compiles to RISC-V, so it exercises the full toolchain
- The witness structure is simple enough that the generated IDL is easy to verify by inspection

The generated IDL:

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

Correct. One field, `Vec<u8>` → `"bytes"`, required, description from the `#[witness(...)]` attribute. The macro ran during the RISC-V release build and wrote `idl.json` to `OUT_DIR`.

---

### What Had to Be Done to the Script

Three additions to the script crate, none touching the script logic:

1. `ckb-idl-derive` added as a git dependency in `Cargo.toml`
2. An empty `build.rs` at the crate root (so Cargo sets `OUT_DIR`)
3. A new `src/witness.rs` file with the `#[derive(CkbWitness)]` struct, declared as `pub mod witness` in `main.rs`

The witness struct itself:

```rust
extern crate alloc;
use alloc::vec::Vec;
use ckb_idl_derive::CkbWitness;

#[derive(CkbWitness)]
pub struct Witness {
    #[witness(description = "Preimage whose blake2b-256 hash must match the hash in script args")]
    pub preimage: Vec<u8>,
}
```

The `extern crate alloc` and `use alloc::vec::Vec` are required because the script crate is `no_std`. `Vec` is not in scope by default in a `no_std` environment — it lives in the `alloc` crate.

---

### Difficulties Encountered

**1. `no_std` and `Vec`**

The witness struct file is compiled as part of the `no_std` script crate. `Vec` is not available without explicitly importing it from `alloc`. This is a friction point that every script author will hit when adding `ckb-idl-derive` to a `no_std` crate. It's not a bug in the macro, but it is something that needs to be documented clearly.

**3. RISC-V toolchain**

The `make build` command in the contract's Makefile uses a `find_clang` script to locate the C cross-compiler. The script lives at the workspace root but the contract Makefile looks for it relative to the contract directory, so it fails with a "No such file or directory" warning. The workaround is to pass `CLANG=$(which clang)` explicitly: `make build CLANG=$(which clang)`. This is a toolchain setup issue, not a macro issue, but it adds friction for new contributors.

**4. The `simple-lock-sim` error**

Running `make build CONTRACT=simple-lock` from the workspace root triggers `cargo build -p simple-lock-sim` — a native simulator package that doesn't exist for this contract. The fix is to run `make build` from inside the contract directory directly, or to create the simulator crate. For now, running from the contract directory is the workaround.

---

### Is This Scalable?

For the current v1 scope — lock scripts with witness fields that map to the seven blessed types — yes. The integration pattern is minimal (three files, no changes to script logic), the macro is fast (it runs in milliseconds during the host-side compilation step), and the output is deterministic.

The scalability concerns are at the edges:

**Type registry coverage.** The seven blessed types cover the most common CKB cryptographic primitives. But real scripts in the wild use `[u8; 32]` for Blake2b hashes, custom byte arrays of arbitrary sizes, and Molecule-encoded structs. Any of these will produce a compile error today. As more scripts are tested, the registry gaps will become clear. The fix is either adding more types to the registry or implementing an override attribute (`#[witness(type = "blake2b_hash")]`).

**`Vec<u8>` framing.** The `"bytes"` type in the IDL tells a consumer that a field is a variable-length byte blob, but not how it's framed in the witness. A script that uses `Vec<u8>` as a "remainder of witness" field is different from one that uses it as a length-prefixed blob, but the IDL looks identical. This is a known v1 limitation. It becomes a real problem when a PSCT coordinator tries to reconstruct the witness layout from the IDL alone.

**One IDL file per crate.** The macro always writes to `$OUT_DIR/idl.json`. If a crate defines multiple witness structs (e.g. for different entry points), each invocation overwrites the previous file. The current workaround is to put each struct in its own module, but this is not enforced. A script with two entry points and two witness structs will silently produce an IDL for only the last one compiled.

**`no_std` friction.** Every script author needs to know to add `extern crate alloc` and import `Vec` from `alloc`. This is standard `no_std` Rust, but it's a stumbling block for developers coming from `std` backgrounds. The documentation needs to cover this explicitly.

---

### Plan Going Forward

The immediate next step is to hand `ckb-idl-derive` to a collaborator in the community with a set of real lock scripts to test against. The goal is to collect IDLs from scripts that exercise different parts of the type registry and surface the gaps.

For each script tested, the collaborator will:
1. Add the three integration files (`Cargo.toml` dep, `build.rs`, `witness.rs`)
2. Build the script
3. Copy the generated `idl.json` into `example-idls/` in the `ckb-idl-derive` repo with the script source link
4. Note any compile errors (unrecognised types, attribute issues) and report back

The results will drive the next iteration of the macro — specifically which types need to be added to the registry and whether the override attribute is needed sooner than planned.

In parallel, the LS-IDL RFC draft (written in week 2) will be prepared for community submission. The `simple-lock` IDL is the first concrete evidence that the compiler integration described in the RFC is implementable. The RFC and the crate will be submitted together.

The PSCT format design remains on hold until the IDL RFC is accepted. There is no point designing the PSCT witness construction protocol before the community has agreed on what an IDL looks like.

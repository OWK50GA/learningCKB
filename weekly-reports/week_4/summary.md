# Weekly Progress Report
## Week 4: Implementing `ckb-idl-derive` — Full Proc-Macro Crate

This week I completed the full implementation of `ckb-idl-derive`, the Rust procedural macro crate that was designed and bootstrapped in week 3. The crate is now a working, tested piece of infrastructure for the LS-IDL ecosystem proposed in week 2.

---

### What Was Built

The crate exposes `#[derive(CkbWitness)]`. When applied to a named-field struct, the macro runs entirely at compile time and does three things:

1. Inspects each field's Rust type and maps it to a blessed IDL type string via a hardcoded registry.
2. Reads any `#[witness(...)]` field attributes to collect `required` and `description` metadata.
3. Writes a `idl.json` file to `$OUT_DIR` and emits a `const _CKB_WITNESS_IDL_PATH: &str` pointing to it.

A CKB lock-script author can now write:

```rust
use ckb_idl_derive::CkbWitness;

#[derive(CkbWitness)]
struct MyWitness {
    sig: [u8; 65],
    #[witness(required = false, description = "sender public key")]
    pubkey: [u8; 33],
    #[witness(required = false)]
    extra: Vec<u8>,
}
```

At compile time this produces an `idl.json` like:

```json
{
  "witness": [
    { "name": "sig",    "type": "secp256k1_sig",    "required": true  },
    { "name": "pubkey", "type": "secp256k1_pubkey",  "required": false, "description": "sender public key" },
    { "name": "extra",  "type": "bytes",             "required": false }
  ]
}
```

---

### Module Breakdown

The crate is split into five focused modules, each with its own unit tests:

| Module | Responsibility |
|--------|---------------|
| `validate` | Rejects enums, unions, tuple structs, and unit structs with precise compile-time error messages |
| `registry` | Maps 7 Rust types to blessed IDL strings; errors on unrecognised types |
| `attr` | Parses `#[witness(required = ..., description = "...")]`; errors on unknown keys |
| `codegen` | Builds the `serde_json::Value` IDL object and emits the `_CKB_WITNESS_IDL_PATH` const |
| `io` | Writes `$OUT_DIR/idl.json`; handles missing `OUT_DIR` and filesystem write failures |

The public entry point in `lib.rs` chains these modules through an internal `impl_ckb_witness` function, converting any `syn::Error` to a `compile_error!` token stream so errors appear at the correct source location.

---

### Type Registry

The supported Rust → IDL type mappings:

| Rust type   | IDL type string     |
|-------------|---------------------|
| `u8`        | `"uint8"`           |
| `u32`       | `"uint32"`          |
| `u64`       | `"uint64"`          |
| `[u8; 65]`  | `"secp256k1_sig"`   |
| `[u8; 33]`  | `"secp256k1_pubkey"`|
| `[u8; 64]`  | `"schnorr_sig"`     |
| `Vec<u8>`   | `"bytes"`           |

---

### Testing

The project uses three layers of testing, totalling 39 tests:

**Unit tests (19)** — test each pure function directly: all 7 type mappings, all attribute parsing cases, all validation branches, and IDL construction edge cases.

**Property-based tests (6, using proptest)** — verify universal correctness properties across hundreds of randomly generated inputs:
- Property 1: `required` flag fidelity — the IDL `"required"` value always matches the attribute
- Property 2: Description round-trip — `description` strings survive serialisation byte-for-byte
- Property 3: IDL structural invariants — the JSON always has a `"witness"` array of the right length and element shape
- Property 4: Generated const presence — the emitted `TokenStream` always contains `_CKB_WITNESS_IDL_PATH: &str`
- Property 5: JSON round-trip — `to_string → from_str → to_string` is idempotent
- Property 6: Serialisation format consistency — all IDL outputs use the same format (compact)

**trybuild integration tests (6)** — compile test files and assert exact error messages:
- `pass/basic_struct.rs` — verifies a valid struct compiles successfully
- `fail/enum_input.rs` — `"CkbWitness can only be derived for structs"`
- `fail/union_input.rs` — `"CkbWitness can only be derived for structs"`
- `fail/tuple_struct.rs` — `"CkbWitness requires a struct with named fields"`
- `fail/unknown_type.rs` — unrecognised type error with field name
- `fail/unknown_attr_key.rs` — unrecognised attribute key error

---

### Connection to the Broader Vision

This crate is the first concrete piece of the LS-IDL ecosystem described in the week 2 RFC. The IDL it generates is the artifact that would be hashed and appended to a script's code cell at deployment time, enabling wallets, hardware signers, and block explorers to understand any IDL-compliant lock script without custom integrations.

The next steps in the ecosystem are:
- A script registry / package manager that indexes deployed scripts by their code hash and serves their IDLs
- A client library for fetching and verifying IDLs against the on-chain hash commitment
- Extending the macro to cover `cell_data` and `args` dimensions (not just `witness`) for full type-script support

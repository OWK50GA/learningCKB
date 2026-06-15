# SUMMARY

## Bridging the IDL–Runtime Gap

This week addressed the most significant developer experience problem in `ckb-idl-derive`: the complete disconnect between the witness struct and the runtime code that actually reads the witness. The struct existed only to generate a JSON file — it played no role at runtime, was not instantiated, and could not be used in lock script logic. The `witness.rs` pattern that emerged was confusing and redundant. This week closed that gap entirely.

### The Problem

Under the original design, a lock script author had to write two separate things that were logically one:

1. A `Witness` struct annotated with `#[derive(CkbWitness)]` — purely for IDL generation, ignored by the compiler at runtime.
2. A `check_hash` function that manually called `load_witness_args`, unpacked raw `Bytes`, and did byte-level manipulation entirely disconnected from the struct.

The struct was documentation. It did nothing. A developer looking at `check_hash` would have no way to know the struct existed, and looking at the struct would give no indication that it was related to the runtime byte parsing. This was the core problem: the IDL artifact and the runtime behaviour were authored separately, compiled separately, and had no enforced relationship.

### The Fix: `from_witness_args` Code Generation

The macro now generates a `from_witness_args` method directly on the annotated struct. The method loads the witness from the CKB transaction, deserialises each field in declaration order, and returns a fully populated instance of the struct. The struct is now live code — it is instantiated, it holds the actual witness data, and the lock script logic operates on its fields.

The developer experience after this change:

```rust
// main.rs — one file, no witness.rs, no raw byte wrangling

#[derive(CkbWitness)]
pub struct Witness {
    #[witness(description = "Preimage whose blake2b-256 hash must match the hash in script args")]
    pub preimage: Vec<u8>,
}

fn check_hash() -> Result<(), Error> {
    let witness = Witness::from_witness_args(0, Source::GroupInput)
        .map_err(|_| Error::MissingWitness)?;

    let actual = blake2b_256(witness.preimage.as_slice());
    // ...
}
```

The `witness.rs` file was deleted. The struct lives next to the logic that uses it, which is where it always should have been.

### Wire Format: Length-Prefixed Encoding

A key design decision was how to encode multiple fields within the raw witness bytes. Two approaches were considered:

- **Fixed-offset + remainder**: Fixed-size fields read at known offsets; the last `Vec<u8>` takes everything that remains. Simple, but constrains the struct to one variable-length field, which must be last. Reordering fields silently breaks existing witnesses.
- **Length-prefix**: Each `Vec<u8>` field is preceded by a 4-byte little-endian length. Fixed-size fields are read directly at their known sizes. Supports any number and ordering of variable-length fields. 4 bytes of overhead per variable field.

Length-prefix was chosen. It supports unrestricted struct shapes, the overhead is negligible compared to actual witness payloads, and the wire format is self-describing enough for external tools to decode from the IDL alone.

The `WireKind` enum was added to `registry.rs` to carry the encoding strategy per field:
- `FixedScalar { size }` — for `u8`, `u32`, `u64`
- `FixedArray { size }` — for `[u8; N]`
- `VarBytes` — for `Vec<u8>`

`FieldMeta` in `codegen.rs` was extended to carry this alongside the existing IDL type string.

### Error Handling: `WitnessError`

`from_witness_args` returns `Result<Self, WitnessError>` rather than a bare `i8` error code. The enum provides structured, actionable diagnostics:

```rust
pub enum WitnessError {
    Load(ckb_std::error::SysError),
    MissingLockField,
    FieldTooShort { field: &'static str, expected: usize, got: usize },
    TrailingBytes { consumed: usize, total: usize },
}
```

`FieldTooShort` includes the field name and byte counts, which is particularly valuable when debugging on a system with no debugger. The developer maps `WitnessError` to their own error enum via `.map_err()` — one line, no boilerplate.

### Companion Crate: `ckb-idl-types`

Proc-macro crates in Rust cannot export regular types — only proc-macros. `WitnessError` therefore lives in a new companion crate, `ckb-idl-types`, which is `no_std`-compatible and feature-gated. Consumer crates that run on CKB enable the `ckb-contract` feature to get the `SysError`-backed `Load` variant. The generated `from_witness_args` code references `::ckb_idl_types::WitnessError` by path — the macro crate itself has no runtime dependency.

### IDL Output Location: `CARGO_MANIFEST_DIR`

The original design wrote `idl.json` to `$OUT_DIR`, which Cargo only sets when a `build.rs` exists in the consumer crate. This meant every lock script had to include a trivial `build.rs` just to satisfy the macro. This was unnecessary friction.

The macro now writes to `$CARGO_MANIFEST_DIR/idl.json`. Cargo sets `CARGO_MANIFEST_DIR` for every crate during macro expansion, regardless of whether a `build.rs` exists. The IDL lands next to the crate's `Cargo.toml` — a predictable, stable location that does not change across rebuilds. All `build.rs` files were removed from consumer crates.

---

### Challenges

**1. `WireKind` and `FieldMeta` Coupling**

Extending `FieldMeta` to carry `WireKind` required updating every test that constructs a `FieldMeta` directly. The property test strategies in `codegen.rs` had to be rewritten to pair each IDL type string with its corresponding `WireKind`, since they are tightly coupled — `"secp256k1_sig"` is meaningless without knowing it corresponds to `FixedArray { size: 65 }`. This is a minor but real friction point: the IDL type string and the wire kind are logically two views of the same type mapping, but they are currently stored as separate fields. A future refactor could unify them into a single `TypeMapping` value that carries both representations.

**2. `no_std` Codegen Safety**

The generated `from_witness_args` method references `::alloc::vec::Vec` explicitly rather than just `Vec`, because the consumer crate may be `no_std` with only `alloc` available. Every generated import path had to be written as an absolute path to avoid relying on anything from the consumer's scope. This adds verbosity to the generated code but is necessary for correctness across both `std` and `no_std` targets.

**3. Proc-Macro Crate Type Restriction**

Rust's proc-macro crate type cannot export regular items — only the proc-macro functions themselves are visible to consumers. This forced the introduction of `ckb-idl-types` as a separate crate. While the split is clean conceptually, it means a consumer crate now has two dependencies (`ckb-idl-derive` and `ckb-idl-types`) instead of one. A developer reading the error type in their IDE will see it originate from `ckb-idl-types` while the derive macro comes from `ckb-idl-derive`, which may be mildly confusing. Clear documentation is the mitigation.

**4. Trybuild Test Harness Update**

The `tests/ui/pass/basic_struct.rs` trybuild test began failing after `from_witness_args` was added to the generated output, because the test crate does not have `ckb_std` or `ckb-idl-types` in scope. Fixing this required adding both as dev-dependencies of the macro crate so the trybuild harness can compile the generated impl. This is correct but means the macro crate's dev-dependencies now include `ckb-std`, which is a heavier dependency than the macro itself warrants. An alternative would be to gate `from_witness_args` generation behind a `ckb-contract` feature flag on the derive macro itself, but that adds surface area that is not yet warranted.

---

### Remaining Weaknesses and Future Improvements

**1. No Type Override Escape Hatch**

The type registry still only maps a hardcoded set of Rust types to IDL strings. Any type outside that set — `[u8; 32]` (a common Blake2b hash), `[u8; 20]`, a newtype wrapper, a Molecule-generated struct — causes a compile error with no way out. A developer who needs `[u8; 32]` to represent a lock script hash has no recourse today. The planned fix is a `#[witness(type = "blake2b_hash")]` override attribute that lets the developer supply the IDL type string explicitly. This unblocks the common case without requiring the registry to have an opinion on every possible byte-array semantic.

**2. Wire Format Is Not Recorded in the IDL**

The generated `idl.json` describes field names, types, and required flags — but it does not record the wire encoding format. A consumer reading the IDL today cannot mechanically decode a wire-format witness because the JSON gives no indication of how variable-length fields are framed. The fix is a top-level `"encoding"` key in the IDL:

```json
{ "encoding": "ckb-idl-v1-length-prefix", "witness": [ ... ] }
```

Until this is added, the IDL is an incomplete contract — it describes the logical layout but not the physical encoding.

**3. No `encode` Method Generated**

`from_witness_args` only goes one direction: deserialise from raw bytes into a struct. There is no corresponding `to_wire_bytes` or `encode` method generated on the struct. This matters for testing and for off-chain tooling: a test harness that wants to construct a valid witness buffer to pass to a lock script has to manually implement the length-prefix encoding, which duplicates logic and is easy to get wrong. Generating an `encode` method alongside `from_witness_args` would make the round-trip testable and give off-chain tooling a canonical encoder.

**4. `required = false` Fields Have No Runtime Enforcement**

The `required` flag is faithfully recorded in the IDL and in the `#[witness(...)]` attribute syntax, but `from_witness_args` ignores it entirely. There is no difference at runtime between `required = true` and `required = false`. The deserialization either succeeds or fails on byte availability — it does not treat an empty or absent field as valid for optional fields. The `required` flag is currently documentation only, which undermines its purpose. A proper implementation would allow optional `Vec<u8>` fields to decode a zero-length payload as `None` and require the field type to be `Option<Vec<u8>>` rather than `Vec<u8>`.

**5. Two-Crate Dependency Burden**

Every consumer crate currently needs two dependencies: `ckb-idl-derive` for the macro and `ckb-idl-types` for `WitnessError`. This is an ergonomic tax on every lock script author. The ideal developer experience is a single dependency. The standard Rust pattern for solving this is a facade crate that re-exports both — `ckb-idl` that depends on both sub-crates and re-exports the macro and the error type, so the consumer only writes one line. Alternatively, the `ckb-idl-derive` macro could emit a local error type definition inline in the generated code rather than referencing an external crate, which would reduce dependencies to one at the cost of making `WitnessError` a different type per crate.

**6. No Molecule Support**

Lock scripts that use Molecule-encoded witness types — including the standard `WitnessArgs` wrapper — cannot use `#[derive(CkbWitness)]` on any field that is a Molecule-generated type. These types are unrecognised by the registry and cause a compile error. This limits the practical applicability of the crate significantly, since Molecule is the canonical CKB serialisation format and many real scripts use `WitnessArgs` directly. Supporting Molecule requires either a naming convention or an explicit attribute like `#[witness(type = "WitnessArgs", encoding = "molecule")]`.

**7. `TrailingBytes` Error Is Too Strict for Extensibility**

The current `from_witness_args` decoder returns `WitnessError::TrailingBytes` if there are any leftover bytes after all fields are decoded. This is correct for a strict parser but makes forward compatibility impossible: a new version of a lock script that adds a field to the witness struct will produce trailing bytes when decoded by an older version of the script running against a newer witness. A more robust design would either ignore trailing bytes by default or provide a `#[witness(allow_trailing)]` opt-in for cases where forward extensibility matters.

**8. Single Witness Source (`lock` Field Only)**

`from_witness_args` always reads from the `lock()` field of `WitnessArgs`. CKB transactions have three witness fields: `lock`, `input_type`, and `output_type`. Scripts that use `input_type` or `output_type` witnesses — type scripts, for instance — cannot use the generated deserializer without modification. A `#[witness(source = "input_type")]` attribute or a parameter on the generated method would cover this without requiring structural changes to the macro.

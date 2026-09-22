# Weekly Progress Report

## Week 22: Expanding LS-IDL from a Flat Witness Registry to a Recursive Interface Model

This week was the most substantial implementation period for LS-IDL since the original procedural macro and client proof of concept. The work began as a type-registry extension: the early version of `ckb-idl-derive` could only describe a handful of flat witness fields (`u8`, `u32`, `u64`, three fixed cryptographic byte-array sizes, and `Vec<u8>`). That was enough to demonstrate derive → validate → commit, but not enough to model many witness layouts that real CKB scripts need.

The result is no longer just a larger list of primitive mappings. LS-IDL now has a path toward describing recursive witness interfaces: fixed-size byte data of arbitrary non-zero length, more integer types, optional fields, typed vectors, nested structures, tagged unions, and a host-side exporter capable of traversing those structures into a complete IDL document.

The implementation remains pre-release. Documents emitted by the macro and exporter now identify themselves as LS-IDL `0.1`; this is a draft schema marker, not a declaration that the public standard has been finalized.

---

### Structural Types and Semantic Labels

The first design change was to separate byte-level structure from application-level meaning.

Previously, a few array lengths had hard-coded semantic meanings:

- `[u8; 65]` → `secp256k1_sig`
- `[u8; 33]` → `secp256k1_pubkey`
- `[u8; 64]` → `schnorr_sig`

That is convenient for common cases, but it does not generalize. A `[u8; 32]` value might be a Blake2b hash, a script hash, a commitment, or simply opaque bytes. Rust’s type alone cannot determine that semantic intent.

The registry now uses structural defaults instead:

```text
[u8; 32]  → bytes_fixed_32
[u8; 65]  → bytes_fixed_65
[u8; N]   → bytes_fixed_N
```

The author can then add a semantic label deliberately:

```rust
#[witness(type = "blake2b_hash")]
pub digest: [u8; 32],
```

The decoder still uses the structural wire type. In the recursive exporter, the public `type` reflects the semantic label while `wire_type` preserves the structural representation required by a generic client to encode and decode safely.

This distinction is important for wallets. A wallet can display “Blake2b hash” rather than “32 opaque bytes,” while still knowing that exactly 32 bytes must be placed on the wire.

---

### Broader Scalar and Array Support

The registry now supports these scalar Rust types:

```text
u8, u16, u32, u64, u128
```

They map to the corresponding unsigned little-endian IDL types. All non-zero `[u8; N]` arrays are accepted as fixed byte sequences.

Zero-length arrays are explicitly rejected. Although `[u8; 0]` is valid Rust, accepting it as a vector element would allow a witness to declare an enormous element count while consuming no bytes per element. Rejecting `[u8; 0]` and `Vec<[u8; 0]>` removes that malformed-input CPU exhaustion path before code generation.

The registry retains `Vec<u8>` as the special `bytes` type: it is encoded as a four-byte little-endian byte length followed by the payload.

---

### Optional Witness Fields

Optional fields are now actual runtime behavior rather than IDL documentation only.

```rust
#[witness(required = false)]
pub memo: Option<Vec<u8>>,
```

The derive checks that `required = false` and `Option<T>` agree in both directions. It rejects a non-optional field marked optional, and an `Option<T>` field left as required.

The current wire convention is trailing exhaustion: an optional field decodes as `None` only when the previous fields have consumed the entire buffer. That makes the format deterministic, but imposes layout rules:

- optional fields must come last;
- nested optional values are rejected;
- `Option<Vec<T>>` for typed vectors is rejected;
- optional nested structs are rejected because they have no safe boundary under the current linear encoding.

These restrictions are intentional. They are preferable to accepting witness layouts that a wallet and script could interpret differently.

---

### Nested Witness Structures

The new `#[derive(CkbInnerWitness)]` derive allows a witness to be composed from reusable named structures.

```rust
#[derive(CkbInnerWitness)]
pub struct Authorization {
    pub signature: [u8; 65],
    pub unlock_after_ms: u64,
}

#[derive(CkbWitness)]
pub struct Witness {
    pub nonce: u16,
    pub authorization: Authorization,
}
```

`CkbInnerWitness` implements `WitnessFields`, which provides field decoding from a shared byte buffer and cursor. The parent witness delegates decoding to that implementation. The compiler enforces the trait relationship, so an arbitrary named Rust type cannot silently be treated as a valid wire structure.

The same boundary rules apply here. A nested struct must be the final field in its containing layout, because an inner struct may itself have trailing optional fields and there is no outer framing delimiter in the current encoding.

Runtime and UI tests now cover successful nested decoding, missing `WitnessFields` implementations, and the required ordering rule.

---

### Typed Vectors

LS-IDL now distinguishes raw bytes from vectors of typed elements.

```rust
pub keys: Vec<[u8; 33]>,
pub amounts: Vec<u64>,
```

These use a four-byte little-endian **element count**, followed by each encoded element. This is different from `Vec<u8>` / `bytes`, whose prefix is a byte length.

The first supported typed-vector surface is deliberately narrow: fixed-size scalar and byte-array elements are supported. Nested vectors, optional vector elements, and vectors of named nested structs are rejected until there is a framing design that makes every element boundary unambiguous.

This is an example of the project’s overall approach this week: add useful expressiveness, but do not describe formats that cannot be decoded consistently by generic tooling.

---

### Tagged Witness Unions

The new `#[derive(CkbWitnessUnion)]` derive supports an enum-style witness choice.

```rust
#[derive(CkbWitnessUnion)]
pub enum Authorization {
    #[witness(tag = 7)]
    Secp(SecpAuthorization),

    #[witness(tag = 42)]
    Multisig(MultisigAuthorization),
}
```

Each union is encoded as a four-byte little-endian tag followed by the selected variant’s inner witness fields. Variants must be single-field tuple variants; unit variants, named-field variants, and multi-field tuple variants are rejected at compile time.

The tag decision was revised during implementation. The initial idea of assigning tags from declaration order is convenient, but unsafe for a durable external ABI: inserting or reordering variants changes the interpretation of already encoded values. Explicit `#[witness(tag = N)]` values make the wire contract stable, reviewable, and independent of source ordering. Duplicate and missing tags are compile errors.

A parent witness marks a union field explicitly:

```rust
#[witness(union)]
pub authorization: Authorization,
```

This explicit marker is necessary because a Rust procedural macro cannot resolve arbitrary type paths and discover whether a named type implements the structure or union trait.

---

### Recursive LS-IDL Export

The most significant architectural addition was a host-side exporter crate: `ckb-idl-export`.

The original macro can inspect only the annotated top-level struct during compilation. It cannot inspect another struct or enum definition merely from a field type path, so it cannot reliably write nested fields or union variants into `idl.json` by itself.

To solve that limitation, the derives now emit allocation-free static schema metadata through `ckb-idl-types`:

```text
WitnessSchema
  └── StructSchema
        └── FieldSchema
              └── TypeSchema
                    ├── primitive / fixed bytes / bytes
                    ├── optional / vector
                    ├── nested struct
                    └── tagged union
```

The host exporter walks this graph recursively and creates the complete LS-IDL 0.1 JSON artifact. A union entry can therefore contain each explicit tag, variant name, and its inner field schema. A nested struct can contain its own field list rather than appearing only as an opaque `"struct"` marker.

For an advanced contract, authors add a small host export entry point:

```rust
// examples/export_idl.rs
ckb_idl_export::export_idl_main!(my_lock::witness::Witness);
```

Then generate the artifact with:

```bash
cargo run --example export_idl -- artifacts/idl.json
```

The exact exported bytes are the artifact intended for commitment at deployment time. The older macro-written flat file remains available as a compatibility path for simple flat witnesses, but recursive schemas should use the exporter output.

---

### Testing and Validation

The implementation work was accompanied by expanded validation rather than only happy-path compilation.

- Unit/property tests cover the extended registry, schema generation, stable JSON behavior, and path-aware wire-kind equality.
- UI tests cover invalid optional layouts, unknown attributes, invalid union shapes, zero-length arrays, and unsupported typed-vector elements.
- Runtime integration tests decode nested structures and tagged unions from actual byte buffers.
- Exporter integration tests verify that nested union variants and semantic type overrides appear in the recursive document with their structural `wire_type` preserved.

The workspace test suite passes across the derive crate, companion types crate, and exporter crate.

---

## Next Steps

The immediate next task is to turn the implementation into a deliberate LS-IDL 0.1 document specification before adding more surface area.

1. **Write the LS-IDL 0.1 specification.** Define the normative JSON grammar, recursive field representation, type and `wire_type` semantics, vector count encoding, optional-field convention, union tag rules, unknown-key policy, canonical examples, and exact-byte commitment requirements.

2. **Align all consumers with the draft schema.** Update the Rust and TypeScript client libraries so they can decode and encode the expanded type system and recursive exporter output. Extend canonical cross-language vectors for integers, fixed bytes, options, typed vectors, nested structs, unions, and malformed buffers.

3. **Decide the minimal wallet-facing metadata surface.** Revisit the planned struct-level metadata work with the specification in hand. The likely priorities are interface name/description, witness endpoint, script-argument schema, and precise signing requirements; source provenance and registry metadata should remain separate concerns.

4. **Automate the commitment lifecycle.** Once the export format is stable, add deployment tooling that validates the exact file, binds its SHA-256 digest to a clean executable, preserves the frozen artifact, and verifies the pair before deployment.

5. **Design Molecule as an encoding profile.** Molecule should be integrated as an additional, explicitly declared encoding path rather than forced into the primitive type registry. The LS-IDL metadata layer can still supply wallet-facing names, descriptions, semantic labels, and signing information.

6. **Begin registry work only after the artifact is stable.** A registry should discover and verify a stable committed LS-IDL artifact; it should not become the place where the document schema is still being decided.

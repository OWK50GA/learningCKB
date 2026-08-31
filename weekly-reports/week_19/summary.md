# Weekly Progress Report
## Week 19: CKBuilder Feedback and Deciding on Nested Struct Support

This week was short on code and long on thinking. The main event was reviewing the community feedback that came in on the CKBuilder issue, working out what it means for the project's direction, and making a concrete decision on the nested struct question that has been sitting open since the type registry was first designed.

---

### CKBuilder Issue Feedback

The project issue on the [CKBuilder programme tracker](https://github.com/Nervos-Community-Catalyst/CKBuilder-projects/issues/29) received its first substantive comment. The two points raised:

**1. The type registry is too narrow.**

The feedback was direct: the current set of seven types — `u8`, `u32`, `u64`, `[u8; 65]`, `[u8; 33]`, `[u8; 64]`, `Vec<u8>` — is enough for simple lock scripts but is not complete. Two-dimensional arrays and structs (nested types) were specifically called out as missing. This confirms what was already noted as a known gap in earlier summaries, but having it said externally makes it a higher-priority design question rather than a deferred concern.

This is fair. The registry was intentionally kept small for v0 to avoid over-designing before real usage patterns emerged. But "useful for some scripts" and "complete enough to be a real ABI system" are different bars, and the feedback is pointing at the second bar.

**2. The relationship to existing serialization ecosystems.**

The commenter pointed out that Molecule and Protobuf are IDLs in the broad sense — they define wire formats with a schema, which is exactly what LS-IDL does. They also noted that `bincode` and `borsh`, which are established Rust serialization libraries, are doing something structurally similar to what `from_witness_args` does: deriving serialization logic from a struct definition.

This is a useful framing. The honest answer is that LS-IDL is occupying the same conceptual space as these formats but is specifically designed for the CKB lock script context: the wire format is simple and fixed (no versioning envelope, no schema evolution), the IDL is committed on-chain so it cannot be changed post-deployment, and the goal is structural validation by external tooling rather than general-purpose serialization. Molecule in particular is the existing CKB-native format, and the lack of Molecule support in the type registry is the most significant practical gap. The commenter is right that these relationships should be acknowledged explicitly in the documentation.

---

### Deciding on Nested Struct Support

The feedback on two-dimensional arrays and structs, combined with the existing gap around hash types (`[u8; 32]`), made the decision clear: nested struct support via a `CkbInnerWitness` derive macro is the right next type system extension.

The design settled on:

- A new derive macro `CkbInnerWitness` (in the same `ckb-idl-derive` crate) that generates a trait implementation rather than `idl.json` or `from_witness_args`. The trait lives in `ckb-idl-types` and exposes the two things a nested struct needs to contribute: its IDL field list (for the parent's `idl.json`) and a cursor-based decode method (for the parent's `from_witness_args`).
- The parent `CkbWitness` macro, when it encounters a field whose type is a `Type::Path` rather than a primitive or known array, emits a call to that trait's decode method instead of inline byte logic. The compiler enforces the trait bound — if the user forgets `#[derive(CkbInnerWitness)]` on the nested type, the generated code does not compile, and the error message points at the missing impl.
- For the IDL, nested struct fields are inlined flat into the parent's witness array for v1. This keeps the wire format and the IDL consumer logic unchanged. A proper `"type": "struct"` with a nested `"fields"` array is the right long-term answer but requires coordinated updates across `ckb-idl-derive`, `ckb-idl-client`, and `idl-derive-ts` simultaneously. Flat inlining avoids that coordination cost for now.

The recursion point from `registry.rs` is where this change enters the codebase: `map_type` and `map_wire_kind` currently return errors for `Type::Path` segments that are not `u8`, `u32`, `u64`, or `Vec`. The extension is to add a branch that recognises any other `Type::Path` as a potential `CkbInnerWitness` type and emits the trait call path instead of a fixed byte-width decode.

This work is the next active implementation milestone after the TypeScript client.

---

### On the Molecule Question

The commenter's point about Molecule deserves a direct answer, because it comes up in the context of CKB development repeatedly.

Molecule is the right format for CKB on-chain data that needs to be read by multiple scripts at the protocol level. It handles schema evolution, has tooling, and is what the core CKB types use. For a lock script's witness — which is an interface contract between the script author and the spender, described once at deployment and fixed for the lifetime of the cell — Molecule's flexibility is mostly overhead. The current wire format (sequential fields, 4-byte LE length prefix for variable bytes) is simpler, self-describing given the IDL, and appropriate for the use case.

That said, many real lock scripts today already use `WitnessArgs` (a Molecule type) as their top-level wrapper. Supporting `WitnessArgs` as a known IDL type — not arbitrary Molecule schemas, just the standard CKB witness wrapper — would significantly improve practical applicability. This is on the radar but is not part of the nested struct work.

---

### Summary

The week produced two concrete outcomes: a clearer understanding of where the project sits relative to the existing ecosystem (and what that means for the documentation and design), and a firm decision to implement `CkbInnerWitness` as the next type system feature. The implementation starts next week.

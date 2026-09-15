# Weekly Progress Report
## Week 20: Mapping the Improvement Space for LS-IDL (Part 1)

This week was spent thinking through what LS-IDL could become beyond its current scope. The codebase is stable, the TypeScript side is built, and the registry collaboration is in motion. Before writing a grant proposal it makes sense to map out what an expanded LS-IDL would actually enable — not just as a feature list, but as a coherent story about why the interface description layer matters at all.

The ideas explored this week cluster around the foundational layer: what the IDL needs to express, and what tooling becomes possible once the expression is rich enough.

---

### Type Registry Extension

The most immediate and concrete improvement. The current registry has seven types, all chosen because they have a single unambiguous CKB meaning. The gaps are real: `[u8; 32]` hash fields, nested structs, arrays of structs, and Molecule-encoded types all appear in real scripts today and cannot be described by the current IDL.

The planned path is two-pronged. First, extend the flat registry with common additions: `[u8; 32]` as `hash256`, `u128`, and a general `bytes_fixed_N` for arbitrary fixed-length arrays. Second, introduce `CkbInnerWitness` as a derive macro for nested struct types, using a trait-based design so the compiler enforces the contract rather than requiring runtime checks. These two extensions together cover the majority of real-world witness layouts that the current registry cannot handle.

Molecule support is a longer-term question. It requires either a naming convention that maps Molecule-generated types to their schema names, or an explicit attribute override. It is not blocked on the trait design but it is a larger surface area change.

---

### Generic Lock Script Transaction Builders

Once the IDL describes a lock script's witness interface completely, it becomes possible to construct a generic transaction builder that does not have per-script knowledge. A builder receives the IDL, the field values from the caller, encodes them using the wire format described in the IDL, and produces the `WitnessArgs.lock` bytes. No script-specific code required.

This is the direct complement to `from_witness_args`. The script deserialises using the generated method; the builder serialises using the IDL. The missing piece today is that `ckb-idl-derive` generates only the deserialiser — there is no `encode` or `to_wire_bytes` method on the struct. Adding that closes the round-trip and makes generic builders possible without any additional specification work.

The CKB tooling today requires wallets and SDKs to implement per-script witness construction. That is friction that prevents new scripts from being used by existing wallets without a specific integration. A generic builder driven by the IDL removes that friction entirely.

---

### Generic Signing Interfaces

Related to transaction builders but at a different layer. A signing interface is not about encoding the witness — it is about deciding what to sign and how. The question LS-IDL raises here is: given that the IDL describes which fields a script requires, could a generic signing interface present those fields to a signer in a human-readable or structured way, rather than presenting raw bytes?

Concretely: a hardware wallet or a signing service today receives a transaction and has no structured way to present what is being signed to the user. If the lock script has an IDL, the wallet can look up the interface, decode the witness fields by name and type, and show the user "you are signing: `unlock_after_ms = 1700000000000`, `signature = [65 bytes]`" rather than a hex dump.

This requires the IDL to be available at signing time — which is exactly what the registry provides. The signing interface is therefore an application built on top of the registry, not a change to the IDL format itself.

---

### Off-Chain Transaction Validation

`ckb-idl-client` already implements structural witness validation against an IDL: given the field list and a raw wire buffer, it decodes field by field and returns typed values or a structured error. This is the foundational piece.

The extension is to move this validation earlier and make it part of the standard transaction submission flow. A wallet or SDK that knows the lock script's code hash can fetch the IDL from the registry, run `validate_witness_bytes` before the transaction is submitted, and surface field-level errors to the user before any on-chain fee is paid.

Today this fails silently on-chain. The transaction is submitted, the VM runs, the script returns an error code, and the user's fee is burned. Off-chain validation with structured errors is a qualitative improvement in developer and user experience that follows directly from the IDL being available and complete.

The gap is not in the validation logic — that already exists. The gap is in the registry endpoint that makes the IDL fetchable by code hash in production. Once the registry is live, off-chain validation becomes a one-line integration for any CKB SDK.

---

### External Signer Interfaces

Distinct from the generic signing interface above. An external signer is a separate process or device — a hardware wallet, a cloud HSM, a multi-party computation service — that holds private key material and signs transactions on request. The question is how such a signer knows what it is signing.

Today, an external signer for a CKB lock script either has hard-coded knowledge of the witness format (which means it only works with scripts it was specifically built for) or it signs blindly over the raw transaction bytes (which means the user has no visibility into what is being authorised).

LS-IDL provides a third option: the signer fetches the IDL for the lock script being spent, decodes the witness, and presents the structured field values to whatever approval mechanism it uses. This is directly analogous to how Ethereum's EIP-712 works — structured data signing where the schema is known and human-readable. LS-IDL is CKB's equivalent primitive for this.

The implication is that designing LS-IDL's type system with signing presentation in mind — not just wire encoding — is the right framing. Field names and descriptions in the IDL are not just documentation; they are the labels that appear to a user on a hardware wallet screen.

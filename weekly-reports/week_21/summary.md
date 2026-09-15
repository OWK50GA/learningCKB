# Weekly Progress Report
## Week 21: Mapping the Improvement Space for LS-IDL (Part 2) and the Registry Plan

Continuing from last week's exploration of foundational improvements, this week looked at the higher-level possibilities that an extended LS-IDL and a live registry would enable — composability, generated tooling, wallet interoperability, protocol evolution, and what it would mean for Fiber channels involving non-standard lock scripts.

The week closed with formalising the registry concept into a proposal document intended for DevRel feedback before a grant submission.

---

### Script-to-Script Composability

CKB scripts compose. A type script can verify the lock script on a cell. A lock script can check the type script of cells it is spending. A protocol can require that input cells carry a specific combination of lock and type scripts.

Today those relationships are expressed in code and nowhere else. There is no machine-readable way to say "this script expects to be used with that script" or "this protocol requires these three scripts to be present together." The only way to discover those relationships is to read the source.

An extended LS-IDL, or the registry model built around it, could express script dependencies explicitly. A registry record could state which other scripts a given script depends on, verifies, or is designed to compose with. This turns the registry from a lookup table into a dependency graph — a directed graph of CKB executable components where each node has an identity, an interface, and a set of edges to its dependencies.

The practical payoff is significant: a transaction builder that understands the dependency graph can automatically include the right `cell_deps` for a script group. A wallet that understands composability relationships can warn the user when a required companion script is absent from a transaction.

---

### Protocol Adapters

A protocol adapter is a piece of tooling that translates between LS-IDL's wire format and another encoding used by a specific ecosystem. The motivating case is Molecule: many CKB scripts use `WitnessArgs` as their top-level witness wrapper, which is a Molecule-encoded struct. Today LS-IDL cannot describe a Molecule-encoded field; a script that uses `WitnessArgs` cannot use `#[derive(CkbWitness)]` at all.

A protocol adapter would not require LS-IDL to natively understand Molecule. Instead, it would be a registered codec: the IDL field declares `"encoding": "molecule"` and a `"type": "WitnessArgs"`, and the adapter handles the encoding and decoding by delegating to the Molecule codec for that type.

This keeps the core IDL format simple while allowing real-world scripts that use existing serialisation conventions to participate in the ecosystem. The adapter layer is extensible: a future adapter could handle `bincode`, `borsh`, or any other encoding without changing the IDL schema itself.

---

### Generated Bindings

Once the IDL is a complete, machine-readable description of a script's witness interface, generating client bindings in arbitrary languages becomes straightforward. The IDL describes field names, types, required flags, and encoding — everything a code generator needs to emit a typed struct and a wire encoder/decoder in any target language.

The TypeScript `idl-derive-ts` package is already a hand-written version of this for the script side. The natural extension is a code generator that takes an `idl.json` and emits TypeScript, Python, Go, or other language bindings automatically — without requiring a port of the derive macro to each language.

Generated bindings make it possible for a wallet developer who has never seen the lock script source to integrate with it in their language of choice by simply running a generator against the published IDL. This is one of the core DX promises of the registry: you find the script, you fetch the IDL, you run the generator, and you have a typed client.

The scope of generated bindings depends directly on the type registry. A richer registry means a richer binding surface. This is one of the reasons the type registry extension is the highest-priority implementation work.

---

### Wallet Interoperability

The most user-visible application of everything above. A wallet that implements the LS-IDL client flow — fetch IDL by code hash, verify commitment, decode witness fields, validate before submission — works with any script that has a published IDL, without any per-script integration work.

Today, adding support for a new lock script in a wallet means a custom integration: understanding the wire format, adding encoding logic, deciding how to present the fields to the user. With LS-IDL, that integration collapses to: fetch the IDL, render the fields using the generic wallet UI, encode using the generic wire encoder.

The precondition is that the IDL is complete and accurate, the registry is live and queryable, and the wallet implements the client protocol. The third point is an adoption question — the registry and the client library do not force any wallet to use them. The value proposition has to be compelling enough that wallet developers choose to implement it.

The generic signing interface and external signer support explored last week are the parts of this story that are most likely to drive wallet adoption, because they solve a problem wallets have today — they cannot meaningfully present CKB witnesses to users — rather than asking them to do extra integration work.

---

### Safe Protocol Evolution

Lock scripts are immutable once deployed (unless they use TYPE_ID). The witness format they accept is fixed for the lifetime of the deployment. This creates a tension: a protocol that needs to evolve its witness format must either deploy a new script version, use TYPE_ID to upgrade in place, or build forward-compatibility into the original design.

LS-IDL needs to have an opinion on all three of these cases. Currently it has no versioning model: the IDL describes a single, fixed interface, and the `TrailingBytes` error is returned for any leftover bytes after decoding — which makes forward compatibility impossible by default.

Safe protocol evolution requires at minimum a version marker in the IDL and a corresponding `allow_trailing` opt-in on the decoder. More ambitiously, the registry could track the version lineage of a script across TYPE_ID upgrades: IDL v1 → IDL v2, with the upgrade path recorded and the compatibility relationship explicit.

Wallet and tooling authors care about this because they need to know whether a witness they constructed for script version 1 will still be accepted by script version 2. Without a versioning model in the IDL, that question has no machine-readable answer.

---

### Generic Script Tooling

The registry and IDL together create a foundation for a class of tools that currently cannot exist because the information they need is not available in a structured form:

- **Script explorers** that let a developer browse deployed scripts, inspect their interfaces, and understand their dependencies without reading source code.
- **Compatibility checkers** that verify whether a proposed witness buffer is valid for a given script version before any transaction is built.
- **Dependency resolvers** that, given a script's code hash, automatically fetch and include its `cell_dep` entries and those of its transitive dependencies.
- **Interface diffing tools** that compare two versions of a script's IDL and report what changed, what was added, and what is backwards-compatible.
- **Documentation generators** that produce human-readable documentation for a script's witness interface from the IDL's field names, types, and descriptions.

None of these tools require understanding how a script is implemented. They operate entirely on the IDL and registry metadata. This is the point of an interface layer: it creates a stable surface that tooling can target without coupling to implementation details.

---

### Fiber Channel Involving a Non-Standard Lock

One concrete scenario that motivated much of this week's thinking: a Fiber payment channel where the funding cell uses a non-standard lock script.

In the standard Fiber case, the funding cell uses a known lock — the Fiber channel lock — and Fiber nodes have hard-coded knowledge of its witness format. This works because there is only one funding lock in the standard protocol.

In a non-standard case — a multisig lock, a timelock, a threshold signature scheme, a custom access control lock — the Fiber node has no way to construct a valid spending witness for the funding cell without per-lock knowledge. This is a hard barrier to composing Fiber with novel lock designs.

If the non-standard lock has an LS-IDL, the picture changes. The Fiber node (or wallet, or channel manager) can fetch the IDL, understand the witness fields required to spend the cell, and either construct the witness generically or delegate to the appropriate external signer. The channel can be opened, funded, and closed using a lock the Fiber implementation has never seen before.

This scenario is speculative — it depends on Fiber adopting the IDL client protocol, which is not planned today. But it illustrates the broader point: the value of LS-IDL compounds as more scripts publish interfaces, because every new IDL-enabled script increases the set of contexts where generic tooling works without custom integration.

---

### The Script Registry Plan

All of the above — generic builders, signing interfaces, off-chain validation, composability, generated bindings, wallet interoperability, protocol evolution, Fiber composability — depends on one piece of infrastructure that does not yet exist: a live, queryable registry that maps code hashes to IDL documents and script metadata.

The registry concept was formalised this week into a proposal document for submission to the Nervos DevRel team. The proposal frames the registry not as a package manager or an Explorer replacement, but as a **language-agnostic artifact layer** that sits above individual toolchains. CellScript, Rust/Cargo, and C toolchains are all producers. Explorer, wallets, SDKs, and IDEs are all consumers. LS-IDL is the interface primitive that the registry makes discoverable.

The proposal sketches a registry record structure: identity (code hash, hash type), deployment (outpoint, network), provenance (publisher, source, version), verification (executable–source binding, IDL commitment), interface (LS-IDL document, version), dependencies, and documentation. No field is required — the model is additive, so legacy scripts can be registered with only identity and deployment information, and IDL support can be added later without changing the executable.

The backwards compatibility design is deliberate. The registry explicitly distinguishes between a cryptographically bound interface (SHA-256 of the IDL matches the last 32 bytes of the code cell) and an externally attested interface (the publisher claims this IDL describes the script, but there is no on-chain commitment). Both are useful; neither is presented as equivalent to the other. Legacy scripts like `secp256k1/blake160` could receive manually authored IDLs under the attested model without requiring any change to the deployed binary.

The proposal was also used to crystallise the planned LS-IDL extensions for the DevRel audience: type registry expansion (the most immediate work), script upgrade/versioning support (TYPE_ID lineage tracking), and type script support (longer-term, not immediate — type scripts have a fundamentally different execution model that raises open questions the lock-script design does not need to answer).

The proposal is intentionally not prescribing a final architecture. It is a discussion document, and the questions it poses — canonical identity model, on-chain versus off-chain metadata, Explorer's role, dependency compatibility representation — are directed at stakeholders who have context the proposal's author does not. The next step is to collect that feedback before committing to an implementation design.

# Weekly Progress Report
## Week 16: Community Publishing, Registry Collaboration, and TypeScript Planning

Week 15 ended with both Rust crates in a stable, documented state and a clear next step: bring the project to the Nervos community and resolve the registry gap. This week that happened. The project was published on Nervos Talk, the biggest missing piece in the system found a concrete path forward through an ecosystem collaboration, and the GitHub issue documenting the project for the CKBuilder programme was drafted. The next implementation phase — the TypeScript client — is now unblocked.

---

### Published on Nervos Talk

The project was formally introduced to the community via a post titled [LS-IDL: a Lock Script Interface Description Language for CKB (derive, validate, commit)](https://talk.nervos.org/t/ls-idl-a-lock-script-interface-description-language-for-ckb-derive-validate-commit/10596).

The post explained the problem directly: CKB lock scripts have no machine-readable description of the witness format they expect. A wallet building a transaction either has out-of-band knowledge of the format or it guesses. A wrong guess only surfaces as a VM error after the transaction hits the chain. In the worst case — a group of people passing a partially signed transaction around — some witnesses are added in the wrong format and nobody finds out until on-chain rejection. The IDL system exists to move that failure to construction time.

The two-crate design was described with real code examples from `simple-lock` and `timelock-lock`. The IDL commitment scheme (`code_cell_data = risc_v_binary || sha256(idl.json)`) was explained with the reasoning: a client verifying the IDL against the on-chain commitment does not need to trust the registry that served it. Whatever registry serves the IDL, the hash check is the trust anchor.

Known limitations were stated honestly: no `[u8; 32]` support, wire encoding not recorded in the IDL JSON, no Molecule support, `required = false` not enforced at runtime, no live registry. Open design questions were directed at the community: registry design, what types are actually missing from the registry, and whether a TypeScript implementation would matter for adoption.

The post received 61 views in its first four days and was included in the CKB Ecosystem Biweekly Update.

---

### Registry Collaboration with ArthurZhang

The registry gap — the biggest gap between "works in development" and "works in practice" — now has a concrete resolution.

ArthurZhang is independently building a script registry for the CKB ecosystem. After the Nervos Talk post, he agreed to support the IDL system: his registry will store IDL documents alongside code cells, queryable by code hash, at the endpoint format that `ckb-idl-client`'s `fetch` method already expects:

```
GET {indexer_url}/idl/{code_hash_hex}  →  IdlDocument JSON
```

This is significant for a specific reason: the `fetch` method in `ckb-idl-client` was written speculatively — it was built against an interface that had no live server behind it. That is now resolved. The client already implements the right shape; the registry will implement the matching endpoint.

The commitment verification stays unchanged and is unaffected by this collaboration. The client fetches the IDL from whatever registry serves it, then independently verifies `sha256(fetched_idl) == code_cell_data[-32:]`. If that check passes, the IDL is authentic regardless of whether the registry is trusted. This design decision — made early — is what makes the registry a convenience rather than a trust assumption.

The immediate consequence for the TypeScript implementation: the client can be written against a live registry from the start, rather than being designed around local-only file loading.

---

### CKBuilder GitHub Issue (In Progress)

Work began on the project tracking issue for the CKBuilder programme, following the structure of existing issues in the repository. The issue documents current status, what is built, what is missing, and what community input is needed.

The issue has not been finalised yet. Two things are being resolved before publishing it:

1. The registry framing needs to reflect the current state: the gap is no longer "no registry exists" but "a collaborator is building the registry; the client already implements the fetch interface." The issue should describe it correctly.
2. The exact repository links need to be confirmed and consistent across both the issue and the Nervos Talk post.

---

### What Comes Next: TypeScript Client

The next implementation milestone is a TypeScript client for `ckb-idl-client`. This is the work described at the end of Week 15 as the first step toward proving the test vectors are language-independent.

The 16 canonical test vectors in `test-vectors.json` are the normative specification. The TypeScript implementation must produce identical decode results for every vector — every `valid` case must succeed with the same decoded field values, and every `error` case must fail with the matching error kind.

The implementation scope:

1. **Wire format decoder** — field-by-field decode matching `validate_witness_bytes` in the Rust client: `uint8`, `uint32`, `uint64`, fixed arrays (`secp256k1_sig`, `secp256k1_pubkey`, `schnorr_sig`), and length-prefixed `bytes`. The decoder needs to surface the same error types: `FieldTooShort`, `TrailingBytes`, `UnknownType`.

2. **IDL commitment verification** — `sha256(idl_json_bytes) === code_cell_data.slice(-32)`. This is straightforward; the crypto is standard.

3. **Test vector runner** — load `test-vectors.json`, run every case through the TypeScript decoder, assert results match the `expect` field and the `decoded` field values.

4. **Live fetch path** — because the registry collaboration is now in place, the TypeScript client can implement the fetch endpoint from the start. The Rust client's `fetch` method is the reference for what the TypeScript equivalent should look like.

The integration surface with CCC (Common Chain Connector), which is the TypeScript transaction building library in the CKB ecosystem, is an open question. The TypeScript client needs to return validated witness field descriptions that CCC transaction builders can consume. The exact API shape for that integration will be the first design decision of the implementation.

---

### Looking Back at the Registry Problem

It is worth being precise about what the registry collaboration actually resolves and what it does not.

The `verify` method in `ckb-idl-client` has always worked. It takes an IDL file, the code cell data, and checks the hash. The `validate_witness_bytes` method has always worked. The test vector suite proves both. What was missing was the network step between "a wallet encounters an unknown script on-chain" and "the wallet has an IDL document to verify and use."

That gap was not a design flaw — the fetch interface was already built correctly. It was a dependency on ecosystem infrastructure that did not exist yet. That dependency now has a concrete counterpart. The system was designed to not trust the registry; the registry is only a delivery mechanism. That design decision is what made it possible to resolve the gap through a collaboration rather than needing to build the registry from scratch as part of this project.

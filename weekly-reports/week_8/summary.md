# Weekly Progress Report
## Week 8: Planning the IDL Verifier

After the heavy implementation sprint in week 7 — which reworked `ckb-idl-derive` from a documentation-only macro into one that generates live, runtime-usable deserialization code — this week was intentionally lighter. The goal was to step back, consolidate, and plan the next concrete piece of the IDL ecosystem: the verifier.

---

### Revisiting the Verification Protocol

Week 7 introduced several breaking changes in quick succession: the `from_witness_args` method, the length-prefix wire format, the `WitnessError` enum, the `ckb-idl-types` crate split, and the move from `OUT_DIR` to `CARGO_MANIFEST_DIR`. Before adding more surface area to `ckb-idl-derive`, it was worth letting that design settle and shifting attention to the next unfinished piece of the puzzle: `ckb-idl-client`, which currently fetches IDLs but does not yet verify them against the on-chain hash commitment.
I went back to first principles on what the verifier actually needs to do, re-examining the trust model underlying the whole IDL design:

- The IDL itself is untrusted until checked against the 32-byte Blake2b hash committed in the script's code cell.
- The verifier's job is narrow and specific: fetch the code cell by `code_hash`, extract the trailing 32 bytes, hash the candidate IDL, and assert equality. Nothing more.
- This needs to work identically whether the IDL came from an indexer over the network or was bundled directly (e.g. for offline/air-gapped signing flows).

This planning was sharpened by going back to Bitcoin's BIP-174 at a lower level than before — specifically the verification discipline a PSBT signer is required to follow before producing a signature. The same discipline applies to the IDL verifier: trust nothing until the hash check passes.

---

### Extending the Design Scope: Type Scripts

While planning the verifier, I worked through whether the current IDL format generalises cleanly to type scripts, not just lock scripts. Lock scripts are witness-driven — the IDL's `witness` array is the whole story. Type scripts are mostly cell-data-driven and rarely touch the witness at all. This means a complete IDL format needs three dimensions, not one — `witness`, `cell_data`, and `args` — and the verifier needs to be agnostic to which of these it's checking, since the hash-commitment mechanism is identical regardless of which dimension the IDL describes.

---

### Where This Intersects With the Registry Idea

I also spent time reconciling this work with a parallel idea from a community collaborator — an on-chain script registry / package manager, closer in spirit to npm or cargo. The conclusion: these are not competing solutions but two halves of the same gap.

- The registry solves discovery and distribution — developer-facing.
- The IDL and its verifier solve runtime interpretation — machine-facing.

The registry is a natural home for IDL hosting and becomes the verifier's primary data source once built.

---

### Plan for Next Week

With the verification protocol now clearly scoped, the next concrete step is implementing it in `ckb-idl-client`:

- A `verify(idl_bytes, code_cell_data) -> Result<()>` method using Blake2b
- Tested against the `simple-lock` IDL generated in week 7
- Tested against synthetic tampered inputs to confirm rejection behaviour
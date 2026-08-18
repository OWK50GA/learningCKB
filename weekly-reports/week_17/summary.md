# Weekly Progress Report
## Week 17: TypeScript Implementation of ckb-idl-derive

Week 16 ended with the registry gap resolved and the TypeScript implementation marked as the next milestone. This week that implementation was built. The result is [`idl-derive-ts`](https://github.com/OWK50GA/ckb-idl-derive-ts): a TypeScript package that mirrors the Rust `ckb-idl-derive` crate, runs against the same canonical test vectors, and adds a CLI and a `defineWitness` API that the Rust side does not have.

---

### What Was Built

The package is structured around eight source modules, each with a single responsibility:

**`registry.ts`** — the type registry, parallel to `registry.rs` in the Rust crate. A `ReadonlyMap` of seven IDL type name strings to `TypeEntry` objects, each carrying a `WireKind` discriminated union:

```typescript
type WireKind =
  | { kind: "fixedScalar"; size: 1 | 4 | 8 }
  | { kind: "fixedArray"; size: number }
  | { kind: "varBytes" }
```

The same three categories as the Rust `WireKind` enum, the same seven types, the same sizes. `lookupType(fieldName, typeName)` throws `UnknownTypeError` for anything not in the map — same behaviour as the Rust compile-time error, just moved to runtime.

**`decoder.ts`** — the wire format decoder, parallel to the generated `from_witness_args` in Rust. `decodeWireBytes(fields, buf)` walks a `Uint8Array` field by field using a `DataView` cursor:

- `fixedScalar` size 1: `getUint8`
- `fixedScalar` size 4: `getUint32(offset, true)` (little-endian)
- `fixedScalar` size 8: `getBigUint64(offset, true)` — returns `bigint`, matching JavaScript's inability to represent 64-bit integers as `number` without precision loss
- `fixedArray`: `buf.slice(cursor, cursor + size)` — returns `Uint8Array`
- `varBytes`: reads 4-byte LE length prefix, then reads that many bytes

The `uint64 → bigint` decision is worth noting explicitly. The Rust side decodes `u64` as a Rust `u64`. The TypeScript equivalent is `bigint` — a JavaScript `number` cannot faithfully represent values above 2^53, and timestamps or CKB amounts routinely exceed that. The test vectors confirm this: the `uint64-roundtrip` vector uses `1700000000000`, which fits in a `number`, but the decoder always returns `bigint` for consistency.

**`errors.ts`** — four error classes, all extending `IdlDeriveError` which itself extends `Error`. The important detail is `Object.setPrototypeOf(this, new.target.prototype)` in the base constructor — without this, `instanceof` checks fail in TypeScript compiled to ES5 because class hierarchies using `extends Error` do not preserve the prototype chain correctly after transpilation. The four errors mirror the Rust equivalents: `UnknownTypeError`, `FieldTooShortError` (with `fieldName`, `expected`, `got`), `TrailingBytesError` (with `consumed`, `total`), and `MissingLockFieldError`.

**`types.ts`** — type definitions including the `WitnessValueType` mapped type, which gives TypeScript-level type inference for decoded field values:

```typescript
type WitnessValueType<T extends IdlTypeName> =
  T extends "uint8" | "uint32" ? number :
  T extends "uint64" ? bigint :
  Uint8Array
```

This means `defineWitness` returns a schema whose `fromWitnessArgs` return type is statically typed based on the field declarations — the caller knows at compile time which fields are `number`, which are `bigint`, and which are `Uint8Array`.

**`idl.ts`** — `buildIdl` constructs an `IdlDocument` from a `WitnessConfig`. `canonicalJson` serialises it with a fixed key order (`idl_version` → `name` → `witness`, and per-field: `name` → `type` → `required` → `description`). Key ordering in `JSON.stringify` depends on insertion order in JavaScript objects, so the canonical form is enforced by constructing a new object with fields added in the right sequence. `description` is only included when present — the key is absent entirely rather than set to `undefined` or `null`.

**`hash.ts`** — two SHA-256 implementations. `syncSha256` tries `require("node:crypto")` first — available in Node.js, fast, and correct. If that fails (inside `ckb-js-vm`, which has no Node.js APIs), it falls back to `sha256`, a pure-TypeScript implementation written from the FIPS 180-4 round constants. Both are verified against the standard FIPS vectors for the empty string, `"abc"`, and `"hello"`.

**`schema.ts`** — `defineWitness` is the main developer-facing API. It takes a `WitnessConfig`, validates all field types eagerly by calling `lookupType` on each one (so a bad type throws at schema definition time, not at decode time), builds the IDL, computes and caches the IDL hash, and returns a `WitnessSchema`. The `fromWitnessArgs` method on the returned schema calls `HighLevel.loadWitnessArgs` from `@ckb-js-std/core` — the CKB JavaScript VM runtime — and runs `decodeWireBytes` on the result. This method is only callable inside `ckb-js-vm`; in a Node.js test environment it is not exercised directly.

**`cli.ts`** — a `ckb-idl-derive` CLI entry point. Takes a path to a JS module that exports a `WitnessSchema` as its default export, dynamically imports it, validates the runtime shape, and writes `idl.json` to the specified output path. This is the TypeScript equivalent of the Rust macro's compile-time `idl.json` generation — a script author who writes their witness schema in TypeScript can run the CLI to produce the IDL artifact.

**`io.ts`** — `writeIdl` wraps `canonicalJson` and `node:fs/promises.writeFile`. Separated so the CLI and any programmatic caller share the same serialisation path.

---

### Test Suite

Five test files covering all modules. The most important is `vectors.test.ts`.

**`vectors.test.ts`** loads `ckb-idl-client/test-vectors.json` from the sibling Rust repository and runs all 16 cases through `decodeWireBytes`. The vector file is the normative specification — the same file the Rust `test_vectors_file` test runs against. Every `valid` case asserts decoded field values match; every `error` case asserts the correct error class is thrown and the error properties match the `error_detail` in the vector.

The remaining four test files cover `decoder.ts`, `registry.ts`, `idl.ts` (including `canonicalJson` key ordering), `hash.ts` (SHA-256 FIPS vectors), and `schema.ts` (including the `idlHash` stability and copy-on-return behaviour).

All 16 test vectors pass. The suite uses Vitest.

---

### The `defineWitness` API — What the Rust Side Does Not Have

The Rust macro derives `from_witness_args` at compile time from a struct definition. The TypeScript equivalent cannot do compile-time code generation in the same sense, so the design inverts: instead of annotating a struct and getting a method, you call `defineWitness` with a configuration object and get back a schema object with `fromWitnessArgs` as a method.

```typescript
const SimpleLock = defineWitness({
  name: "simple-lock",
  fields: {
    preimage: field("bytes", {
      description: "Preimage whose blake2b-256 hash must match the hash in script args"
    }),
  },
})

// Inside ckb-js-vm:
const witness = SimpleLock.fromWitnessArgs(0, Source.GroupInput)
// witness.preimage is Uint8Array — known at compile time
```

The `WitnessValueType` mapped type gives TypeScript-level inference: `witness.preimage` is typed as `Uint8Array`, not `unknown`. The same schema object also holds `idl` (the `IdlDocument`) and `idlHash()` (the SHA-256 commitment bytes), making it the single source of truth for both the on-chain artifact and the runtime decoder — matching the design intent of the Rust side.

---

### What Comes Next

**`ckb-idl-client` TypeScript** — the client-side library: IDL commitment verification (`sha256(idl_json) === code_cell_data.slice(-32)`) and structural witness validation from a fetched or local IDL document. This is the tooling-side counterpart to `idl-derive-ts` the same way `ckb-idl-client` (Rust) is the counterpart to `ckb-idl-derive` (Rust). The decoder is already built in `idl-derive-ts` and can be shared.

**On conformance tests and the test vectors**

The test vectors in `ckb-idl-client/test-vectors.json` are the wire format specification. They cover every type, every error kind, and both reference scripts. A new implementation that passes all 16 vectors has demonstrated conformance to the wire format.

What the vectors do not cover is everything else: the IDL commitment scheme (the SHA-256 append-to-code-cell mechanism), the `canonicalJson` key ordering (which affects the commitment hash), and the `defineWitness` / `buildIdl` IDL construction behaviour. Those are covered by the unit tests in `idl.test.ts` and `schema.test.ts`.

Additional conformance tests would only be needed if a third implementation (Python, Go) appeared and needed to prove it produces the same IDL JSON and the same commitment hash as the Rust and TypeScript implementations. Until then, the test vectors plus the unit tests are sufficient — adding more tests that re-cover the same ground in a different style adds noise without adding signal. The vectors are the right contract boundary. What matters next is getting the spec document written so that boundary is formally described rather than implied by the test file.

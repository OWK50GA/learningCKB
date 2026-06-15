# Challenges, Tradeoffs, and Design Decisions
## `ckb-idl-derive` — Week 4

---

## 1. Proc-Macro Testing Constraints

**The challenge:** Proc-macro crates cannot call their own public entry points from `#[cfg(test)]`. The proc-macro host process is separate from the test binary, so `ckb_witness(...)` simply cannot be invoked in a unit test. This is a fundamental constraint of how Rust proc-macros work.

**The tradeoff:** The test strategy had to be split across three layers:
- Unit tests on pure internal functions (`validate`, `registry`, `attr`, `codegen`, `io`) — these work fine because they are ordinary Rust functions.
- Property-based tests that call `impl_ckb_witness` — an internal function that takes `proc_macro2::TokenStream` instead of `proc_macro::TokenStream`. This is the standard workaround: keep the real logic in a testable inner function, and make the public entry point a thin wrapper.
- `trybuild` integration tests that invoke `cargo` as a subprocess to compile test files and assert on the compiler output.

**The decision:** Introduce `impl_ckb_witness(input: proc_macro2::TokenStream) -> syn::Result<proc_macro2::TokenStream>` as the real implementation, and make the public `ckb_witness` a one-liner that converts types and calls it. This is the accepted pattern in the Rust proc-macro community, but it adds a layer of indirection that is not obvious to someone reading the code for the first time.

---

## 2. `OUT_DIR` in Tests

**The challenge:** The macro writes `idl.json` to `$OUT_DIR`, which Cargo sets automatically during a normal build. But in unit and property tests, `OUT_DIR` is not set — the test binary runs outside of a Cargo build script context.

**The tradeoff:** Tests that exercise the full `impl_ckb_witness` pipeline (Property 4) need a real writable directory. The options were:
- Skip end-to-end tests entirely and only test the pure functions — safe but leaves the file-writing path untested.
- Use `tempfile::tempdir()` and manually set `OUT_DIR` via `std::env::set_var` — works, but `set_var` is `unsafe` in Rust 2024 edition because it is not thread-safe.
- Use a fixed path — fragile and would pollute the repository.

**The decision:** Use `tempfile::tempdir()` with `unsafe { std::env::set_var(...) }`, with a comment explaining the safety invariant (tests run single-threaded in this binary). This is pragmatic but requires care: if the test suite is ever run with `--test-threads > 1`, two tests could race on `OUT_DIR`. The `io` module tests also use this pattern. A cleaner long-term solution would be to inject `OUT_DIR` as a parameter rather than reading it from the environment, but that would change the public interface of `io::write_idl`.

**The trybuild `pass` test** had the same problem from a different angle: trybuild compiles each test file as a standalone crate, and that crate has no `build.rs`, so Cargo does not set `OUT_DIR` for it. The fix was to add a `build.rs` to the `ckb-idl-derive` crate itself — an empty one, whose only purpose is to make Cargo set `OUT_DIR` for the trybuild test crate that depends on it. This is a non-obvious workaround that took some investigation.

---

## 3. Semantic Ambiguity in the Type Registry

**The challenge:** Rust's type system carries structural information but not semantic intent. `[u8; 32]` could be a Blake2b hash, a lock script hash, a Schnorr public key, or an arbitrary 32-byte payload. The type alone does not tell us which.

**The tradeoff:** The registry could be:
- **Strict** — only map sizes with a single unambiguous CKB meaning, and error on everything else. This is safe but limits practical applicability.
- **Permissive** — map all `[u8; N]` to a generic `"bytesN"` type. This never errors but produces IDLs that are semantically vague.
- **Override-based** — map known sizes strictly, and allow `#[witness(type = "blake2b_hash")]` to override for unknown sizes. This is the most ergonomic but requires more attribute-parsing surface.

**The decision:** Go strict for v1. Only `[u8; 33]`, `[u8; 64]`, and `[u8; 65]` are mapped, because these are the only sizes with a single dominant CKB meaning. Any other array size produces a compile error. This means a field like `hash: [u8; 32]` currently cannot be used with the macro — a real limitation for scripts that include Blake2b hashes in their witness. The override attribute is the planned path forward but was deferred to keep the v1 surface minimal.

---

## 4. `Vec<u8>` and Length Framing

**The challenge:** `Vec<u8>` maps to `"bytes"`, which correctly describes a variable-length byte blob. But the IDL consumer cannot know how that blob is framed within the witness — length-prefixed, fixed-tail, or remainder-of-witness are all common conventions in CKB scripts.

**The tradeoff:** Including framing information in the IDL would make it more useful for automated witness construction, but it requires either a new attribute (`#[witness(layout = "remainder")]`) or a richer type system in the IDL format. Both add complexity. Omitting it keeps the v1 surface simple but means the IDL is incomplete for any witness with more than one variable-length field.

**The decision:** Accept this as a known v1 limitation and document it. The IDL is still useful for single-variable-length-field witnesses (which covers many real lock scripts), and the `layout` hint is the natural extensibility point for v2.

---

## 5. Single IDL File Per Crate

**The challenge:** The macro always writes to `$OUT_DIR/idl.json`. If a crate defines more than one struct with `#[derive(CkbWitness)]`, each invocation overwrites the previous file. The last struct wins.

**The tradeoff:** Options include:
- Per-struct filenames (`idl_MyWitness.json`) — avoids collisions but requires the consumer to know the struct name.
- Append mode — accumulate all structs into a single file — but this requires reading the existing file on each invocation, which is fragile in parallel builds.
- Enforce one-struct-per-crate — simple but restrictive.

**The decision:** The current implementation silently overwrites, which is the simplest behaviour. The implicit assumption is one witness struct per crate, which matches the intended use case (one lock script per crate). This constraint is not enforced or documented clearly, which is a gap. A future version should either enforce it explicitly (error if `idl.json` already exists in `OUT_DIR`) or switch to per-struct filenames.

---

## 6. Rust 2024 Edition and `unsafe` Environment Variable Mutation

**The challenge:** Rust 2024 edition made `std::env::set_var` and `std::env::remove_var` `unsafe` functions, because mutating the process environment is not thread-safe. The test code that sets `OUT_DIR` for property tests had to be updated to use `unsafe` blocks.

**The tradeoff:** The `unsafe` is genuinely justified here — the tests run single-threaded and no other thread reads `OUT_DIR` concurrently. But it is still `unsafe` code in a test suite, which is unusual and slightly alarming to a reader who does not know the context.

**The decision:** Use `unsafe` with an explicit safety comment explaining the invariant. The cleaner long-term fix is to refactor `io::write_idl` to accept an optional `out_dir: Option<&Path>` parameter, so tests can pass the temp directory directly without touching the environment. This was not done in v1 to keep the public interface simple.

---

## 7. Property Test Strategy: Keyword Collision

**The challenge:** The property test for generated const presence (Property 4) generates random field names using a regex strategy. The regex `[a-z][a-z0-9_]{0,10}` can produce strings that are valid Rust keywords (`do`, `in`, `fn`, `if`, etc.), which are not valid identifiers and cause `syn::parse_str::<Ident>` to panic.

**The tradeoff:** Options were:
- Tighten the regex to exclude all keywords — complex and fragile as the keyword list can change.
- Use `prop_filter` to reject keyword matches — clean but adds rejection overhead.
- Use `prop_assume!` after filtering — the approach taken.
- Catch the parse error and skip — hides real bugs.

**The decision:** Filter the generated names against a hardcoded keyword list and use `prop_assume!` to skip cases where all names were filtered out. This was discovered as a real failure during testing (proptest found `"do"` as a minimal failing case and saved it to `proptest-regressions/lib.txt`), which is exactly the kind of edge case property testing is designed to surface.

---

## 8. `syn::Field` Does Not Implement `Parse`

**The challenge:** When writing unit tests for the `attr` module, the natural approach was to construct a `syn::Field` directly using `syn::parse_str::<Field>(...)` or `parse_quote! { #[witness(...)] x: u8 }`. But `syn::Field` does not implement `syn::parse::Parse`, so neither approach compiles.

**The tradeoff:** The options were:
- Parse a full `ItemStruct` and extract the first field — verbose but correct.
- Parse a `FieldsNamed` — also works but slightly less readable.
- Use a helper macro — adds indirection.

**The decision:** Parse a full `ItemStruct` with `parse_quote! { struct S { ... } }` and extract the first field with a small helper function. This is the standard workaround in the syn ecosystem and is clear enough once you know why it is necessary. The test code is slightly more verbose than ideal but is correct and readable.

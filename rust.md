## Code style

### Rust edition and linting

- Use Rust 2024 edition.
- Format with **nightly rustfmt**: `cargo +nightly fmt --all`. `rustfmt.toml` opts into `imports_granularity = "Crate"` and `group_imports = "StdExternalCrate"`, which are nightly-only — stable rustfmt prints a warning about each and then silently skips them, so a stable-formatted file is *not* equivalent to a nightly-formatted one.
- Ensure the following checks pass at the end of each complete task (you don't need to do this for intermediate steps):
  - `cargo +nightly fmt --all -- --check`
  - `cargo clippy --all-targets --all-features -- -D warnings -W clippy::unwrap_used -W clippy::expect_used`
  - `cargo clippy --all-targets --no-default-features -- -D warnings -W clippy::unwrap_used -W clippy::expect_used`
  - `cargo test --workspace`
  - `cargo test --workspace --no-default-features`
- Use `cargo clippy` in place of `cargo build` — it typechecks and lints in one pass, so a separate build buys nothing.
- Iterate in debug. Reach for `--release` only when benchmarking or packaging.
- No `unwrap()` or `expect()` in production code; tests are fine. `unwrap_used` and `expect_used` are allow-by-default restriction lints, so the check commands above request them explicitly — plain `-D warnings` doesn't catch them.
- Never silence a lint without a concrete reason documented in a comment above it. In almost all cases the right move is to restructure the code.
- When you do suppress one, prefer `#[expect(...)]` to `#[allow(...)]`. `expect` warns once the suppression is no longer needed, so stale suppressions can't quietly accumulate as the code changes around them.

### Type system patterns

- **Builder patterns** for complex construction (e.g. `TestRunnerBuilder`).
- **Type states** encoded in generics when state transitions matter.
- **Lifetimes** used extensively to avoid cloning (e.g. `TestInstance<'a>`).
- **Newtypes** for domain types, per the general rule against bare primitives.
- **`#[non_exhaustive]`** on public types in a library crate with a stable API, so a new variant or field isn't a breaking change. Internal crates don't need it.
- **Parameter structs over long argument lists**: when a function approaches the `clippy::too_many_arguments` threshold, bundle the cohesive parameters into a struct rather than threading more positional arguments. A request struct, or a shared seam like `Engine { store, graph, clock }` that several call shapes pass along.
  - **Never** silence that lint with an `allow`. The lint firing means a struct is wanted. The one exception is a signature you don't own — an FFI shim, or a hook mirroring a foreign ABI — where the `allow` is the honest annotation.

### Error handling

- Do not use `thiserror`. Instead, manually implement `std::fmt::Display` and `std::error::Error` for a given error `struct` or `enum`. `Error::source` returns the wrapped cause where there is one, so the chain stays walkable.
- Group errors by category with an `ErrorKind` enum when appropriate.
- Provide rich error context using structured error types.
- Two-tier error model:
  - `ExpectedError`: User/external errors with semantic exit codes.
  - Internal errors: Programming errors that may panic or use internal error types.
- Every error's `Display` leads with a `<context>:` prefix naming the subsystem or operation it belongs to, then the cause: `event store: …`, `lua: block commit failed: …`, `could not open the event log at /path: …`. Messages stay lowercase, so they compose.
  - An aggregating error prefixes its own layer's context and delegates to the inner error. A chained error then reads as nested context: `turn: lua: block commit failed: event store: …`.
  - Add resource context — a path, an id — at the layer that has it. Avoid bare "failed to {x}" glue; name the operation instead.

### Async patterns

- Do not introduce async to a project without async.
- Use `tokio` for async runtime (multi-threaded).
- Use async for I/O and concurrency, keep other code synchronous.
- Use `parking_lot::Mutex` for synchronous locks (the default); its guard is non-poisoning and must never be held across an `.await`. Reserve `tokio::sync::Mutex` for the rare guard that must survive an `.await`, since most locks are acquired, used, and dropped within a synchronous span.
- Use bounded `mpsc` channels; an unbounded channel hides backpressure.

### Logging

- Use `tracing` for diagnostic and operational logging throughout, emitting at meaningful points, not noisily.
- Install the subscriber only in binaries, and send logs to stderr.
- Operator and diagnostic programs route their output through `tracing` too. Reserve `stdout`/`println!` for genuine machine-readable command output.

### Module organisation

- Use `mod.rs` files to re-export public items; nontrivial logic lives in submodules, not in `mod.rs` itself.
- Keep module boundaries strict with restricted visibility, but prefer `pub(crate)` and `pub(super)` over `pub(in <path>)`. The `pub(in …)` form scopes to a named ancestor, which is precise but reads as a smell; reach for it only when neither `pub(crate)` nor `pub(super)` expresses the intended scope.
- Use `#[cfg(unix)]` and `#[cfg(windows)]` for conditional compilation.
- **Always** import types or functions at the very top of the module, with the one exception being `cfg()`-gated functions. Never import types or modules within function contexts, other than this `cfg()`-gated exception.
- It is okay to import enum variants for pattern matching, though.
- Re-exports follow the same rule: a `pub use` belongs at the top of the module with the imports, not beside the item it re-exports. In a `mod.rs`, the `mod` declarations come first, then the `pub use` block, so the module's public surface reads as one list.
- A path used more than once in a module gets imported at the top — the specific items, not the module — rather than repeated in full at each call site.
  - A path used once may stay fully-qualified. Unless it's unwieldy, meaning more than three module segments deep, in which case import it anyway.
  - And when the module already imports a sibling from the same parent, import the new item alongside it rather than writing it inline.
- **Always** anchor intra-crate paths at `crate::`, never `super::`. Write `crate::graph::Graph`, not `super::Graph` or `super::super::Graph`. The one exception is a test module, where `use super::*;` (pulling the parent module into the `#[cfg(test)]` block) is the idiomatic form and stays.
- Prefer a single grouped `use` statement per crate root rather than several siblings under it, collapsing shared prefixes: `use axum::{extract::State, http::StatusCode, routing::get};`, not three separate lines. Group imports into three blocks separated by blank lines: `std`, external crates, then `crate`/`super`/`self`. `imports_granularity = "Crate"` and `group_imports = "StdExternalCrate"` in `rustfmt.toml` enforce both automatically under nightly rustfmt.

### Code organisation

The general code-organisation rules apply; these are the Rust specifics.

- **The file hierarchy is the architecture diagram.** A newcomer should be able to read the directory listing and infer what the project does.
  - A subsystem with a public entry point is a folder module whose `mod.rs` states the boundary, with private submodules inside. Only `lib.rs`, `main.rs`, and genuinely cross-cutting types like `errors.rs` stay as top-level single files.
  - Avoid a top-level single file where a folder grouping is natural. If several files are semantically related, or file A is only consumed by file B, merge them.
- **The file-size threshold is around 1000 lines.** Split into a folder and re-export the public items from `mod.rs`, so consumers see a stable API.
  - Splitting is cheaper than it looks: methods on one type can spread across several `impl` blocks in different files of the same module, so there's no visibility or trait plumbing to do.
  - A large inline `#[cfg(test)] mod tests` counts toward the total. Extracting it to a sibling `tests.rs` is often the whole fix.
  - Existing oversized files are grandfathered. Split one when a change touches it substantially, not in drive-by churn.
- Shared helpers for a split test module live in its `mod.rs`.

### Control flow and state machines

- Avoid rightward drift. If a function is nesting three `select!` blocks or four levels of `match`/`if let`, extract each arm into a named function that takes a context struct. The control flow at the top level should read like an outline.
- Model each state machine as an explicit `enum` with named variants, even if only one field differs between them. Favour an exhaustive `match` plus a `transition` helper over scattered `if let` chains.
- Where a subsystem transitions through phases that own different local state (e.g. `Idle`, `Starting`, `Running`, `Draining`), extract each phase body into its own function and pass a typed context struct. This is the pragmatic version of the typestate pattern for actor-style loops; it keeps invariants local without requiring full type-parameterised phases.
- Invalid transitions should be unrepresentable at the boundary where they're consumed. If `transition()` returns `Option<State>`, the caller should never `.unwrap()` it in production — either enumerate the legal inputs ahead of time, or make the caller total.

### Platform coupling

- A file that depends on platform-specific facilities says so on the first line of its module-level docstring: `//! Linux-only: reads /proc/{pid}/cmdline.` The convention is explicit enough that a port to a second platform — or a reviewer — knows exactly what the contract is.
- The outside-world seams take the form of traits — `Fs`, `ProcessSpawner`, `GpuProbe`, `Clock` — bundled into one dependency struct that production and test construct differently. A second OS implementation then falls out of the same shape the fakes already needed.
- When a second platform does land, gate the existing impl with `#[cfg(target_os = "…")]` and add the alternative under a sibling gate; the trait definition stays platform-neutral.

### Reaching through smart pointers

- To borrow the value inside a lock guard, a `Box`, or an `Arc`, prefer `.as_ref()` / `.as_mut()` over a manual double-deref: write `state.lock().as_ref()`, not `&**state.lock()`. The named form reads as "borrow the value" rather than as deref bookkeeping. The same applies to an `Arc<dyn Trait>`: `model.as_ref()`, not `&**model`.

### Serde

- Use `#[serde(deny_unknown_fields)]` on config types, and `#[serde(default)]` on new fields so they stay backwards-compatible.
- Use `serde_ignored` when deserialising config, so a typo'd or stale field is reported rather than ignored.
- Avoid `#[serde(untagged)]` when deserialising — the error messages it produces are useless. Write a custom visitor instead.
- Reserve `#[serde(flatten)]` for the case it is genuinely for: extending a shared struct with local fields, or a `toml::Table` catch-all that preserves unknown fields across a round-trip. Note that it interacts badly with `serde_ignored` and with `deny_unknown_fields`.

### Memory and performance

- Be deliberate about when you clone and when you borrow. Share immutable data behind `Arc`/`Rc` or borrow it; where the data has a natural tree or graph shape, share the nodes rather than cloning subtrees.
- Use `smol_str` for efficient small string storage.
- Use `smallvec` for collections that are usually small, to avoid heap allocations in the common case.
- Stream data (e.g. iterators) where possible rather than buffering.

### Dependency versions

- **Applications, binaries, and workspace-internal crates: pin exactly.** Cargo's default caret range lets resolution drift between `cargo update` runs; `serde = "=1.0.219"` makes a bump a reviewable change instead of an ambient one.
- **Crates published to a registry: use the narrowest range that works.** An exact pin in a published library breaks diamond-dependency unification — if two of a consumer's dependencies pin different patch versions of the same crate, they get two copies or a hard resolution failure. Publish a range and let the consumer's lockfile do the pinning.
- Manage shared versions in the workspace root's `[workspace.dependencies]` and reference them with `{ workspace = true }`.
- Check the current version before adding or bumping a crate rather than writing one from memory.
- Comment on a non-obvious dependency choice, including a feature-flag choice made for a reason.

## Testing

### Testing tools

- **test-case**: For parameterised tests.
- **proptest**: For property-based testing.
- **insta**: For snapshot testing.
- **libtest-mimic**: For custom test harnesses.
- **pretty_assertions**: For better assertion output.
- Use `cargo nextest run` in place of `cargo test` where it's available.

### Testing conventions

- Unit tests live in the same file as the code under test, in a `#[cfg(test)] mod tests` block. Integration tests live under `tests/`.
- Never `#[ignore]` a test, and never let one silently skip. A test that cannot run in an environment is gated by a `cfg` or a feature, so its absence is visible.
- Tests must be deterministic: no real subprocesses, no real filesystem, no wall-clock sleeps, no network. Route each of those through the seam traits above and substitute the fake. Gate a fake behind `#[cfg(any(test, feature = "test-fakes"))]` when integration tests need it too.
- Time is the narrow exception when everything already runs on `tokio::time`: `#[tokio::test(start_paused = true)]` plus `tokio::time::advance` gives virtual time without needing another seam.

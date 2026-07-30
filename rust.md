## Code style

### Rust edition and linting

- Use Rust 2024 edition.
- Format with **nightly rustfmt**: `cargo +nightly fmt --all`. `rustfmt.toml` opts into `imports_granularity = "Crate"` and `group_imports = "StdExternalCrate"`, which are nightly-only — stable rustfmt prints a warning about each and then silently skips them, so a stable-formatted file is *not* equivalent to a nightly-formatted one.
- Ensure the following checks pass at the end of each complete task (you do not need to do this for intermediate steps):
  - `cargo +nightly fmt --all -- --check`
  - `cargo clippy --all-targets --all-features -- -D warnings`
  - `cargo clippy --all-targets --no-default-features -- -D warnings`
  - `cargo test --workspace`
  - `cargo test --workspace --no-default-features`
- Use `cargo clippy` in place of `cargo build` — it typechecks and lints in one pass, so a separate build buys nothing.
- Iterate in debug. Reach for `--release` only when benchmarking or packaging.

### Type system patterns

- **Builder patterns** for complex construction (e.g., `TestRunnerBuilder`)
- **Type states** encoded in generics when state transitions matter
- **Lifetimes** used extensively to avoid cloning (e.g., `TestInstance<'a>`)
- **Restricted visibility**: Use `pub(crate)` and `pub(super)` liberally
- **Newtypes for domain types**: never a bare `String` or `u32` where the domain has a meaning for it.
- **Parameter structs over long argument lists**: when a function approaches the `clippy::too_many_arguments` threshold, bundle the cohesive parameters into a struct (a request struct, or a shared seam like `Engine { store, graph, clock }` that several call shapes pass along) rather than threading more positional arguments. **Never** silence the lint with `#[allow(clippy::too_many_arguments)]`; the lint firing means a struct is wanted. The exception is a signature you don't own — an FFI shim or a hook that must mirror a foreign ABI — where the `allow` is the honest annotation.
- More generally, never use `#[allow(clippy::...)]` to silence a lint without a concrete reason documented in a comment above the allow. In almost all cases the right move is to restructure the code.
- No `unwrap()` or `expect()` in production code; tests are fine.

### Error handling

- Do not use `thiserror`. Instead, manually implement `std::fmt::Error` for a given error `struct` or `enum`.
- Group errors by category with an `ErrorKind` enum when appropriate.
- Provide rich error context using structured error types.
- Two-tier error model:
  - `ExpectedError`: User/external errors with semantic exit codes.
  - Internal errors: Programming errors that may panic or use internal error types.
- Every error's `Display` leads with a `<context>:` prefix naming the subsystem or operation it belongs to (e.g. `event store: …`, `lua: block commit failed: …`, `could not open the event log at /path: …`), then the cause. Aggregating errors prefix their own layer's context and delegate the inner error, so a chained error reads as nested context (`turn: lua: block commit failed: event store: …`). Add resource context like a path at the layer that has it. Avoid bare "failed to {x}" glue.

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

### Module organization

- Use `mod.rs` files to re-export public items; nontrivial logic lives in submodules, not in `mod.rs` itself.
- Keep module boundaries strict with restricted visibility, but prefer `pub(crate)` and `pub(super)` over `pub(in <path>)`. The `pub(in …)` form scopes to a named ancestor, which is precise but reads as a smell; reach for it only when neither `pub(crate)` nor `pub(super)` expresses the intended scope.
- Use `#[cfg(unix)]` and `#[cfg(windows)]` for conditional compilation.
- **Always** import types or functions at the very top of the module, with the one exception being `cfg()`-gated functions. Never import types or modules within function contexts, other than this `cfg()`-gated exception.
- It is okay to import enum variants for pattern matching, though.
- When a path is used more than once in a module, import it at the top of the module (the specific items, not the module) rather than repeating the fully-qualified path at each call site. A path used only once may stay fully-qualified — unless it is unwieldy (more than three module segments deep), in which case import it regardless of use count. And when the module already imports a sibling from the same parent, import the new item alongside it rather than writing it inline.
- **Always** anchor intra-crate paths at `crate::`, never `super::`. Write `crate::graph::Graph`, not `super::Graph` or `super::super::Graph`. The one exception is a test module, where `use super::*;` (pulling the parent module into the `#[cfg(test)]` block) is the idiomatic form and stays.
- Prefer a single grouped `use` statement per crate root rather than several siblings under it, collapsing shared prefixes: `use axum::{extract::State, http::StatusCode, routing::get};`, not three separate lines. Group imports into three blocks separated by blank lines: `std`, external crates, then `crate`/`super`/`self`. `imports_granularity = "Crate"` and `group_imports = "StdExternalCrate"` in `rustfmt.toml` enforce both automatically under nightly rustfmt.

Within each module, organize code as follows:
1. **Public API first** - all `pub` structs, enums, and functions at the top
2. **Private implementation below** - constants, helper functions, and internal types
3. **Order by use** - private items should appear in the order they're called/used by the public API (topological order)

### Code organization

- **The file hierarchy is the architecture diagram.** A newcomer should be able to intuit what the project does by reading the directory listing. Subsystems with public entry points live as folder modules whose `mod.rs` states the boundary, with private submodules inside; only `lib.rs`, `main.rs`, and genuinely cross-cutting types (e.g. `errors.rs`) stay as top-level single files. Avoid a top-level single-file module where a natural folder grouping exists — if several files are semantically related, or file A is only consumed by file B, merge into a folder module or into the consumer.
- **Keep files under the size threshold.** Split a file into multiple files within a folder when it exceeds ~1000 lines, re-exporting the public items from `mod.rs` so consumers see a stable API. Methods on one type can be spread across several `impl` blocks in different files of the same module, so no visibility or trait plumbing is needed. A large inline `#[cfg(test)] mod tests` counts toward the total — extracting it to a sibling `tests.rs` is often the whole fix. Existing oversized files are grandfathered: split them when a change touches them substantially, not in drive-by churn.
- **Split by concern, not by size alone.** A file should be split along natural seams — distinct data types, feature groups, or functional areas — not arbitrarily at the line limit. A cohesive single-concern file that slightly exceeds the threshold is preferable to a fragmented one.
- **Organize wide folders into subfolders.** When a folder accumulates many direct children, group them by domain or role. A flat folder of 20+ files is a signal that subfolders are needed.
- **Test files follow the same threshold.** A long test file is split by concern group, with shared helpers in the test module's `mod.rs`.

### Control flow and state machines

- Avoid rightward drift. If a function is nesting three `select!` blocks or four levels of `match`/`if let`, extract each arm into a named function that takes a context struct. The control flow at the top level should read like an outline.
- Model each state machine as an explicit `enum` with named variants, even if only one field differs between them. Favour an exhaustive `match` plus a `transition` helper over scattered `if let` chains.
- Where a subsystem transitions through phases that own different local state (e.g. `Idle`, `Starting`, `Running`, `Draining`), extract each phase body into its own function and pass a typed context struct. This is the pragmatic version of the typestate pattern for actor-style loops; it keeps invariants local without requiring full type-parameterised phases.
- Invalid transitions should be unrepresentable at the boundary where they're consumed. If `transition()` returns `Option<State>`, the caller should never `.unwrap()` it in production — either enumerate the legal inputs ahead of time, or make the caller total.

### Platform coupling

- A file that depends on platform-specific facilities says so on the first line of its module-level docstring: `//! Linux-only: reads /proc/{pid}/cmdline.` The convention is explicit enough that a port to a second platform — or a reviewer — knows exactly what the contract is.
- Isolate outside-world coupling behind a small trait (`Fs`, `ProcessSpawner`, `GpuProbe`, `Clock`), bundled into one dependency struct that production and test construct differently. The trait exists so tests can substitute an in-memory or fake implementation; a second OS implementation falls out of the same shape.
- When a second platform does land, gate the existing impl with `#[cfg(target_os = "…")]` and add the alternative under a sibling gate; the trait definition stays platform-neutral.

### Reaching through smart pointers

- To borrow the value inside a lock guard, a `Box`, or an `Arc`, prefer `.as_ref()` / `.as_mut()` over a manual double-deref: write `state.lock().as_ref()`, not `&**state.lock()`. The named form reads as "borrow the value" rather than as deref bookkeeping. The same applies to an `Arc<dyn Trait>`: `model.as_ref()`, not `&**model`.

### Serde

- Use `#[serde(deny_unknown_fields)]` on config types, and `#[serde(default)]` on new fields so they stay backwards-compatible.
- Avoid `#[serde(untagged)]` when deserialising — the error messages it produces are useless. Write a custom visitor instead.
- Reserve `#[serde(flatten)]` for the case it is genuinely for: extending a shared struct with local fields, or a `toml::Table` catch-all that preserves unknown fields across a round-trip. Note that it interacts badly with `serde_ignored` and with `deny_unknown_fields`.

### Memory and performance

- Use `Arc` or borrows for shared immutable data.
- Use `smol_str` for efficient small string storage.
- Use `smallvec` for collections that are usually small, to avoid heap allocations in the common case.
- Careful attention to cloning referencing. Avoid cloning if code has a natural tree structure.
- Stream data (e.g. iterators) where possible rather than buffering.

## Testing 

### Testing tools

- **test-case**: For parameterized tests.
- **proptest**: For property-based testing.
- **insta**: For snapshot testing.
- **libtest-mimic**: For custom test harnesses.
- **pretty_assertions**: For better assertion output.
- Use `cargo nextest run` in place of `cargo test` where it's available.

### Testing conventions

- Unit tests live in the same file as the code under test, in a `#[cfg(test)] mod tests` block. Integration tests live under `tests/`.
- Never `#[ignore]` a test, and never let one silently skip. A test that cannot run in an environment is gated by a `cfg` or a feature, so its absence is visible.
- Tests must be deterministic: no real subprocesses, no real filesystem, no wall-clock sleeps, no network. Route each of those through the seam traits above and substitute the fake. Gate a fake behind `#[cfg(any(test, feature = "test-fakes"))]` when integration tests need it too.
- Time is the narrow exception when everything already runs on `tokio::time`: `#[tokio::test(start_paused = true)]` plus `tokio::time::advance` gives virtual time without another trait. Prefer that, or waiting on an explicit state predicate, over `sleep`ing to let real time pass.



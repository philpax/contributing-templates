## General conventions

### Correctness over convenience

- Model the full error space—no shortcuts or simplified error handling.
- Handle all edge cases, including race conditions, signal timing, and platform differences.
- Use the type system to encode correctness constraints.
- Prefer compile-time guarantees over runtime checks where possible.

### User experience as a primary driver

- Provide structured, helpful error messages that can be rendered with an appropriate library at a later stage.
- Make progress reporting responsive and informative.
- Maintain consistency across platforms even when underlying OS capabilities differ. Use OS-native logic rather than trying to emulate Unix on Windows (or vice versa).
- Write user-facing messages in clear, present tense: "Frobnicator now supports..." not "Frobnicator now supported..."

### Pragmatic incrementalism

- "Not overly generic"—prefer specific, composable logic over abstract frameworks.
- Evolve the design incrementally rather than attempting perfect upfront architecture.

### Dependencies

- Neither reflex is right: reaching for a dependency for something trivial and handrolling something with a long correctness tail are both mistakes, and the second is the more expensive one.
- Handroll it when it's small, self-contained, and you can see the whole problem — a builder, a wrapper, a couple of pure functions.
- Take the dependency when the problem has a tail you'd otherwise discover in production: dates and timezones, Unicode, text encodings, compression, TLS, anything cryptographic. Never handroll cryptography.
- Prefer an existing dependency (including one of its feature flags) over a new one, and a focused library over a framework you'd use 5% of.
- Where the project documents a chosen stack, use it rather than silently introducing an alternative to something already covered.
- When it's genuinely unclear, ask rather than picking silently. Either direction is cheap to change early and annoying to change late.

### Production-grade engineering

- Use type system extensively: newtypes, builder patterns, type states, lifetimes. Never use a bare string or integer where the domain has a meaning for it; a recognised closed set of values rides as an enum, not as bare strings.
- Use message passing or the actor model to avoid data races in concurrent code.
- Isolate coupling to the outside world (filesystem, clock, network, subprocesses, devices) behind a small seam — a trait, an interface, a dependency struct — with a production implementation and a test fake. Tests are deterministic because they substitute the fake, not because they clean up after the real thing.
- Prefer a **functional core, imperative shell**: keep decision logic in pure functions that take data in and return data out, and keep the I/O and concurrency primitives at the edges. The core is then testable without fakes at all.
- Test comprehensively, including edge cases, race conditions, and stress tests.
- Pay attention to what facilities already exist for testing, and aim to reuse them.
- When fixing a bug, add the failing regression test first, then make it pass.
- Don't write a test that only exercises serialisation or a derived implementation. A round-trip earns its place only when it guards a real wire: a versioned payload, a public API, or a file format.
- No personal names in fixtures, and never the author's own identity. Anonymise every test and fixture to invented placeholders; do not seed one from any real person's name, handles, or biographical details, even when real data reproduces the behaviour under test. Reproduce the *shape* of what you observed, never the actual content.
- Getting the details right is really important!

### Documentation

- Use inline comments to explain "why," not just "what".
- Don't add narrative comments in function bodies. Only add a comment if what you're doing is non-obvious or special in some way, or if something needs a deeper "why" explanation.
- Module-level documentation should explain purpose and responsibilities.
- Comments and docs describe the present state. Reserve past-tense narration for the rare case where history explains a standing "why".
- Keep the user-facing docs in sync with the code. Where a document restates something the code owns — a config default, an example config file, a generated reference, an API surface — name the code as the source of truth, and update the document in the same change. Treat a code change that lands without its doc update as incomplete, and prefer generating the document from the code (with a `--check` mode in CI) over trusting anyone to remember.
- **Always** use periods at the end of code comments.
- **Never** use title case in headings and titles. Always use sentence case.
- Always use the Oxford comma.
- Don't omit articles ("a", "an", "the"). Write "the file has a newer version" not "file has newer version".

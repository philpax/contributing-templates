## General conventions

### Correctness over convenience

- Model the full error space—no shortcuts or simplified error handling.
- Handle all edge cases, including race conditions, signal timing, and platform differences.
- Use the type system to encode correctness constraints.
- Prefer compile-time guarantees over runtime checks where possible.
- **Never silently drop content you can't represent.** When adapting data between formats, return an error for anything the target can't express. A caller may then choose to ignore it; silent data loss is a correctness bug, not a convenience.
- Validate at each layer the data crosses, not only where the bug happened to surface. One check is bypassed by the next code path, refactor, or test double; the goal is to make the bad state structurally impossible, not locally absent.

### User experience as a primary driver

- Provide structured, helpful error messages that can be rendered with an appropriate library at a later stage.
- Make progress reporting responsive and informative.
- Maintain consistency across platforms even when underlying OS capabilities differ. Use OS-native logic rather than trying to emulate Unix on Windows (or vice versa).
- Write user-facing messages in clear, present tense: "Frobnicator now supports..." not "Frobnicator now supported..."

### Pragmatic incrementalism

- "Not overly generic"—prefer specific, composable logic over abstract frameworks.
- Evolve the design incrementally rather than attempting perfect upfront architecture.
- **The rule of three**: don't abstract until you've seen the pattern three times. Three similar lines beat a premature abstraction, which is harder to remove than it was to add.
- Don't build for hypothetical future requirements.

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
- A library never reads environment variables. Configuration — keys, URLs, feature flags, paths — arrives as parameters, and only the application entry point reads the environment. Otherwise the library can't be tested without manipulating the environment, and it's silently coupled to a deployment.
- When changing anything serialised to disk or sent over a wire, walk the whole version matrix: old reader with new data, new reader with old data, and **old writer with new data**.
  - That third case is the one that gets missed. A default lets an old reader parse new data, but an old writer then drops the fields it doesn't know about on write-back — which corrupts the file rather than failing on it.
  - Bump the format version when you add the field, not when you first depend on it.
- Getting the details right is really important!

### Functional core, imperative shell

- Keep decision logic in pure functions that take data in and return data out. Keep I/O, concurrency primitives, and orchestration in a thin shell at the edges.
- The shape is gather, then process, then persist: the shell collects the inputs, the core decides, the shell writes the result. A core function that reaches out to read something mid-decision is the thing this is meant to prevent.
- Isolate coupling to the outside world — filesystem, clock, network, subprocesses, devices — behind a small seam: a trait, an interface, a dependency struct, with a production implementation and a test fake. Tests are deterministic because they substitute the fake, not because they clean up after the real thing.
- The payoff is testability. A pure core needs no fakes at all, and a thin shell has little logic left worth mocking. When a test needs elaborate setup to reach the behaviour it's checking, that's usually the code's shape talking, not the test's.
- Name files for what they hold, not for a category. No `utils`, `helpers`, `common`, or `misc` — they become dumping grounds, and nothing stops unrelated code being added to a file whose name claims nothing. `string-formatting`, `date-arithmetic`, `api-error-handling`: it's hard to put the wrong thing in those.

### Testing

- Test comprehensively, including edge cases, race conditions, and stress tests.
- Pay attention to what facilities already exist for testing, and aim to reuse them.
- When fixing a bug, add the failing regression test first, then make it pass.
- **Use real instances of what you control, and fakes for what you don't.** Your own database or filesystem is a managed dependency: talking to it is an implementation detail you can refactor freely, so test against the real thing. A third-party API, an SMTP server, a message bus is unmanaged: that conversation is observable behaviour, so put it behind a fake.
- **Don't mock what you don't own.** Wrap the third-party library in your own thin interface and substitute that, rather than mocking the library's own surface. It makes the test simpler and the design better.
- Never add a method to production code that only tests call. Cleanup and inspection helpers belong in test utilities.
- Wait for conditions, not for durations. Poll for the state you're expecting with a timeout, rather than sleeping long enough that it's *probably* ready — the latter passes locally and fails in CI. A fixed sleep is right only when the thing under test is itself about timing, like a debounce, and then the comment says why that duration.
- Clean up long-lived resources: containers, VMs, processes, cloud objects. Don't bother scrubbing database rows and log entries — perfect data cleanup is a fool's errand that makes multi-step integration tests nearly impossible. A test that needs pristine state should mint unique identifiers instead of depending on an empty table.
- Don't write a test that only exercises serialisation or a derived implementation. A round-trip earns its place only when it guards a real wire: a versioned payload, a public API, or a file format.
- No personal names in fixtures, and never the author's own identity. Anonymise every test and fixture to invented placeholders; do not seed one from any real person's name, handles, or biographical details, even when real data reproduces the behaviour under test. Reproduce the *shape* of what you observed, never the actual content.

Where a function has a property worth stating, prefer a property-based test over a handful of examples. Reach for the strongest property that applies — roughly, in increasing order of strength: doesn't crash, preserves the type, holds an invariant, is idempotent (`f(f(x)) == f(x)`), round-trips (`decode(encode(x)) == x`). An oracle property, where a new implementation must agree with the old one, is the tool for a rewrite.

Two ways a property test can look fine and test nothing. It can be tautological, comparing an expression against itself. Or it can restate the function's own logic in the assertion, in which case a bug in your reasoning appears in both halves and cancels out. Include the degenerate cases — empty, single element — explicitly rather than trusting the generator to find them.

### Documentation

- Use inline comments to explain "why," not just "what".
- Don't add narrative comments in function bodies. Only add a comment if what you're doing is non-obvious or special in some way, or if something needs a deeper "why" explanation.
- Module-level documentation should explain purpose and responsibilities.
- Comments and docs describe the present state. Reserve past-tense narration for the rare case where history explains a standing "why".
- Keep the user-facing docs in sync with the code. Where a document restates something the code owns — a config default, an example config file, a generated reference, an API surface — name the code as the source of truth and update the document in the same change. A code change that lands without its doc update isn't finished.
- Better still, generate the document from the code and check it in CI. Don't rely on anyone remembering.
- **Always** use periods at the end of code comments.
- **Never** use title case in headings and titles. Always use sentence case.
- Always use the Oxford comma.
- Don't omit articles ("a", "an", "the"). Write "the file has a newer version" not "file has newer version".

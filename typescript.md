## TypeScript code style

The same correctness-first mindset applies here as anywhere else: TypeScript's type system is strong enough to encode most of the same invariants, and it should be pushed to do so. "Just cast it" is not an acceptable answer.

### Tooling and workflow

- These checks must pass at the end of each complete task, and CI enforces them:
  - `npm run typecheck` — `tsc -b` with no errors.
  - `npm run lint` — ESLint.
  - `npm run format:check` — Prettier. `npm run format` writes the fixes.
  - `npm run build` — the production build, where the project produces one.
- Run them frequently during development, not just at the end of a task. A clean lint is cheap to maintain and expensive to recover.

### Compiler settings

- Keep the project on the strictest practical settings: `strict`, `noUnusedLocals`, `noUnusedParameters`, `noFallthroughCasesInSwitch`, `erasableSyntaxOnly`, and `verbatimModuleSyntax`. Do not relax these.
- Prefer `import type { … }` for type-only imports, as `verbatimModuleSyntax` requires.
- Do not disable a rule or a flag to make a specific piece of code compile. Fix the code instead.

### Type system patterns

Treat these as the TypeScript analogues of the Rust patterns. The goal is the same: make illegal states unrepresentable.

- **Discriminated unions** for modelling state machines and result types — the equivalent of Rust enums. Always include a `kind` (or similar) tag and narrow on it.
- **Exhaustiveness checking** via a `never`-typed default branch in switches and `if`/`else` chains, so adding a new variant becomes a compile error everywhere it is handled.
- **Branded (nominal) types** for values that share a representation but not a meaning (e.g. `UserId` vs. `ProjectId` both being `string`). This is the parallel of Rust newtypes.
- **`readonly`** on arrays, tuples, and object properties by default, and `Readonly<T>` on reference-type parameters. Reach for mutability only when it is genuinely needed.
- **`as const`** for literal data that should be inferred as narrowly as possible, and **`satisfies`** to check a value against a type without widening its inferred type.
- **Template literal types** and mapped/conditional types to encode constraints at the type level where it pays off.
- **Prefer `unknown` over `any`**. If you reach for `any`, stop and reconsider; if it is truly unavoidable, isolate it behind a narrow boundary and document why.
- **Avoid type assertions** (`as SomeType`) and non-null assertions (`!`). Use type guards, discriminated unions, or restructured code instead. A type assertion is a claim the compiler cannot verify, so it is a liability.
- **Validate at boundaries**. Data from the network, `localStorage`, URL parameters, or any other untyped source must be parsed and validated before being treated as typed. Do not trust a `JSON.parse` result.
- Use `type` for object shapes rather than `interface`. Reserve `interface` for the cases that need declaration merging or a framework convention that expects it.
- **String literal unions over `enum`s.** A union of literals narrows properly, needs no import at the use site, and doesn't generate a runtime object. `type Status = "pending" | "active" | "failed"`.
- **Name booleans with a prefix** — `is`, `has`, `can`, `should`, `will` — or as a plain adjective on a data field (`active`, `visible`). Avoid negative names: `isEnabled` beats `isDisabled`, because `!isDisabled` is one negation too many.
- Be consistent about which of `null` and `undefined` means "absent", and convert at the boundary rather than letting both flow through the same field. `undefined` is the ecosystem default (`field?: T`, optional props, default parameters); prefer it unless a wire format or a backend makes `null` the natural choice.

### Errors

- Model the full error space. Prefer a discriminated union result type (`{ kind: "ok"; value: T } | { kind: "err"; error: E }`) or similar over throwing for expected failure modes.
- Exceptions are for genuinely exceptional, programmer-error situations.
- User-facing error messages follow the same rules as the rest of the project: present tense, sentence case, with periods.

### Module organization

- Import types and values at the top of the file. No inline `require`/`import()` inside function bodies except for genuinely dynamic imports (code-splitting).
- Use named exports. Reserve default exports for cases where a framework or tool requires them (e.g. route modules, some bundler entry points).
- Use function declarations for top-level functions and arrow functions for inline callbacks. Annotate return types explicitly on exported functions, including `Promise<T>`.
- Keep file layout consistent with the rest of the project: public API first, private helpers below, ordered by use.
- Keep files under ~400 lines, split by concern rather than arbitrarily at the limit.
- Group a `lib/` (or equivalent) folder by concern — `api/`, `model/`, `format/`, `nav/` — rather than letting it go flat. A flat folder of 20+ files is a signal that subfolders are needed.
- Use lowerCamelCase filenames for modules that export values and functions.

### Generated bindings across a language boundary

Where the frontend talks to a backend in another language, that backend owns the wire contract. The TypeScript side of it is generated — never hand-edited, and generally not committed.

Wire the generation into the backend's own build so it can't go stale, and have CI run that build before the frontend checks. A change to a wire type is then validated in the same commit that makes it. Lint and format skip the generated directories.

Regenerating makes new fields *typed*; it doesn't make them *visible*. When a backend change adds or restructures state that the frontend displays, update the display code in the same change — a field that arrives typed but unrendered is a gap, where the UI shows stale behaviour while the backend has moved on.

### Testing tools

- **Vitest**: Unit and component tests.
- **Playwright**: For end-to-end flows against a running backend, if and when one is justified. Do not reach for this for what a unit test can cover.

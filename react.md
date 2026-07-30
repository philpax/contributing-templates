## React

- Write function components and hooks. No class components.
- Follow the rules of hooks strictly, and keep `eslint-plugin-react-hooks` warnings at zero. Add the plugin's rules to the ESLint config and include them in the `npm run lint` check.
- Where the React Compiler is enabled, manual memoization (`useMemo`, `useCallback`, `React.memo`) is unnecessary and should not be added preemptively — prefer plain derivation. Reach for it only when the compiler demonstrably cannot handle a case, and say so in a comment. The compiler relies on the code staying within the Rules of React, so the hooks lint is load-bearing, not advisory.
- Keep components small and focused. Lift state only as far as it needs to go.
- Type component props explicitly. Do not rely on inference for the public shape of a component. Props are the one place `interface` is preferred over `type`.
- Prefer composition over configuration — a few focused components beat one component with a dozen boolean props.

### File and folder layout

- One main component per file, with its co-located sub-components. Non-component utilities (hooks, constants, pure functions) go in separate files so hot-reload boundaries stay clean.
- Use PascalCase for `.tsx` component files and lowerCamelCase for `.ts` utility files. Where a component and its utilities would collide, give the utilities a `Utilities` suffix (e.g. `channelUtilities.ts`).
- A `components/` folder is for shared components only — those used by two or more views. A component used by exactly one view lives in that view's folder; a component used by exactly one application shell lives with that shell.

### Testing

- **React Testing Library** for component tests — query by user-visible semantics, not implementation details.

## Styling: Tailwind first, CSS last

- Styling is done with Tailwind utility classes in the markup.
- **Do not write custom CSS unless it truly, genuinely cannot be expressed in Tailwind.** This is a hard rule, not a soft preference. "It would be slightly cleaner in CSS" is not sufficient justification; neither is "I'm more comfortable with CSS". If you think you need custom CSS, first check whether an arbitrary value (`[…]`), a variant, a theme extension, or a small component abstraction solves it.
- When custom CSS is genuinely required (e.g. a keyframe animation, or a selector Tailwind cannot express), keep it minimal, colocated, and leave a comment explaining why Tailwind was not sufficient.
- Define the design tokens — colours, spacing, type scale — in the Tailwind theme, and reach for those tokens rather than ad-hoc values, so the visual system stays coherent and design changes stay centralised. Name them for their role in the design, not for the colour they currently are.

### Linting

Class order, duplicate and conflicting utilities, typo'd class names, and shorthand collapsing are all machine-checkable — so let the linter own them rather than spending review on them. Use [`eslint-plugin-better-tailwindcss`](https://github.com/schoero/eslint-plugin-better-tailwindcss) with its `recommended` config, and include it in the `npm run lint` check. Two things to get right when setting it up:

- It reports stylistic rules as warnings and correctness rules as errors by default. Treat both as errors, so lint stays a binary signal.
- Point it at the theme, or every custom token you define will be reported as an unknown class: `entryPoint` for a v4 CSS-based config, `tailwindConfig` for v3.

```js
settings: { "better-tailwindcss": { entryPoint: "src/app.css" } }
```

Compose conditional classes with `clsx` (plus `tailwind-merge` where later classes must override earlier ones), or `cva` for a component with variants. Never build a class name by interpolating fragments into a string — it defeats both the linter and Tailwind's own class extraction.

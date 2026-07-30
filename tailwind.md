## Styling: Tailwind first, CSS last

- Styling is done with Tailwind utility classes in the markup.
- **Do not write custom CSS unless it truly, genuinely cannot be expressed in Tailwind.** This is a hard rule, not a soft preference. "It would be slightly cleaner in CSS" is not sufficient justification; neither is "I'm more comfortable with CSS". If you think you need custom CSS, first check whether an arbitrary value (`[…]`), a variant, a theme extension, or a small component abstraction solves it.
- When custom CSS is genuinely required (e.g. a keyframe animation, or a selector Tailwind cannot express), keep it minimal, colocated, and leave a comment explaining why Tailwind was not sufficient.
- Define the design tokens — colours, spacing, type scale — in the Tailwind theme, and reach for those tokens rather than ad-hoc values, so the visual system stays coherent and design changes stay centralised. Name them for their role in the design, not for the colour they currently are.

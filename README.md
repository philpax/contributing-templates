# contributing-templates

Inspiration for `CONTRIBUTING.md` files for my own projects, which should serve as guidelines for humans and agents alike.

The intended use is to initialise the `CONTRIBUTING.md` for a given project by combining `general.md` with whichever of the other files apply:

- `general.md` — language-agnostic conventions. Always.
- `rust.md`, `typescript.md` — per language.
- `react.md`, `tailwind.md` — per library, on top of `typescript.md`. Only include one if the project actually uses it; a plain TypeScript project takes neither.

So a Rust daemon with a React + Tailwind frontend takes all five, and a Rust CLI takes two.

These are a starting point, not a standard. Every project that uses them ends up amending its copy with project-specific detail, and diverging where the project's needs differ; the templates aren't a source of truth to sync back to. The reverse direction is worth doing periodically, though: when several projects independently arrive at the same rule, it belongs here.

As per [Coding Agents Are Easy, Actually](https://karashiiro.leaflet.pub/3mbfapvdvss2b), this explicitly targets `CONTRIBUTING.md`, and not `AGENTS.md`; humans need the same kind of documentation!

Some of the conventions here are drawn from [ed3d's house-style skills](https://github.com/ed3dai/ed3d-plugins), which cover similar ground as a set of Claude Code skills. Where they and these templates disagree — `thiserror`, import granularity, `null` versus `undefined` — I've kept my own house style, but the reasoning in theirs is worth reading regardless, and several rules here are better for having been through it.

At the time of writing, these files are a cut-down and edited version of [cargo-nextest's `AGENTS.md`](https://github.com/nextest-rs/nextest/blob/main/AGENTS.md), which was primarily authored by [@sunshowers](https://github.com/sunshowers). 
I really admire the thoughtful work they've done in putting together `nextest`'s `AGENTS.md`, and I'm equally thankful that it's something that the community as a whole can benefit from! Accordingly, this repository is also licensed MIT and Apache2.

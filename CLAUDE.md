# CLAUDE.md — Cellular Automata Playground

This file is the standing brief for Claude Code. Read it fully at the start of every session, then read `docs/ROADMAP.md` to find the current phase.

## What we are building

A website where people experiment with cellular automata (CA). The core experience is a single "playground" page: a grid the user can edit, run, pause, speed up, resize, and configure (rule, boundary condition, later cell shape), plus import/export of patterns. Over time the site grows into several pages (2D, 1D, 3D, unbounded/infinite, etc.).

It is a non-commercial side project that will grow in small steps and feed a technical blog (published elsewhere — the blog is **not** part of this site). Correctness, clear architecture, and measurable performance matter more than feature count.

The owner (Ahmad) is a full-stack developer with a math background. Development is AI-first: you write most of the code; he reviews. Explain non-obvious decisions briefly, in plain terms.

## Tech stack (decided)

| Concern | Choice | Notes |
|---|---|---|
| Language | TypeScript (strict mode) | No `any` without a comment explaining why. |
| Build/dev | Vite | |
| UI | React + React Router | React is for controls, layout and routing **only**. |
| UI state | Zustand | Settings such as rule, speed, dimensions. Never per-cell data. |
| Styling | Plain CSS / CSS Modules | No CSS framework unless we agree to add one. |
| 2D rendering | Canvas 2D (WebGL later if profiling demands it) | |
| 3D rendering | Three.js (from npm), added in the 3D phase | |
| Tests | Vitest | Engine code must be covered by tests. |
| Lint/format | ESLint + Prettier | |
| Package manager | npm | |
| WASM (later) | Rust + wasm-pack | Only when profiling shows the TS engine is the bottleneck. |
| Hosting | Cloudflare Workers with Static Assets (free tier), SPA mode | Deployed from GitHub. |
| Analytics | Cloudflare Web Analytics (cookie-less, free) | Production builds only. |

Do not add a dependency without stating in your plan what it is for and why the platform or existing code can't do it.

## Architecture rules (the important part)

The codebase has four layers. Dependencies point **downward only**.

```
src/
  engine/      Pure TypeScript. No DOM, no React, no browser APIs. Runs in Node, a Web Worker, or gets swapped for WASM.
  formats/     Pure TypeScript parsers/serializers (RLE, CSV, URL encoding). May depend on engine types.
  render/      Draws engine state to a canvas (2D now, Three.js later). Knows nothing about React.
  ui/          React components, pages, routing, stores. Talks to the engine through a controller, never touches cells in render loops.
  patterns/    Built-in pattern library (RLE text + metadata).
```

### Engine design constraints

These exist so that later phases (multi-state, hex cells, custom rules, Web Workers, WASM, unbounded grids, 3D) do not force a rewrite. Respect them even when a phase does not need them yet.

1. **Cell storage:** flat typed arrays (`Uint8Array`), row-major, with **double buffering** (read from current, write to next, swap). One byte per cell so up to 256 states are possible from day one, even though phase 1 only uses 0/1.
2. **States are integers.** 0 is always "dead/empty". Never encode state as booleans anywhere in the engine API.
3. **Separate concepts, separate modules.** Each of these is its own abstraction with its own tests:
   - **Grid** — dimensions and the cell buffers.
   - **Topology** — the cell shape and the neighbor list each cell has (square-Moore first; von Neumann, hexagonal and others later). A topology describes neighbors as offsets; the rule never hard-codes "8 neighbors".
   - **Boundary condition** — what lies beyond the edge: `dead` (fixed 0) and `wrap` (torus) in phase 1; `reflect` and possibly `alive`/fixed later. Boundary logic lives in **one** place. The preferred technique is a **halo (ghost-cell) border**: allocate the buffer with a margin, fill the margin from the boundary rule before each step, and keep the inner loop branch-free.
   - **Rule** — maps (current state, neighborhood) → next state. Rules are parsed from strings by a rule parser registry, so new rule families plug in without touching the stepper.
4. **Rules compile to lookup tables where possible.** An outer-totalistic rule becomes a table indexed by (state, live-neighbor count). Later, a fully general 2-state Moore rule becomes a 512-entry table indexed by the 9-bit neighborhood. The stepper's hot loop does table lookups, not string or branch logic.
5. **The engine owns edits.** Toggling a cell, setting a cell, randomizing, clearing and **stamping a pattern at (x, y)** (with optional rotation/flip) are engine operations, so the UI, the importer, and the future pattern sidebar all share one code path.
6. **Simulation state is serializable.** Grid size, topology, boundary, rule string, generation count and cells can be fully captured and restored. This underpins import/export, URL sharing, and moving the engine into a Worker.
7. **One engine interface.** Define a TypeScript interface for the engine (create/resize, step(n), edit operations, read cells, serialize). The TS implementation is the first; a Worker-backed proxy and a WASM implementation will implement the same interface later. The UI depends only on the interface.

### Rendering notes

- For large grids, write cells into an `ImageData` buffer at one pixel per cell on an offscreen canvas, then `drawImage` it scaled up with image smoothing turned off. Draw grid lines only when zoomed in far enough for them to be visible.
- The run loop uses `requestAnimationFrame` with a time accumulator to honor "steps per second". At high speeds, run several steps per frame and render once.
- Mouse/touch coordinates map to cells through a single function owned by the renderer (this will matter for hexagonal cells).

### UI notes

- Desktop-first. The 2D playground should be usable on a phone (layout doesn't break; tapping toggles a cell), but we make no promises about advanced mobile interactions. The 3D pages are desktop-only, with a polite notice on small screens.
- Editing with the mouse is allowed while paused. Pattern placement is only allowed while paused.

## File formats

- **RLE** is the primary pattern format (the Life community standard used by Golly and LifeWiki). Support the header `x = <w>, y = <h>, rule = <rule>`, `#` comment lines, and multi-state RLE (`.`, `A`–`X`, `pA`…) for later phases. Golly's bounded-grid suffix on the rule maps to our boundary conditions: `:T<w>,<h>` = wrap (torus), `:P<w>,<h>` = dead edges. Anything RLE cannot express (e.g. reflective boundaries) goes in a `#C` comment line with a clear app-specific prefix, and import must tolerate those lines being missing.
- **CSV** of state integers (one row per line) as a simple raw export. Dimensions come from the row and column counts.
- **URL sharing**: the full simulation state, compressed and encoded into the URL hash, so a blog post can link to an exact configuration.

## Working agreement

1. **One phase at a time.** Work only on the phase or task Ahmad asks for. If you spot something for a later phase, note it in `docs/ROADMAP.md` under that phase instead of building it.
2. **Plan before coding** for any task larger than a small fix: list the files you'll touch, the interfaces you'll add or change, and how you'll test it. Then implement.
3. **Engine work is test-first.** Use known CA behavior as fixtures: a blinker has period 2; a block is still; a glider moves (1, 1) every 4 generations; a Gosper glider gun has period 30; wrap-around keeps population counts that dead edges don't.
4. **Before each commit**, run `npm run typecheck`, `npm run lint` and `npm test`. All must pass.
5. **Small commits** with conventional-commit style messages (`feat(engine): …`, `fix(render): …`, `test: …`, `docs: …`).
6. **Keep docs current.**
   - Tick items off in `docs/ROADMAP.md` as they are completed.
   - Record significant decisions in `docs/DECISIONS.md` (date, decision, alternatives considered, why). One short entry each.
   - Append notable findings to `docs/DEVLOG.md`: benchmarks with numbers, bugs that taught something, surprising CA behavior. This file is raw material for blog posts, so include concrete numbers and before/after comparisons.
7. **Don't deviate from this architecture silently.** If a constraint above seems wrong for the task at hand, stop and explain the trade-off before changing course.
8. Don't commit secrets. The analytics token is read from an environment variable at build time.

## Commands

These are set up in phase 0. Keep this list accurate if scripts change.

- `npm run dev` — local dev server
- `npm run build` — production build to `dist/`
- `npm run preview` — serve the production build locally
- `npm test` — run Vitest once; `npm run test:watch` for watch mode
- `npm run typecheck` — `tsc --noEmit`
- `npm run lint` / `npm run format`
- `npm run deploy` — build and deploy to Cloudflare with Wrangler (if not deploying automatically from GitHub)

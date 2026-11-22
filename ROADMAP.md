# Roadmap — Cellular Automata Playground

How to use this file:
- Work through phases in order. Within a phase, tasks can be done in the listed order.
- A phase is finished only when every **Done when** check passes.
- Each phase ends with a **Blog angle**: what is worth writing about once it ships. Log the evidence (numbers, screenshots, surprises) in `docs/DEVLOG.md` as you go.
- Phases 0–4 are the first release. Everything after that is the growth path, and its order can be changed.

Suggested prompt to start a phase in Claude Code:
> Read CLAUDE.md and docs/ROADMAP.md. We're starting Phase N. Propose a plan (files, interfaces, tests) before writing code.

---

## Phase 0 — Project setup and deployment pipeline

**Goal:** an empty but real site, deployed, with tooling, CI and analytics working. Deploying before there are features means deployment never becomes a last-minute problem.

- [ ] Scaffold Vite + React + TypeScript (strict). Add React Router with placeholder routes: `/` (landing), `/2d` (playground), `/about`.
- [ ] Add ESLint, Prettier, Vitest, plus the npm scripts listed in CLAUDE.md.
- [ ] Create the folder layout from CLAUDE.md (`engine/`, `formats/`, `render/`, `ui/`, `patterns/`), each with a short README stating its rules. Add an ESLint rule (e.g. `no-restricted-imports`) that stops `engine/` and `formats/` from importing React, DOM-dependent code, or `ui/`.
- [ ] Create `docs/DECISIONS.md` and `docs/DEVLOG.md`, with a first DECISIONS entry recording the stack choice.
- [ ] GitHub Actions CI: on push and PR, run typecheck, lint, test, build.
- [ ] Cloudflare deployment using **Workers with Static Assets** (`wrangler.jsonc`, `assets.directory = "./dist"`, `not_found_handling = "single-page-application"` so deep links like `/2d` work). Either connect the GitHub repo through Cloudflare's Git integration (Workers Builds) or add a `deploy` step to CI using a Cloudflare API token stored as a GitHub secret. Choose one, document it in DECISIONS.md, and write the manual setup steps Ahmad has to do in the dashboard in `docs/DEPLOY.md`.
- [ ] Cloudflare Web Analytics: inject the JS beacon only in production builds, with the token from a build-time env var (`VITE_CF_BEACON_TOKEN`). Document how to get the token in `docs/DEPLOY.md`.
- [ ] Basic `README.md`: what the project is, how to run it, and links to the docs.

**Done when:** the site is live on a `*.workers.dev` URL; refreshing on `/2d` doesn't 404; CI is green; a visit shows up in Cloudflare Web Analytics; the ESLint layer rule fails if you deliberately import React into `engine/`.

**Blog angle:** "Free hosting and privacy-friendly analytics for a side project in 2026."

---

## Phase 1 — Engine core (2 states, built to grow)

**Goal:** a pure, tested simulation engine for Life-like rules on a bounded square grid. No UI yet.

- [ ] Define the engine interface (create/resize, `step(n)`, edit operations, read access, serialize/deserialize, generation counter, population count).
- [ ] Grid: flat `Uint8Array` double buffer with a halo border.
- [ ] Topology abstraction, implemented for **square / Moore** (8 neighbors). Write it so von Neumann and hexagonal can be added later as data (offset lists), not as new stepping code.
- [ ] Boundary conditions: `dead` and `wrap`, applied by filling the halo before each step. Leave a clear spot for `reflect` later.
- [ ] Rule parser registry with a first parser for **B/S notation** (`B3/S23`; also accept `23/3` S/B legacy notation). Validate input and return helpful error messages (shown in the UI in phase 2). Compile to a (state, count) lookup table.
- [ ] Edit operations: set/toggle cell, clear, randomize(density, seed), stamp(pattern, x, y, rotation, flip). Seedable RNG so tests and shared links are reproducible.
- [ ] Serialization of full state (dimensions, topology id, boundary, rule string, generation, cells).
- [ ] Tests: still lifes, oscillators (blinker period 2, pulsar period 3), glider displacement, wrap-around vs dead-edge behavior at corners, rule parsing (valid/invalid cases), a few non-Life rules (HighLife `B36/S23`, Seeds `B2/S`), stamp with rotation/flip.
- [ ] A micro-benchmark script (Node) that times N generations on 256², 512² and 1024² random grids. Record the baseline in DEVLOG.md.

**Done when:** all tests pass; the engine has no imports from DOM/React; the benchmark baseline is recorded.

**Blog angle:** "Designing a CA engine that won't need a rewrite: halos, lookup tables and typed arrays."

---

## Phase 2 — 2D playground page

**Goal:** the core user experience from the original spec.

- [ ] Canvas renderer (ImageData + scaled `drawImage`), plus grid lines when zoomed in. Uses theme colors for alive/dead cells.
- [ ] Engine controller connecting engine ↔ renderer ↔ UI store, with a rAF loop and a steps-per-second accumulator.
- [ ] Controls:
  - [ ] Start/stop and single step
  - [ ] Speed dropdown (e.g. 1, 5, 10, 30, 60 steps/s, Max)
  - [ ] Dimensions dropdown (e.g. 32², 64², 128², 256², 512², plus custom W×H). Resizing keeps the existing pattern centered.
  - [ ] Boundary condition dropdown (dead / wrap)
  - [ ] Rule input: a text field with a presets dropdown (Life, HighLife, Seeds, Day & Night `B3678/S34678`, Maze `B3/S12345`, Replicator `B1357/S1357`) and inline validation errors
  - [ ] Randomize (with density slider) and Clear
  - [ ] Live readout: generation and population
- [ ] Mouse: click to toggle a cell; click-and-drag to paint (the drag sets cells to the state of the first cell's opposite, so you can paint or erase). Editing only while paused.
- [ ] Touch: tap toggles a cell. Layout stacks the controls under the grid on narrow screens.
- [ ] Keyboard shortcuts: Space (play/pause), N/→ (step), R (randomize), C (clear). List them in a small help popover.
- [ ] Settings persist in `localStorage` (wrapped in try/catch).

**Done when:** everything in the original spec items 1–6 works on desktop Chrome/Firefox/Safari; a 256² grid runs smoothly at 60 steps/s; the page loads and toggling works on a phone.

**Blog angle:** "Rendering a million cells at 60 fps in Canvas 2D."

---

## Phase 3 — Import, export and sharing

**Goal:** patterns move in and out of the site easily, including links into blog posts.

- [ ] RLE parser and writer in `formats/` (header, comments, run lengths, line wrapping at 70 chars, Golly `:T`/`:P` bounded-grid suffixes, app-specific `#C` lines for anything else). Parse multi-state RLE now even though only 2 states render, so phase 6 has it ready.
- [ ] Round-trip tests: parse → write → parse gives identical state. Include real-world RLE samples with comments and odd whitespace.
- [ ] CSV export and import (integers, rows = lines).
- [ ] UI: "Download" menu (RLE, CSV, PNG snapshot); "Import" via file picker, drag-and-drop onto the grid, and pasting RLE text into a dialog. Importing a pattern larger than the grid asks whether to resize.
- [ ] Shareable URL: the serialized state compressed (e.g. with the browser's built-in `CompressionStream`, or a tiny well-known library if needed) and base64url-encoded in the hash. "Copy link" button. Opening the link restores the exact configuration, paused.
- [ ] Optional `?embed=1` mode: a minimal UI (grid + play/pause) for embedding in an iframe on the personal blog.

**Done when:** a Gosper glider gun pasted as RLE from LifeWiki loads and runs correctly; exported RLE re-imports identically; Golly can open our exported RLE; a copied link restores state in a fresh browser.

**Blog angle:** "The RLE format: how the Life community has shared patterns for decades."

---

## Phase 4 — Pattern library and placement

**Goal:** the sidebar from the original spec (item 7).

- [ ] Pattern library in `patterns/`: RLE text plus metadata (name, category such as still life / oscillator / spaceship / gun / methuselah, period, rule it belongs to, source/attribution). Start with roughly 20 classics: block, beehive, loaf, blinker, toad, beacon, pulsar, pentadecathlon, glider, LWSS/MWSS/HWSS, R-pentomino, acorn, diehard, Gosper glider gun, Simkin glider gun.
- [ ] A test for every library pattern verifying its claimed behavior (still lifes don't change; oscillators return after their period; spaceships reappear shifted after their period). This catches mistyped RLE.
- [ ] Sidebar UI: categories, search, small preview thumbnail per pattern. Patterns for a different rule than the current one are labeled as such.
- [ ] Placement mode (paused only): selecting a pattern shows a translucent ghost under the cursor; R rotates, F flips, click stamps it, Esc cancels. On touch, tap places it at the tapped cell.
- [ ] Record pattern sources and licenses in `patterns/README.md`.

**Done when:** every library pattern passes its behavior test; placing, rotating and flipping work; placement is disabled while running.

**Blog angle:** "Testing a pattern library with the patterns' own physics."

**🎉 First public release after this phase.**

---

## Phase 5 — Performance pass I

**Goal:** bigger grids, smooth UI, and measurements to justify later work.

- [ ] Profile at 1024² and 2048². Record results in DEVLOG.md.
- [ ] Move the engine into a **Web Worker** behind the same engine interface (a proxy implementation). Transfer frames with transferable buffers or a `SharedArrayBuffer` (the latter needs COOP/COEP headers, so document what that requires on Cloudflare before choosing it).
- [ ] Optimize the TS stepper based on the profile (e.g. running neighbor sums, skipping inactive regions, bit-packing as an experiment). Keep every optimization behind the tests.
- [ ] Zoom (wheel/pinch) and pan (drag with a modifier or the middle mouse button) for large grids.
- [ ] Optional `/bench` page that runs the benchmark in the browser and shows generations per second.

**Done when:** a 1024² grid runs without UI jank; before/after numbers are in DEVLOG.md.

**Blog angle:** "Profiling a cellular automaton: what actually made it faster."

---

## Phase 6 — Rule system expansion (multi-state and custom rules)

**Goal:** go beyond B/S, up to letting the user define essentially any rule. Ordered from simplest to most general. Each step is a new parser in the registry, not a stepper rewrite.

- [ ] **Neighborhood suffixes** on B/S rules: `V` for von Neumann (needs the von Neumann topology; check whether phase 7 should come first for this item).
- [ ] **Generations rules** (multi-state, `B2/S/C3` style, e.g. Brian's Brain): renderer palettes for N states; multi-state RLE fully wired.
- [ ] **Isotropic non-totalistic rules** (Hensel notation, e.g. `B2-a/S12`): the rule depends on the *shape* of the live neighbors, not just how many there are. Compile to a 512-entry table.
- [ ] **MAP rules** (the Golly/LifeViewer format: a base64-encoded 512-bit table). This is fully general for 2-state Moore rules. Every rule above can also be exported as a MAP string.
- [ ] **Visual rule editor**: shows the neighborhood configurations (grouped by symmetry so there are 102 classes instead of 512 cases) and lets the user click which ones produce a live cell. It outputs a MAP or Hensel string, so rules created in the editor are shareable like any other rule.
- [ ] **Code rules**: the user writes a small JavaScript function `(cellState, neighborStates) → nextState`. It runs in a sandboxed Worker. Where the state space is small enough (2-state Moore = 512 inputs; multi-state totalistic = states × counts), the app enumerates every input once and compiles the function into a lookup table, so it runs as fast as a built-in rule. Code from a shared link or imported file never runs without an explicit "Run this code?" confirmation.

**Done when:** each rule family has parser tests plus at least one known-pattern behavior test; a rule made in the editor round-trips through a shared URL.

**Blog angle:** "Every 2-state cellular automaton is a 512-bit number" — plus a separate post on rule notations.

---

## Phase 7 — Cell topologies (hexagonal and more)

**Goal:** choose the shape of the cell.

- [ ] **Hexagonal grid**: store it in the same rectangular array using **axial coordinates** (the approach Golly uses), so the hex neighborhood is just a fixed set of 6 offsets. The renderer skews or offsets rows to draw real hexagons. Support the `H` rule suffix (e.g. `B2/S34H`).
- [ ] Hex-aware hit-testing (pointer → cell) and hex rendering for both zoomed-in (polygons) and zoomed-out (pixels) views.
- [ ] Decide how each boundary condition works on a hex grid; record it in DECISIONS.md.
- [ ] **von Neumann** square topology (if not already done in phase 6).
- [ ] Optional: **triangular** cells (neighbors depend on whether the triangle points up or down — the first topology with per-cell neighbor lists).
- [ ] Topology selector in the UI. Patterns and rules declare which topology they need; the app warns about mismatches.

**Done when:** known hex patterns behave correctly under hex rules in tests; clicking hexes hits the correct cell at all zoom levels.

**Blog angle:** "Life on a honeycomb."

---

## Phase 8 — WebAssembly engine (Rust)

**Goal:** a Rust implementation of the engine interface, chosen by benchmark rather than assumption.

- [ ] Set up a Rust crate (`wasm/`) built with wasm-pack; integrate it with Vite; add the Rust toolchain to CI.
- [ ] Implement the stepper for lookup-table rules on square/hex topologies, operating directly on a buffer shared with JS (no per-step copying).
- [ ] Run the **same test suite** against both engines (parameterize the tests by engine implementation).
- [ ] Benchmark TS vs WASM (with and without SIMD) across grid sizes. Record everything in DEVLOG.md.
- [ ] Engine selection: use the faster engine automatically, with a debug toggle to force either.

**Done when:** both engines pass identical tests; benchmark results are documented; WASM is used only where it is actually faster.

**Blog angle:** "Same automaton, two engines: TypeScript vs Rust/WASM, measured."

---

## Phase 9 — Unbounded ("infinite") grids

**Goal:** patterns that grow forever.

- [ ] Chunked sparse grid: a hash map of fixed-size tiles that are allocated when activity reaches them and freed when they die out. Implements the engine interface; the boundary concept doesn't apply to it.
- [ ] Viewport rendering for an unbounded plane (pan/zoom without limits, "fit pattern" button).
- [ ] **HashLife** (memoized quadtree) as a separate engine for huge generation jumps (step 2ⁿ generations at once). Show the generation count in a large-number format.
- [ ] A new route `/infinite` (or a mode toggle on `/2d`; decide and record why).

**Done when:** a glider gun runs for 10⁶+ generations in HashLife with correct population counts versus known values; memory use stays bounded while the gun's output streams away.

**Blog angle:** "HashLife explained: computing the far future by remembering the past."

---

## Phase 10 — 3D automata

**Goal:** a `/3d` page.

- [ ] 3D grid in the engine (same principles: flat array, halo, topology = 26-neighbor Moore / 6-neighbor von Neumann, lookup-table rules). 3D rule notation (e.g. `B5/S45` style and the common "survive/born/states/neighborhood" 3D notation); research and record in DECISIONS.md which notation to adopt.
- [ ] Three.js renderer using `InstancedMesh` (one draw call for all live cells), orbit controls, and optionally coloring by state or age.
- [ ] Editing in 3D: a movable slice plane where the user edits a 2D layer with the existing 2D tools; pattern stamping onto the active slice.
- [ ] 3D import/export format (decide and document; probably a simple JSON or a layered RLE).
- [ ] Performance: Worker/WASM engine from the start; decide whether rendering needs only the surface cells.
- [ ] Small-screen notice: 3D is desktop-only.

**Done when:** a 64³ grid runs and renders smoothly on a mid-range laptop; slice editing works.

**Blog angle:** "Cellular automata in three dimensions: rendering and editing a voxel world."

---

## Floating tasks (can slot in after Phase 4)

- [ ] `/1d` page: elementary cellular automata (Wolfram rules 0–255), drawn as a space-time diagram, plus totalistic 1D rules. A quick, very visual page and an easy blog post (Rule 30, Rule 110).
- [ ] Landing page with live mini-demos linking to each page.
- [ ] Event analytics (e.g. "pattern placed", "link shared") if Cloudflare Web Analytics proves too coarse. Candidates: GoatCounter or Umami; record the choice in DECISIONS.md.
- [ ] Custom domain.
- [ ] Accessibility pass (keyboard-operable controls, labels, reduced-motion setting).

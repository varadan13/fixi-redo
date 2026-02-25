---
name: fixi-redo-reimplementation
overview: Reimplement the minimalist fixi.js generalized hypermedia controls library in the empty fixi-redo repo, preserving the current API and behavior while setting up a clean structure for future evolution.
todos:
  - id: analyze-existing-fixi
    content: Catalog the existing fixi.js behavior, events, and attribute semantics from the old repo and README to use as a spec.
    status: pending
  - id: scaffold-fixi-redo
    content: Create minimal structure in fixi-redo (src, dist, test, README) without adding heavy tooling.
    status: pending
  - id: implement-core-engine
    content: Reimplement the core fixi engine (init, process, request config, fetch lifecycle, swap logic, MutationObserver) in src/fixi.js.
    status: pending
  - id: port-test-suite
    content: Port the browser-based test.html and ensure it runs against the new fixi-redo build with all tests passing.
    status: pending
  - id: document-api
    content: Write README for fixi-redo that documents attributes, events, and examples, referencing relevant MDN docs.
    status: pending
isProject: false
---

## Goal

Recreate the existing `fixi.js` behavior (as seen in the original `fixi` repo) inside the currently empty `fixi-redo` repo, keeping the library minimalist and API-compatible while organizing the codebase for easier future evolution (tests, packaging, extensions).

## Current Understanding of the Existing Repo

- **Original implementation files** (in the older repo):
  - `[fixi.js](/home/varadan13/dev/fixi/fixi.js)`: single-file, unbundled implementation of the library.
  - `[minimal-fixi.js](/home/varadan13/dev/fixi/minimal-fixi.js)`: ultra-minimal variant showing the bare core idea.
  - `[test.html](/home/varadan13/dev/fixi/test.html)`: browser-based test harness using mocked `fetch()` and visual test output.
  - `[README.md](/home/varadan13/dev/fixi/README.md)`: full documentation describing attributes, events, properties, examples, and philosophy.
- **Core behavior (from `fixi.js` & README):**
  - On `DOMContentLoaded`, a `MutationObserver` is attached and all existing `[fx-action]` elements are processed.
  - Each `fx-action` element gets a single async handler stored as `elt.__fixi`, with metadata (`evt`, `requests`).
  - The handler builds a config object (`cfg`) with:
    - `action`, `method`, `target`, `swap`, `body`, `headers`, `drop`, `transition`, `fetch`, etc.
    - It supports form participation, `FormData`, and request de-duplication (`drop` / `requests` set).
  - Event lifecycle around `fetch()`:
    - `fx:config` (configure/override `cfg`, maybe cancel or mock),
    - optional confirmation (`cfg.confirm()`),
    - `fx:before`, then `fetch`, then `fx:after` (with `response` + `text`),
    - `fx:error` on network/abort errors,
    - `fx:finally` always, `fx:swapped` after DOM swap + optional View Transition.
  - Swapping behavior via `cfg.swap`:
    - Can be a function, a property on the element, or an `insertAdjacentHTML` position.
    - Default target and swap are configurable with attributes, following README semantics.
  - Initialization events: `fx:init`, `fx:inited`, `fx:process` for manual re-processing.
  - Size & philosophy constraints: tiny single file, no build step, no dependencies, easy-to-debug unminified JS.

## Reimplementation Strategy in `fixi-redo`

### 1. Repository Structure & Tooling

- **Minimal but modern layout** in `fixi-redo`:
  - `src/fixi.js` (or `src/fixi.ts` if we decide to use TypeScript internally).
  - `dist/fixi.js` as the single-file distributable (what users will vendor or load via `<script>`).
  - `test/test.html` mirroring and incrementally porting the existing `test.html` from the old repo.
  - `README.md` in `fixi-redo` explaining usage, API, and differences from the original.
- **Keep philosophy:**
  - No runtime dependencies; distributable is a single unminified JS file.
  - Any build tooling (if added) is dev-only and must not bloat the shipped file or complicate usage.
- **Optional (can be phased in later):**
  - Add `package.json` and simple npm scripts for `build` and `test`.
  - A very light bundler/transform (or just `cp src/fixi.js dist/fixi.js`) so we stay close to the current no-build ethos.

### 2. Define Compatibility & Scope

- **Compatibility target:**
  - Match the current semantics documented in the original `README.md` for:
    - Attributes: `fx-action`, `fx-method`, `fx-target`, `fx-swap`, `fx-trigger`, `fx-ignore`.
    - Events: `fx:init`, `fx:inited`, `fx:process`, `fx:config`, `fx:before`, `fx:after`, `fx:error`, `fx:finally`, `fx:swapped`.
    - Properties: `document.__fixi_mo`, `elt.__fixi` (shape and behavior).
- **Non-goals for the first pass:**
  - Do not add new attributes or extended features (e.g., full htmx-style inheritance, history, queues) beyond what’s already in the README, except via clearly documented extension points/events.
  - Do not change the public event names, attribute names, or base behavior unless a bug or serious inconsistency is identified and explicitly documented.

### 3. Rebuild the Core Engine

- **3.1. Event & attribute helpers**
  - Implement helpers similar to the original:
    - `send(elt, type, detail, bubbles?)` for dispatching `CustomEvent("fx:" + type, ...)`.
    - `attr(elt, name, defaultVal)` for attribute lookup with defaults.
    - `ignore(elt)` that respects `fx-ignore` on the element or ancestors.
- **3.2. Initialization & lifecycle**
  - Implement `init(elt)` to:
    - Create `elt.__fixi` async handler if not present and not ignored.
    - Attach the event listener based on `fx-trigger` and element type (`form`, inputs, everything else) exactly as in the original.
    - Fire `fx:init` before initialization and `fx:inited` after.
  - Implement `process(node)` to:
    - Optionally early-return if `ignore(node)`.
    - Initialize `node` if it matches `[fx-action]`.
    - Initialize all descendants that match `[fx-action]`.
  - Wire up:
    - `document.addEventListener("fx:process", evt => process(evt.target))`.
    - `DOMContentLoaded` handler that:
      - Creates and stores `document.__fixi_mo` as a `MutationObserver` watching `document.documentElement` / `document.body`.
      - Calls `process(document.body)` for the initial DOM.
- **3.3. Request configuration and `fetch` lifecycle**
  - In `elt.__fixi`:
    - Maintain `elt.__fixi.requests` as a `Set` of active configs.
    - Determine form participation and construct `FormData` (`body`) in accordance with the existing rules.
    - Build `cfg` object with all fields used in the original implementation (`trigger`, `action`, `method`, `target`, `swap`, `body`, `headers`, `drop`, `abort`, `signal`, `preventTrigger`, `transition`, `fetch`, etc.).
    - Fire `fx:config` with `{ cfg, requests }` and respect `preventDefault()` / changes to `cfg`.
    - Respect `cfg.preventTrigger` to call `evt.preventDefault()` when appropriate.
    - Handle `GET` / `DELETE` query-param encoding via `URLSearchParams` and rewrite `cfg.action` correctly (without mutating behavior vs. the original).
  - `fetch` & confirm logic:
    - Handle optional `cfg.confirm` callback (sync or async) and abort the flow when it signals `false`.
    - Before sending, fire `fx:before` with `{ cfg, requests }` and respect `preventDefault()`.
    - Call `cfg.fetch(cfg.action, cfg)` (allows mocking via `fx:config`) and await `.text()`.
    - Fire `fx:after` with `{ cfg }`; if prevented, skip swapping.
    - On errors (including aborts), fire `fx:error` with `{ cfg, error }` and skip swapping.
    - In `finally`, remove `cfg` from `requests` and fire `fx:finally`.
- **3.4. DOM swap behavior**
  - Implement swapping exactly as in the existing `fixi.js`:
    - If `cfg.swap` is a function, call it with `cfg`.
    - Else if it matches `/(before|after)(begin|end)/`, use `insertAdjacentHTML`.
    - Else if `cfg.swap` is a property name on `cfg.target`, assign `cfg.target[cfg.swap] = cfg.text`.
    - Else if `cfg.swap !== 'none'`, throw `cfg.swap`.
  - Implement optional View Transition:
    - If `cfg.transition` is present, wrap swap in `await cfg.transition(doSwap).finished`.
    - Otherwise call `await doSwap()` directly.
  - Fire `fx:swapped` on `elt` after completion, and also on `document` if the element is no longer in the DOM.

### 4. Port and Strengthen the Test Suite

- **4.1. Bring over the existing tests**
  - Copy `test.html` into `fixi-redo/test/test.html` and adjust paths to load the new `dist/fixi.js`.
  - Maintain the in-browser test harness (`test`, `mock`, `assertEq`, etc.) as-is to avoid regressions.
- **4.2. Ensure coverage of all core features**
  - Confirm that existing tests still exercise:
    - Basic button/form behavior, form participation, and method handling.
    - Drop/throttling behavior using `cfg.drop` and `requests`.
    - Confirmation (`cfg.confirm`) and abort flows.
    - Swap variants (innerHTML, outerHTML, none, before/after positions, property swaps like `className`).
    - Dynamic content + MutationObserver (`fx:process` and added content being live).
    - Web component-related behavior.
    - Event dispatch consistency (`fx:init`, `fx:inited`, `fx:config`, `fx:before`, `fx:after`, `fx:finally`, `fx:error`, `fx:swapped`).
- **4.3. Optional incremental improvements**
  - Add a minimal script to run tests in headless browsers (e.g., `playwright` or `puppeteer`) if desired, but keep manual `test.html` as the primary, philosophy-aligned test harness.

### 5. Documentation & Examples in `fixi-redo`

- **Port the key sections of the original README** into the new `README.md` in `fixi-redo`:
  - Attributes table and semantics.
  - Events, properties, and extension patterns.
  - Minimal example and a couple of small patterns (click-to-edit, delete row, lazy load).
- **Update references**:
  - Clearly state that this is a reimplementation/redesign and note any small deliberate behavior changes or bug fixes.
  - Document the size goals and any differences from the original project’s constraints.

### 6. Optional Future Enhancements (Non-blocking)

- **TypeScript internal implementation**
  - Rebuild the core in `src/fixi.ts` and compile to `dist/fixi.js`, while keeping the public API identical.
- **Packaging & CI**
  - Publish to npm under a distinct name if desired, with ESM/CJS entry points.
  - Add GitHub Actions CI to run the test suite on pushes.

### Key Web Platform References

These are the primary browser APIs used and should guide implementation details:

- **Fetch API**: [https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- **FormData**: [https://developer.mozilla.org/en-US/docs/Web/API/FormData](https://developer.mozilla.org/en-US/docs/Web/API/FormData)
- **URLSearchParams**: [https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams)
- **MutationObserver**: [https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)
- **CustomEvent**: [https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent)
- **View Transition API**: [https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API)
- **HTML forms and form submission**: [https://html.spec.whatwg.org/multipage/forms.html](https://html.spec.whatwg.org/multipage/forms.html)


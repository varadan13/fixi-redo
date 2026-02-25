## fixi-redo

`fixi-redo` is a reimplementation of the experimental, minimalist
[`fixi.js`](https://github.com/bigskysoftware/fixi) generalized hypermedia
controls library. The goal is to preserve the existing public behavior while
organizing the codebase for easier evolution (tests, packaging, extensions).

### Repository Layout

- `src/fixi.js`: source file where the new core engine will live.
- `dist/fixi.js`: single-file distributable intended to be vendored or loaded
  via a `<script>` tag.
- `test/test.html`: browser-based test harness, mirroring and incrementally
  porting the original `test.html`.

The distributable remains a single unminified JavaScript file with **no runtime
dependencies**, in line with the original project philosophy.

### Compatibility Target

The behavior of `fixi-redo` is defined by the current semantics described in
the original [`fixi` README](https://github.com/bigskysoftware/fixi) for:

- **Attributes**
  - `fx-action`
  - `fx-method`
  - `fx-target`
  - `fx-swap`
  - `fx-trigger`
  - `fx-ignore`
- **Events**
  - `fx:init`
  - `fx:inited`
  - `fx:process`
  - `fx:config`
  - `fx:before`
  - `fx:after`
  - `fx:error`
  - `fx:finally`
  - `fx:swapped`
- **Properties**
  - `document.__fixi_mo`: the `MutationObserver` used to watch for new content.
  - `elt.__fixi`: the event handler installed on `fx-action` elements, with
    `evt` and `requests` properties.

As the core engine is rebuilt in `src/fixi.js`, each of these semantics will be
preserved unless a discrepancy or bug is explicitly identified and documented.

### Non‑Goals for the First Pass

For the initial reimplementation, the following are explicitly **out of scope**:

- Adding new attributes or extended features beyond those already documented in
  the original README (e.g., full htmx-style inheritance, history, or request
  queues), except via clearly documented extension points/events.
- Changing public event names, attribute names, or base behavior, unless a bug
  or serious inconsistency is identified and called out in this README.

This keeps `fixi-redo` aligned with the original library’s minimalist design
while leaving room for carefully considered extensions in future iterations.


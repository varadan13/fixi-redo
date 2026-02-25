## Design Patterns in fixi-redo

This document briefly maps parts of the `fixi-redo` engine (see `src/fixi.js`) to classic patterns from the *Gang of Four* book, while staying within idiomatic JavaScript and DOM APIs.

### Observer

- **What it is here**
  - `MutationObserver` watching the DOM for added nodes.
  - `CustomEvent("fx:*")` plus `addEventListener("fx:...")` listeners on elements and `document`.
- **Why it matches Observer**
  - The engine does not poll; instead, it reacts to **notifications**:
    - DOM mutations are “observed” by `MutationObserver`, which then calls `process(node)`.
    - Application code subscribes to `fx:*` events (e.g. `fx:config`, `fx:before`, `fx:after`, `fx:error`, `fx:finally`, `fx:swapped`) to react to lifecycle changes.

Relevant platform docs:

- MutationObserver: <https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver>  
- DOM events / `addEventListener`: <https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener>  
- CustomEvent: <https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent>  

### Command

- **What it is here**
  - `elt.__fixi` is an async function attached to each `[fx-action]` element.
- **Why it matches Command**
  - It encapsulates:
    - All the data needed for a request (`cfg`).
    - The operations to perform (build form data, emit events, issue `fetch`, swap DOM, etc.).
  - It can be invoked by different triggers (`click`, `submit`, `change`, or a custom `fx-trigger`) without changing the handler itself.

Relevant platform docs:

- JavaScript functions: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions>  
- Fetch API: <https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API>  
- FormData: <https://developer.mozilla.org/en-US/docs/Web/API/FormData>  

### Strategy

- **What it is here**
  - The `cfg.swap` property:
    - Can be a **function** supplied by user code via events.
    - Or a **named strategy** (`innerHTML`, `outerHTML`, `beforebegin`, `afterbegin`, `beforeend`, `afterend`, `none`, or any other property like `value` or `className`).
- **Why it matches Strategy**
  - The overall request lifecycle is fixed, but **how** the response text is applied to the DOM is chosen at runtime by swapping in different strategies.
  - Users can inject their own swap functions in `fx:config` or other events without modifying the engine.

Relevant platform docs:

- Element.innerHTML / outerHTML: <https://developer.mozilla.org/en-US/docs/Web/API/Element/innerHTML>  
- Element.insertAdjacentHTML: <https://developer.mozilla.org/en-US/docs/Web/API/Element/insertAdjacentHTML>  

### Template Method

- **What it is here**
  - The fixed high-level lifecycle in `elt.__fixi`:
    - `fx:config` → optional confirmation → `fx:before` → `fetch` → `fx:after` or `fx:error` → `fx:finally` → swap → `fx:swapped`.
- **Why it matches Template Method**
  - `src/fixi.js` defines the **skeleton** of the algorithm once.
  - Application code customizes specific steps by:
    - Mutating `cfg` in `fx:config` / `fx:before` / `fx:after`.
    - Providing `cfg.confirm` for confirmation.
    - Providing a custom `cfg.swap` function.
  - The engine guarantees the order and invariants; users provide the “hook” behavior.

Relevant platform docs:

- CustomEvent (event-based hooks): <https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent>  
- AbortController / AbortSignal: <https://developer.mozilla.org/en-US/docs/Web/API/AbortController>  

### Facade

- **What it is here**
  - The entire `fixi` engine (IIFE + `process` + `fx:*` events) acts as a **facade** over:
    - `fetch`, `FormData`, `URLSearchParams`, `AbortController`, `MutationObserver`, `CustomEvent`, `insertAdjacentHTML`, etc.
- **Why it matches Facade**
  - Users primarily interact through a small, declarative surface:
    - The `fx-*` attributes (`fx-action`, `fx-method`, `fx-target`, `fx-swap`, `fx-trigger`, `fx-ignore`).
    - The `fx:*` events for extension.
  - All the underlying web APIs and wiring are hidden behind this simpler interface.

Relevant platform docs:

- Fetch API: <https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API>  
- FormData: <https://developer.mozilla.org/en-US/docs/Web/API/FormData>  
- URLSearchParams: <https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams>  
- MutationObserver: <https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver>  

### Mediator

- **What it is here**
  - `fx:*` events on `document` act as a **mediator** between otherwise independent parts of the page:
    - Extensions listening on `document` (e.g. polling, debouncing, indicators in the original README).
    - Individual fixi-powered elements dispatching events.
- **Why it matches Mediator**
  - Elements do not talk to each other directly.
  - Instead, they communicate via events on a shared mediator (`document`), which coordinates behavior.

Relevant platform docs:

- CustomEvent: <https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent>  
- EventTarget / addEventListener: <https://developer.mozilla.org/en-US/docs/Web/API/EventTarget>  

### Chain of Responsibility

- **What it is here**
  - Multiple listeners can be attached to the same `fx:*` event (`fx:config`, `fx:before`, `fx:after`, etc.).
- **Why it matches Chain of Responsibility**
  - Each handler in the chain can:
    - Inspect and mutate `evt.detail.cfg`.
    - Decide to **stop further processing** by calling `preventDefault()`.
  - The engine doesn’t know or care which handler ultimately “handles” the event; it just walks the chain via the DOM event system.

Relevant platform docs:

- Event.cancelable and preventDefault: <https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault>  
- Event propagation model: <https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Events>  

### Decorator (via Extensions)

- **What it is here**
  - User-space extensions (e.g. debouncing, indicators, polling from the original fixi README) **wrap** existing behavior:
    - They attach additional listeners on `fx:init`, `fx:before`, `fx:after`, etc.
    - They often store extra data on `elt.__fixi` or related state.
- **Why it matches Decorator**
  - The base behavior (request + swap) stays the same.
  - Additional responsibilities (disable button, show indicator, debounce, poll) are layered on without modifying the engine source.

Relevant platform docs:

- Element datasets / attributes: <https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/dataset>  
- CustomEvent: <https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent>  

### Singleton

- **What it is here**
  - `document.__fixi_mo` is initialized once and reused.
- **Why it matches Singleton**
  - The engine explicitly checks `if (document.__fixi_mo) return` on startup.
  - This guarantees a single `MutationObserver` instance managing all fixi-powered content for the page.

Relevant platform docs:

- MutationObserver: <https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver>  

### Notes on GoF Reference

The canonical description of these patterns is in:

- *Design Patterns: Elements of Reusable Object-Oriented Software*  
  by Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides (Addison-Wesley).


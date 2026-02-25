## fixi-redo Architecture

This document describes the internal structure and request lifecycle of `fixi-redo`'s core engine in `src/fixi.js`.

### High-Level Engine Structure

```mermaid
flowchart TD
  A[IIFE\n;(function () { ... })] --> B[Environment Guards\n- document defined?\n- document.__fixi_mo?]
  B --> C[Global Wiring\n- document.__fixi_mo = new MutationObserver\n- DOMContentLoaded listener\n- fx:process listener]
  C --> D[Helpers\n- send(elt, type, detail, bub)\n- attr(elt, name, defaultVal)\n- ignore(elt)]
  D --> E[Processing Pipeline: process(node)\n- skip if fx-ignore\n- init(node)\n- init(descendants [fx-action])]
  E --> F[Initialization: init(elt)\n- fx:init\n- build elt.__fixi\n- choose trigger event\n- addEventListener\n- fx:inited]
  F --> G[Per-Request Handler: elt.__fixi(evt)\n- build FormData/body\n- build cfg object\n- fx:config\n- drop logic\n- confirm()\n- fx:before\n- fetch(cfg.action, cfg)\n- fx:after / fx:error\n- fx:finally\n- swap\n- fx:swapped on elt & document]
  C --> E
```

### Request Lifecycle

```mermaid
sequenceDiagram
  participant U as User
  participant E as fx-element (elt)
  participant H as elt.__fixi
  participant C as cfg
  participant S as Server

  U->>E: trigger event (click/submit/change)
  E->>H: call elt.__fixi(evt)
  H->>H: build FormData/body
  H->>C: build cfg (action, method, target, swap, headers, etc.)
  H->>E: dispatch fx:config { cfg, requests }
  alt event canceled or drop
    E-->>H: preventDefault / cfg.drop truthy
    H-->>U: return (no request)
  else proceed
    H->>E: dispatch fx:before { cfg, requests }
    alt canceled
      E-->>H: preventDefault (no request)
      H-->>U: return
    else ok
      H->>S: cfg.fetch(cfg.action, cfg)
      S-->>H: Response
      H->>H: cfg.response, cfg.text
      H->>E: dispatch fx:after { cfg }
      alt canceled
        E-->>H: preventDefault (no swap)
      else swap
        H->>H: doSwap(cfg) / ViewTransition
        H->>E: dispatch fx:swapped { cfg }
        H->>document: fx:swapped { cfg } (if elt removed)
      end
    end
  end
  H->>E: dispatch fx:finally { cfg }
```

### Key Platform APIs

The engine relies on the following web platform features:

- **MutationObserver** (DOM change observation): <https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver>
- **CustomEvent** (fx:\* event dispatch): <https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent>
- **Fetch API** (HTTP requests): <https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API>
- **FormData** (form participation): <https://developer.mozilla.org/en-US/docs/Web/API/FormData>
- **URLSearchParams** (query parameter encoding): <https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams>
- **AbortController / AbortSignal** (cancellation): <https://developer.mozilla.org/en-US/docs/Web/API/AbortController>
- **View Transition API** (optional transitions around swaps): <https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API>


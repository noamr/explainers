# Explainer for the `focusreset` Event

## Proponents

- Noam Rosenthal (Google)

## Participate

- [whatwg/html#12842 — Alternative to `blur` and `focusout` on removal](https://github.com/whatwg/html/issues/12842)
- [whatwg/html#13050 — HTML specification PR](https://github.com/whatwg/html/pull/13050)
- [whatwg/dom#1512 — DOM specification PR](https://github.com/whatwg/dom/pull/1512)
- [web-platform-tests/wpt#63595 — WPT PR](https://github.com/web-platform-tests/wpt/pull/63595)

---

## The Problem

When the currently focused element (or an ancestor containing it) is removed from the DOM, the browser synchronously resets focus to the viewport (`document.body` / `document.documentElement`) via the HTML Standard's [node-remove focus fixup rule](https://html.spec.whatwg.org/multipage/infrastructure.html#node-remove-focus-fixup)—**without firing `blur` or `focusout`**.

When an application or component library does not realize focus was reset:
- **Users lose their place on the page**: Keyboard focus silently drops to `document.body`, forcing users to `Tab` through the entire header and navigation bar again.
- **Focus escapes modal dialogs**: Removing a focused button inside a modal (e.g., "Next step") drops focus to `document.body`, letting the next `Tab` press escape into background content.
- **Assistive technologies lose context**: Screen-reader and keyboard-navigation users lose their active cursor without warning.

### Current Workarounds Are Expensive and Fragile

Because the platform fires no event when focus is lost on DOM removal, libraries and extensions rely on costly workarounds ([whatwg/html#12842](https://github.com/whatwg/html/issues/12842)):

1. **Document-wide `MutationObserver` ([Radix UI `FocusScope`](https://github.com/radix-ui/primitives/blob/main/packages/react/focus-scope/src/focus-scope.tsx#L122-L165))**:
   ```js
   const handleMutations = (mutations) => {
     if (document.activeElement !== document.body) return;
     for (const mutation of mutations) {
       if (mutation.removedNodes.length > 0) focus(container);
     }
   };
   new MutationObserver(handleMutations).observe(document.body, { childList: true, subtree: true });
   ```
   *Problems*: Runs on *every* DOM mutation on the page while a dialog is open, fails across Shadow DOM boundaries, and misfires if an unrelated node is removed while `document.activeElement` was already `document.body`.

2. **Subtree `removedNodes` scanning ([`focus-trap`](https://github.com/focus-trap/focus-trap/blob/master/index.js#L981-L1035))**:
   *Problems*: Misses Shadow DOM removals, requires manual bookkeeping of the last focused node, and runs even when the removal code already moved focus to a replacement element.

3. **Timer polling via `setInterval` ([Material UI `FocusTrap`](https://github.com/mui/material-ui/blob/master/packages/mui-material/src/Unstable_TrapFocus/FocusTrap.tsx#L360-L377), [Tridactyl](https://github.com/tridactyl/tridactyl/blob/f49e1c94bcd481fdc86fb219335bafc4ce0f25ab/src/content.ts#L180-L187))**:
   ```js
   setInterval(() => {
     if (document.activeElement !== lastActiveElement) onFocusChanged();
   }, 200);
   ```
   *Problems*: Continuous CPU wakeups and a multi-frame gap where focus is broken between timer ticks.

4. **Cross-browser divergence**: Gecko and WebKit follow the standard (no `blur` on removal), while Blink has been unable to remove legacy synchronous `blur`-on-removal without breaking sites and tools that have no alternative ([whatwg/html#12842](https://github.com/whatwg/html/issues/12842)).

---

## Proposed Solution: The `focusreset` Event on `Document`

When the focused area of a focused document is removed from the DOM, the browser synchronously resets focus to the viewport (as today) and marks the `Document` as having a pending focus reset.

At the end of the [mutation observer microtask](https://dom.spec.whatwg.org/#notify-mutation-observers) (after `MutationObserver` callbacks and `slotchange` events), if:
1. focus has **not** been explicitly moved to another focusable area in the meantime, and
2. the document still **has focus** ([has focus steps](https://html.spec.whatwg.org/multipage/interaction.html#has-focus-steps)),

the user agent fires an `Event` named `focusreset` (`bubbles: false`, `cancelable: false`, `composed: false`) at `Document`.

```webidl
partial interface Document {
  attribute EventHandler onfocusreset;
};
```

```js
document.addEventListener("focusreset", () => {
  // Focused element was removed and nothing else was focused:
  fallbackElement.focus();
});
```

---

## Examples & Use Cases

### 1. Focus traps without polling or `MutationObserver`
```js
document.addEventListener("focusreset", () => {
  const firstFocusable = dialog.querySelector(
    "button, [href], input, select, textarea, [tabindex]:not([tabindex='-1'])"
  );
  (firstFocusable ?? dialog).focus();
}, { signal });
```

### 2. Dynamic lists and roving `tabindex` widgets
```js
class TodoList extends HTMLElement {
  #lastIndex = -1;

  connectedCallback() {
    this.addEventListener("focusin", (e) => {
      const items = [...this.querySelectorAll("li[tabindex]")];
      this.#lastIndex = items.indexOf(e.target.closest("li[tabindex]"));
    });

    this.ownerDocument.addEventListener("focusreset", () => {
      if (this.#lastIndex === -1) return;
      const items = [...this.querySelectorAll("li[tabindex]")];
      const next = items[Math.min(this.#lastIndex, items.length - 1)];
      this.#lastIndex = -1;
      (next ?? this).focus();
    });
  }
}
```
If a delete handler explicitly runs `item.remove(); undoButton.focus();`, the `undoButton.focus()` call cancels the pending `focusreset` automatically.

### 3. Restoring focus across partial DOM updates
```js
let lastFocusedId = null;
document.addEventListener("focusin", (e) => { lastFocusedId = e.target.id || null; });
document.addEventListener("focusreset", () => {
  if (lastFocusedId) document.getElementById(lastFocusedId)?.focus();
});
```
Because `focusreset` runs in the mutation observer microtask, focus is restored before the next paint.

---

## Key Design Details

### Microtask timing (`notify mutation observers`)
Operations in `notify mutation observers` run in order:
1. `MutationObserver` callbacks
2. `slotchange` events
3. Clone and clear **pending focus reset documents**, firing `focusreset` at each document that still has focus.

This ensures `focusreset` is:
- **Safe from synchronous re-entrancy**: DOM removal completes before any script runs.
- **Coalesced per document**: Multiple focus/remove cycles in one task produce at most one `focusreset`.
- **Coordinatable with `MutationObserver` / `slotchange`**: Focus repairs inside a `MutationObserver` cancel `focusreset`; removals inside a `MutationObserver` fire `focusreset` in the same checkpoint.
- **Pre-paint**: Even if removal happens inside `requestAnimationFrame` or `ResizeObserver`, `focusreset` runs before rendering.

### Automatic cancellation when focus is moved
In HTML's [focus update steps](https://html.spec.whatwg.org/multipage/interaction.html#focus-update-steps), whenever focus actually changes (`old` or `new` focus chain contains an `Element` or `Document`), involved documents are removed from `pending focus reset documents`:
- `replacement.focus()`, `dialog.showModal()`, `dialog.show()`, or `popover.showPopover()` (with `autofocus`) cancels a pending `focusreset`.
- No-op focus calls (`removedElement.focus()`, `inertElement.focus()`, or `document.body.blur()`) do **not** cancel `focusreset`.

### Nested documents (`<iframe>`s) and focus relevance
`focusreset` checks HTML's `has focus steps` both when the node is removed and when the microtask runs:
- **Background tabs and unfocused frames**: Do not fire `focusreset`.
- **`iframeButton.remove(); iframe.remove();` in the same task**: When the microtask runs, the detached child document no longer has focus, so `focusreset` fires **only at the parent `Document`**.
- **Cross-origin `<iframe>`s**: Isolated per agent and per `Document`.

### Interaction with other features
- **`moveBefore()`**: Preserves focus and does **not** fire `focusreset`.
- **All DOM removal APIs**: Works across `remove()`, `removeChild()`, `replaceChildren()`, `innerHTML = ""`, shadow host removal, etc.
- **Non-removal fixups (`disabled`, `hidden`, `display: none`, `inert`) & Navigation API `focusReset`**: Already run focusing steps and fire `blur` / `focusout`, so they do not fire `focusreset`.

---

## Alternatives Considered

- **Firing `blur` / `focusout` synchronously on removal**: Unsafe—runs script mid-removal on partially updated DOM trees.
- **Firing `blur` / `focusout` asynchronously on the nearest connected ancestor**: Semantically inaccurate (the ancestor was never focused) and ambiguous when ancestors are simultaneously reparented or moved across shadow roots.
- **Firing `focusreset` during the rendering step instead of a microtask**: Would miss the current frame if a focused node is removed inside `ResizeObserver` or `requestAnimationFrame`, and would leave `document.activeElement === document.body` visible to subsequent tasks.
- **Exposing the removed element (`relatedTarget`)**: Would retain detached DOM subtrees in memory and leak closed Shadow DOM nodes on a `Document`-level event.
- **Firing unconditionally even when focus moved**: Would cause fallback handlers and focus traps to overwrite intentional `nextElement.focus()` calls.

---

## Security and Privacy

`focusreset` exposes no new information or capabilities:
- It fires only on a `Document` after a same-document DOM removal while `document.hasFocus()` is `true`—state already observable via `MutationObserver`, `document.activeElement`, and `document.hasFocus()`.
- It omits any reference to the removed node, preserving closed Shadow DOM encapsulation and avoiding detached DOM leaks.
- It does not cross origin boundaries between parent and child `<iframe>`s, and does not fire while in BFCache or in Private Browsing any differently than normal mode.

---

## Stakeholder Feedback & References

- **Chromium**: Positive ([ChromeStatus](https://chromestatus.com/feature/5068521917775872), [crbug.com/572677865](https://issues.chromium.org/issues/572677865))
- **Gecko**: Pending ([mozilla/standards-positions#1469](https://github.com/mozilla/standards-positions/issues/1469), [Bug 2081188](https://bugzilla.mozilla.org/show_bug.cgi?id=2081188))
- **WebKit**: Pending ([WebKit/standards-positions#732](https://github.com/WebKit/standards-positions/issues/732), [Bug 327387](https://bugs.webkit.org/show_bug.cgi?id=327387))

Many thanks for feedback and advice from Dominic Farolino, Mason Freed, Ryosuke Niwa, Scott O'Hara, Simon Pieters, and Steve Orvell.

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

Whenever the currently focused element can no longer hold focus—because it (or an ancestor) was **removed from the DOM**, or became **unfocusable** (`disabled`, `hidden`, `inert`, `display: none`, `visibility: hidden`, closed `<details>` / `popover`, etc.)—the platform automatically resets focus to the viewport (`document.body` / `document.documentElement`):

- **On DOM removal**, HTML's [node-remove focus fixup rule](https://html.spec.whatwg.org/multipage/infrastructure.html#node-remove-focus-fixup) resets `document.activeElement` to `document.body` synchronously **with zero events fired** (`blur` and `focusout` do not fire).
- **On style or attribute changes** (`disabled`, `inert`, `display: none`, `hidden`), HTML's [focus fixup rule](https://html.spec.whatwg.org/multipage/webappapis.html#focus-fixup-rule) resets focus to the viewport during *update the rendering*. Although `blur` and `focusout` fire in these cases, `focusout` also fires on every normal focus transfer (`relatedTarget` may even be `null` when moving focus into a closed shadow tree or non-focusable click target), making it difficult to distinguish *"the platform dropped focus back to the viewport"* from ordinary focus movement—and `focusout` still misses DOM removal altogether.

When an application or component library does not catch an unhandled focus reset to the viewport:
- **Roving-`tabindex` widgets, shortcuts, and focus traps break**: Although browsers preserve the *Sequential Focus Navigation Starting Point* (so pressing `Tab` still advances from the former position), `document.activeElement` drops to `document.body`—breaking arrow-key / roving-`tabindex` components, `Escape` / shortcut handlers, and focus-trap containment.
- **Focus escapes modal dialogs and composite widgets**: Removing or disabling a focused "Submit" / "Next step" button inside a dialog drops `activeElement` to `document.body`, breaking modal focus management.
- **Assistive technologies lose context ([w3c/aria#2920](https://github.com/w3c/aria/issues/2920))**: Screen-reader and keyboard-navigation users lose their active element without a unified signal for containers to restore focus.

### Current Workarounds Are Expensive and Fragile

Because the platform lacks a unified signal for unhandled focus resets—and fires nothing at all on DOM removal—libraries and extensions rely on costly workarounds ([whatwg/html#12842](https://github.com/whatwg/html/issues/12842)):

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
   *Problems*: Misses Shadow DOM removals, requires manual bookkeeping of the last focused node, and runs even when application or framework effect code is about to move focus to a replacement element before paint.

3. **Timer polling via `setInterval` ([Material UI `FocusTrap`](https://github.com/mui/material-ui/blob/master/packages/mui-material/src/Unstable_TrapFocus/FocusTrap.tsx#L360-L377), [Tridactyl](https://github.com/tridactyl/tridactyl/blob/f49e1c94bcd481fdc86fb219335bafc4ce0f25ab/src/content.ts#L180-L187))**:
   ```js
   setInterval(() => {
     if (document.activeElement !== lastActiveElement) onFocusChanged();
   }, 200);
   ```
   *Problems*: Continuous CPU wakeups and a multi-frame gap where focus is broken between timer ticks.

4. **Cross-browser divergence**: Gecko and WebKit follow the standard (no `blur` on removal), while Blink has been unable to remove legacy synchronous `blur`-on-removal without breaking sites and tools that have no alternative ([whatwg/html#12842](https://github.com/whatwg/html/issues/12842)).

---

## Proposed Solution: Bubbling `focusreset` Event After Rendering Focus Fixup

Whenever **the platform moves focus to the viewport** because the focused element was removed or ceased to be focusable (even in cases where `blur` and `focusout` already fired), the `Document` records a **pending `focusreset` target**:
- If the element is still connected (e.g., `disabled`, `hidden`, `inert`, `display: none`), the target is **that element itself**.
- If the element (or an ancestor) was removed from the DOM, the target is **its nearest surviving connected ancestor** (`Element`, `ShadowRoot`, or `Document`)—walking further up to the next parent if that surviving ancestor is also removed before rendering.

During [update the rendering](https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering), **immediately after the [focus fixup rule](https://html.spec.whatwg.org/multipage/webappapis.html#focus-fixup-rule) step**, if:
1. focus has **not** moved to another focusable area in the meantime (which cancels the pending `focusreset`), and
2. the document still **has focus** ([has focus steps](https://html.spec.whatwg.org/multipage/interaction.html#has-focus-steps)),

the user agent fires an `Event` named `focusreset` (`bubbles: true`, `composed: true`, `cancelable: false`) at the pending target.

```webidl
partial interface mixin GlobalEventHandlers {
  attribute EventHandler onfocusreset;
};
```

```js
dialog.addEventListener("focusreset", () => {
  // Focus inside `dialog` was reset to the viewport (via removal, disabled,
  // hidden, display:none, inert, or inside a child ShadowRoot) and nothing
  // else claimed focus before rendering:
  fallbackElement.focus();
});
```

---

## Examples & Use Cases

### 1. Scoped focus traps without `Document` listeners, polling, or `MutationObserver`
Because `focusreset` fires on the nearest surviving ancestor and bubbles (`composed: true`), a `<dialog>` or container can listen directly on itself—no `document` listener cleanup required, and all involuntary resets (`remove()`, `disabled`, `display: none`, `inert`, and internal Shadow DOM removals) are handled in one place:

```js
dialog.addEventListener("focusreset", () => {
  const firstFocusable = dialog.querySelector(
    "button:not(:disabled), [href], input:not(:disabled), select:not(:disabled), textarea:not(:disabled), [tabindex]:not([tabindex='-1'])"
  );
  (firstFocusable ?? dialog).focus();
});
```

### 2. Dynamic lists and roving `tabindex` widgets
Custom elements and Shadow DOM components can listen directly on `this` or `this.shadowRoot`:

```js
class TodoList extends HTMLElement {
  #lastIndex = -1;

  connectedCallback() {
    this.addEventListener("focusin", (e) => {
      const items = [...this.querySelectorAll("li[tabindex]")];
      this.#lastIndex = items.indexOf(e.target.closest("li[tabindex]"));
    });

    this.addEventListener("focusreset", () => {
      const items = [...this.querySelectorAll("li[tabindex]:not([hidden])")];
      const next = items[Math.min(this.#lastIndex, items.length - 1)];
      this.#lastIndex = -1;
      (next ?? this).focus();
    });
  }
}
```
If a delete handler or framework post-commit effect (`useEffect`, `LitElement.updated`, `requestAnimationFrame`) runs `undoButton.focus()` before rendering, that focus transfer cancels the pending `focusreset` automatically.

### 3. Restoring focus across partial DOM updates
Because `focusreset` bubbles up to `Document`, page-level coordinators can also observe `event.target` (the surviving container, retargeted across shadow boundaries) or restore focus by ID before paint:

```js
let lastFocusedId = null;
document.addEventListener("focusin", (e) => { lastFocusedId = e.target.id || null; });
document.addEventListener("focusreset", () => {
  if (lastFocusedId) document.getElementById(lastFocusedId)?.focus();
});
```

---

## Normative Processing Model

### 1. `Document` Bookkeeping ("Pending `focusreset` Target")

Each `Document` has an internal **pending `focusreset` target** (`EventTarget` or `null`, initially `null`). Whenever non-null, it is **always connected to `Document` (or is `Document` itself)**, so detached DOM subtrees are never retained in memory:

1. **On DOM removal ([node-remove focus fixup rule](https://html.spec.whatwg.org/multipage/infrastructure.html#node-remove-focus-fixup)):**
   When `removedNode` is removed from `oldParent` in `doc`:
   - If `removedNode` is a [shadow-including inclusive ancestor](https://dom.spec.whatwg.org/#concept-shadow-including-inclusive-ancestor) of `doc`'s focused area:
     1. Synchronously reset `doc`'s focused area to `doc`'s viewport (as today).
     2. If the [has focus steps](https://html.spec.whatwg.org/multipage/interaction.html#has-focus-steps) for `doc` return `true`, set `doc`'s **pending `focusreset` target** to `oldParent`.

2. **Walking up on subsequent ancestor removals ([DOM `remove`](https://dom.spec.whatwg.org/#concept-node-remove)):**
   Whenever *any* `removedNode` is removed from `oldParent` in `doc`:
   - Let `target` be `doc`'s **pending `focusreset` target**.
   - If `target` is non-null and `removedNode` is a **shadow-including inclusive ancestor** of `target`, set `doc`'s **pending `focusreset` target** to `oldParent`.
   - *(This handles chained upward removals like `btn.remove(); row.remove();` as well as reparenting `row` into a detached fragment, stopping once the ancestor remains connected or reaches a tree root such as `Document`.)*

3. **On rendering focus fixup ([focus fixup rule](https://html.spec.whatwg.org/multipage/webappapis.html#focus-fixup-rule)):**
   In *update the rendering*, when `doc`'s designated focused area `el` is no longer a focusable area (`disabled`, `hidden`, `inert`, `display: none`, `visibility: hidden`, etc.):
   - If the *has focus steps* for `doc` return `true`, set `doc`'s **pending `focusreset` target** to `el`.
   - Run the focusing steps for `doc`'s viewport (firing `blur` and `focusout` on `el` as today).
   - *(Note: if a `blur` or `focusout` listener moves focus to another element, step 2 below cancels `pending focusreset target`; if a `blur` or `focusout` listener removes `el`, rule 1.2 above walks `pending focusreset target` up to `el`'s surviving parent.)*

### 2. Automatic Cancellation in Focusing Steps

In HTML's [focus update steps](https://html.spec.whatwg.org/multipage/interaction.html#focus-update-steps) `(old chain, new chain, new focus target)`:
- Because [focusing steps](https://html.spec.whatwg.org/multipage/interaction.html#focusing-steps) bail out early before *focus update steps* when a candidate cannot be focused, reaching *focus update steps* guarantees focus actually changed.
- Whenever `new focus target` is **not** a `Document` viewport (i.e., another focusable area such as an `Element`, focusable scroll region, or child navigable gained focus), set `new focus target`'s `Document`'s **pending `focusreset` target** to `null` (along with any ancestor container `Document`s in `new chain`).
- **What cancels `focusreset`**: `replacement.focus()`, `dialog.showModal()`, `dialog.show()`, `popover.showPopover()` (with `autofocus`), or user focus changes before rendering.
- **What does NOT cancel `focusreset`**: No-op calls (`removedElement.focus()`, `disabledElement.focus()`, `inertElement.focus()`, `hiddenElement.focus()`, or `document.body.blur()`), and viewport resets performed by the platform itself.

### 3. Dispatching After "Focus Fixup" in *Update the Rendering*

In HTML's [update the rendering](https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering), immediately **after** the step that runs the *focus fixup rule* for each fully active `Document` `doc` in `docs` (which runs *after* `requestAnimationFrame` callbacks and `ResizeObserver`, and *before* `IntersectionObserver` and paint):

1. Let `target` be `doc`'s **pending `focusreset` target**.
2. If `target` is `null`, continue.
3. Set `doc`'s **pending `focusreset` target** to `null`.
4. If the *has focus steps* given `doc` return `false`, continue.
5. Fire an event named `focusreset` at `target`, with `bubbles` set to `true` and `composed` set to `true`.

Why this timing works across both frameworks and low-level DOM code:
- **Framework-friendly**: Synchronous handlers, microtasks (`LitElement.updated`, `Promise.then`), post-commit hooks (`useEffect`), `requestAnimationFrame` callbacks, and `ResizeObserver` callbacks all get a chance to call `nextElement.focus()` and cancel `focusreset` before it fires.
- **Pre-paint (including `rAF` / `ResizeObserver` removals)**: Because the focus fixup step runs *after* `requestAnimationFrame` and `ResizeObserver`, focus losses triggered inside `rAF` or `ResizeObserver` still fire `focusreset` before the frame is painted.
- **Coalesced & loop-safe**: Multiple focus/reset cycles in the same frame coalesce to a single `focusreset`, and clearing `pending focusreset target` before dispatch prevents infinite recursion within a single rendering turn.

### 4. Shadow DOM Encapsulation & Propagation

Because `focusreset` has **`bubbles: true` and `composed: true`** (matching `focusin` and `focusout`):
- **Inside a `ShadowRoot`**: If an internal shadow element is removed or disabled, `pending focusreset target` is the surviving internal shadow node (or the `ShadowRoot` itself if a direct child of `ShadowRoot` was removed). Internal listeners on shadow ancestors or `this.shadowRoot` see the exact surviving shadow target and can repair roving focus internally.
- **Across the shadow boundary**: Standard [DOM Event Retargeting](https://dom.spec.whatwg.org/#retarget) retargets `event.target` to `shadowRoot.host` as the event bubbles out into the light DOM—preserving closed Shadow DOM encapsulation while allowing outer `<dialog>` focus traps and `Document` listeners to catch unhandled resets.
- **Shadow host also removed (`shadowBtn.remove(); host.remove();`)**: Per [DOM § 4.2.2](https://dom.spec.whatwg.org/#concept-shadow-including-inclusive-ancestor), `host` is a shadow-including ancestor of both internal shadow elements and `ShadowRoot` itself, so removing `host` automatically walks `pending focusreset target` out to `host`'s surviving light-DOM parent.
- **Inner component handled vs. outer trap**: If an inner shadow listener calls `internalFallback.focus()`, then by the time outer light-DOM bubble listeners run, `document.activeElement` is already `host` (no longer `document.body`), or the inner component can call `event.stopPropagation()`.

### 5. Nested Documents (`<iframe>`s) and Other Platform Interactions

- **Nested `<iframe>`s**: `has focus steps` are checked both when recording and when dispatching `focusreset`. Background tabs and unfocused frames do not fire `focusreset`. If `iframeButton.remove(); iframe.remove();` run in the same frame, the detached child `Document` no longer has focus at rendering time, so `focusreset` fires **only in the parent `Document`** (at `iframe`'s surviving parent).
- **`moveBefore()`**: Moving the focused element within the same document preserves focus unless it lands in a hidden or `inert` subtree—in which case *update the rendering*'s focus fixup rule resets focus to the viewport and triggers `focusreset` automatically.
- **Explicit `element.blur()` & Navigation API `focusReset`**: Calling `element.blur()` is an explicit author action (not an involuntary platform fixup), and Navigation API's post-transition focus reset intentionally resets focus for a new page view rather than representing an accidental loss of focus within a widget.
- **Migration path for deprecating Blink's synchronous `blur` ([whatwg/html#12842](https://github.com/whatwg/html/issues/12842))**: Major UI libraries (`Radix`, `MUI`, `focus-trap`) already maintain fallback paths for Gecko and WebKit; shipping `focusreset` gives them a single standard container-level event so Blink can deprecate its legacy synchronous `blur` on removal.

---

## Alternatives Considered

- **Firing only on `Document` (`bubbles: false`)**: Requires every modal dialog, custom element, and roving-`tabindex` widget to register and manually clean up `ownerDocument` listeners, cannot distinguish which subtree lost focus when the focused node was removed, and misses local Shadow DOM handling.
- **Firing in the `MutationObserver` microtask instead of *update the rendering***: Fires prematurely before framework post-commit hooks (`useEffect`, `requestAnimationFrame`) can move focus to a replacement element, and cannot unify DOM removal with layout-dependent focus fixups (`display: none`, `visibility: hidden`). Running right after *update the rendering*'s focus fixup step still runs *after* `rAF` and `ResizeObserver` and *before* paint.
- **Firing `blur` / `focusout` synchronously on removal**: Unsafe—runs script mid-removal on partially updated DOM trees.
- **Firing `blur` / `focusout` asynchronously on the nearest surviving ancestor**: Semantically inaccurate on removal (the ancestor itself was never focused), and `focusout` cannot distinguish *"focus dropped to the viewport"* from ordinary focus movement into closed Shadow DOM or non-focusable click targets.
- **Exposing the removed element (`relatedTarget`)**: Would retain detached DOM subtrees in memory until rendering and leak closed Shadow DOM nodes across shadow boundaries.
- **Firing unconditionally even when another element gained focus**: Would cause fallback handlers and focus traps to overwrite intentional `nextElement.focus()` calls.

---

## Security and Privacy

`focusreset` exposes no new cross-origin or encapsulated information:
- It fires only within a focused `Document` when the platform resets focus to the viewport—state already observable via `document.activeElement`, `MutationObserver`, and `document.hasFocus()`.
- Walking `pending focusreset target` up on DOM removal guarantees that detached nodes are never retained or exposed, and `composed: true` uses standard DOM retargeting so closed Shadow DOM internals are never leaked to outer trees.
- Cross-origin `<iframe>`s remain isolated per `Document` and per agent, and BFCache / Private Browsing behave identically to normal browsing.

---

## Stakeholder Feedback & References

- **Chromium**: Positive ([ChromeStatus](https://chromestatus.com/feature/5068521917775872), [crbug.com/572677865](https://issues.chromium.org/issues/572677865))
- **Gecko**: Pending ([mozilla/standards-positions#1469](https://github.com/mozilla/standards-positions/issues/1469), [Bug 2081188](https://bugzilla.mozilla.org/show_bug.cgi?id=2081188))
- **WebKit**: Pending ([WebKit/standards-positions#732](https://github.com/WebKit/standards-positions/issues/732), [Bug 327387](https://bugs.webkit.org/show_bug.cgi?id=327387))

Many thanks for feedback and advice from Dominic Farolino, Mason Freed, Ryosuke Niwa, Scott O'Hara, Simon Pieters, and Steve Orvell.

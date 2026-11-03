# Explainer for the `focusreset` Event

## Proponents

- Noam Rosenthal (Google)

## Participate

- [whatwg/html#12842 — Alternative to `blur` and `focusout` on removal](https://github.com/whatwg/html/issues/12842)
- [whatwg/html#13050 — HTML specification PR](https://github.com/whatwg/html/pull/13050)
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

4. **Cross-browser divergence**: Gecko and WebKit follow the standard (no `blur` on removal), while some engines still fire a legacy synchronous `blur` on removal, which they cannot remove while sites and tools have no alternative signal ([whatwg/html#12842](https://github.com/whatwg/html/issues/12842)).

---

## Proposed Solution: Bubbling `focusreset` Event After Rendering Focus Fixup

Whenever **the platform moves focus to the viewport** because the focused element was removed or ceased to be focusable (even in cases where `blur` and `focusout` already fired), the `Document` records:
- a **pending `focusreset` target**:
  - if the element is still connected (e.g., `disabled`, `hidden`, `inert`, `display: none`), **that element itself**;
  - if the element (or an ancestor) was removed from the DOM, **its nearest surviving connected ancestor** (`Element`, `ShadowRoot`, or `Document`)—walking further up to the next parent if that surviving ancestor is also removed before rendering; and
- a **pending `focusreset` related target**: the `Element` that lost focus (retained only until the event fires or is cancelled, and retargeted across shadow boundaries following `focusout` encapsulation rules so closed Shadow DOM internals are never exposed).

This happens whether or not the document currently has system focus, so a reset that happens in a background tab or an unfocused `<iframe>` is not lost.

During [update the rendering](https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering), **immediately after the [focus fixup rule](https://html.spec.whatwg.org/multipage/webappapis.html#focus-fixup-rule) step**, if focus has **not** moved to another focusable area in the meantime (which cancels the pending `focusreset`), the user agent fires a `FocusResetEvent` named `focusreset` (`bubbles: true`, `composed: true`, `cancelable: false`) at the pending target, with `relatedTarget` exposing the element that lost focus (retargeted against `currentTarget`).

Hidden documents (background tabs, documents in the back/forward cache) don't run *update the rendering*, so for them the event fires in the first rendering update after they become visible again, **before the user sees the frame**. When the user returns to the tab, the page has already had a chance to put focus back where it belongs.

```webidl
[Exposed=Window]
interface FocusResetEvent : Event {
  constructor(DOMString type, optional FocusResetEventInit eventInitDict = {});
  readonly attribute Element? relatedTarget;
};

dictionary FocusResetEventInit : EventInit {
  Element? relatedTarget = null;
};

partial interface mixin GlobalEventHandlers {
  attribute EventHandler onfocusreset;
};
```

```js
dialog.addEventListener("focusreset", (e) => {
  // Focus inside `dialog` was reset to the viewport (via removal, disabled,
  // hidden, display:none, inert, or inside a child ShadowRoot) and nothing
  // else claimed focus before rendering.
  // `e.target` is the surviving ancestor; `e.relatedTarget` is the element
  // that lost focus (retargeted across shadow boundaries):
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
Because `focusreset` bubbles up to `Document` and exposes the removed or unfocused element on `event.relatedTarget` (retargeted across shadow boundaries), page-level coordinators can restore focus to a replacement element matching the previous ID or attributes with zero `focusin` bookkeeping:

```js
document.addEventListener("focusreset", (e) => {
  if (e.relatedTarget?.id) document.getElementById(e.relatedTarget.id)?.focus();
});
```

---

## Processing Model

At a high level, `focusreset` integrates into three existing HTML Standard hooks (`removing steps`, `focus update steps`, and `update the rendering`):

1. **What is recorded:** Whenever a `Document` resets focus to the viewport (whether or not it currently has system focus)—either synchronously on DOM removal (`#node-remove-focus-fixup`) or during rendering (`#focus-fixup-rule` for `disabled`, `hidden`, `inert`, `display: none`, `visibility: hidden`, etc.)—the `Document` records a **pending `focusreset` target** (the element itself if still connected, or its nearest surviving connected ancestor, walking further up if that ancestor is also removed before rendering) and a **pending `focusreset` related target** (the `Element` that lost focus).
2. **What cancels it:** Any successful focus transfer to a non-viewport focusable area before dispatch (`replacement.focus()`, `dialog.showModal()`, `autofocus`, or user navigation)—whether in synchronous code, microtasks, `useEffect`, `requestAnimationFrame`, or `ResizeObserver`—clears the pending fields and cancels `focusreset`. No-op calls (such as `.focus()` on a disabled/removed element) do not cancel it.
3. **When it fires:** Immediately after `#focus-fixup-rule` in the `Document`'s next *update the rendering* (before `IntersectionObserver` and paint), if `pending focusreset target` is still non-null, the browser clears both pending fields and fires `FocusResetEvent("focusreset", { bubbles: true, composed: true })` at that target. For a hidden `Document`, that is the first rendering update after it becomes visible.

Handlers run before paint, so a handler that calls `.focus()` can scroll the newly focused element into view before the frame is painted. This is intentional: the user sees focus (and scroll position) already restored, never the intermediate state.

| Boundary / Case | Behavior |
|---|---|
| **Shadow DOM** | `bubbles: true, composed: true`. Inside the shadow tree, `target` and `relatedTarget` expose the internal shadow nodes; across the shadow boundary, both retarget to `shadowRoot.host` (even if `relatedTarget` was detached from the `ShadowRoot` before dispatch), preserving closed-shadow encapsulation. |
| **Background tabs, unfocused windows & `<iframe>`s** | Recorded regardless of system focus. Visible documents fire at their next rendering update; hidden documents fire at the first rendering update after becoming visible. A handler in an unfocused `<iframe>` that calls `.focus()` follows the existing rules for focusing inside an unfocused frame; handlers that must not pull focus into the frame can check `document.hasFocus()`. If `iframeBtn.remove(); iframe.remove();` run in the same frame, the `<iframe>`'s document is discarded, so `focusreset` fires only in the parent `Document` (with `relatedTarget === iframe`). |
| **`dialog.close()` & `hidePopover()`** | See [Dialogs and popovers](#dialogs-and-popovers) below. |
| **Reparenting vs. `moveBefore()`** | `appendChild` runs `remove()` first, anchoring `pending focusreset target` to the original parent; `moveBefore()` preserves connectedness and focus unless moved into an `inert`/hidden subtree (caught by `#focus-fixup-rule`). |
| **Explicit `element.blur()` & Navigation API** | Neither triggers `focusreset` (`blur()` is intentional author action; Navigation API's post-transition focus reset is a page-level navigation transition). |

### Dialogs and popovers

Closing a dialog or popover that contains focus is the most common source of focus loss. Both [`dialog.close()`](https://html.spec.whatwg.org/multipage/interactive-elements.html#close-the-dialog) and [`hidePopover()`](https://html.spec.whatwg.org/multipage/popover.html#hide-popover-algorithm) already try to restore focus to the **previously focused element** (the element that had focus when the dialog or popover opened):

- **Previously focused element is still focusable:** Focus moves there. That is a normal focus transfer, so `focusreset` does not fire (and any pending `focusreset` is cancelled).
- **Previously focused element was removed or became unfocusable** (for example the list item that opened a "Delete?" dialog was deleted): the restore is a no-op, so focus stays on an element inside the now-hidden dialog or popover. The rendering focus fixup then resets focus to the viewport and `focusreset` fires **at that element**, bubbling through the closed `<dialog>` / popover and up to `Document`. A page-level or container-level listener can then pick a sensible fallback:

```js
list.addEventListener("focusreset", (e) => {
  // Fires if the opener (a deleted list item) can't take focus back.
  (list.querySelector("li[tabindex]") ?? list).focus();
});
```

- **No previously focused element** (for example the dialog was opened from script with nothing focused, or `<dialog open>` in markup): same as above—focus is reset by the rendering focus fixup and `focusreset` fires at the element inside the closed dialog or popover.

Because the event's target is inside the closed `<dialog>`, a listener on the dialog itself also receives it; dialog-scoped focus traps should check `dialog.open` before trying to refocus inside the dialog.

<details>
<summary><strong>Full Normative Processing Model & Edge-Case Rationale</strong></summary>

All required specification hooks (`removing steps`, `focus update steps`, and `update the rendering`) already exist in the HTML Standard—no changes to the DOM Standard are required.

Each `Document` has:
- an associated **pending `focusreset` target** (a `Node` or `null`, initially `null`)—whenever non-null, it is **always [connected](https://dom.spec.whatwg.org/#connected)** (`target`'s shadow-including root is `Document`); and
- an associated **pending `focusreset` related target** (an `Element` or `null`, initially `null`)—retained only until dispatch or cancellation by another focus change, and cleared immediately on dispatch.

To **retarget a `focusreset` related target** *related* (with surviving `Node`-or-null *contextTarget*) against *destination*:
1. Let *candidate* be the result of [retargeting](https://dom.spec.whatwg.org/#retarget) *related* against *destination*.
2. If *candidate* is a node, *candidate* is not [connected](https://dom.spec.whatwg.org/#connected), *contextTarget* is non-null, *contextTarget*'s root is a shadow root, and (*destination* is not a node or *contextTarget*'s root is not a shadow-including inclusive ancestor of *destination*):
   1. Return the result of [retargeting](https://dom.spec.whatwg.org/#retarget) *contextTarget*'s root's host against *destination*.
3. Return *candidate*.

*(Why this helper is needed: once `shadowBtn.remove()` detaches `shadowBtn` from its `ShadowRoot`, `shadowBtn`'s tree root is `shadowBtn` itself rather than the `ShadowRoot`, so plain DOM `retarget(shadowBtn, destination)` would stop early instead of walking out to `shadowRoot.host`. Using *contextTarget* restores the exact `focusout` shadow-retargeting chain across arbitrary open/closed shadow boundaries.)*

### 1. HTML `removing steps(removedNode, isSubtreeRoot, oldAncestor)`

In [DOM's `remove(node)` algorithm](https://dom.spec.whatwg.org/#concept-node-remove), `node` is unlinked from `parent` (`oldAncestor`) before any script can run, and HTML's [removing steps](https://html.spec.whatwg.org/multipage/infrastructure.html#node-remove-focus-fixup) are invoked first for the removed subtree root (`isSubtreeRoot = true`) while `removedNode`'s internal subtree and shadow trees are still intact and `oldAncestor` still has its exact pre-removal connectedness, and only afterwards for each shadow-including descendant (`isSubtreeRoot = false`).

Replacing HTML's `#node-remove-focus-fixup` step with a subtree-root check (`isSubtreeRoot = true` and `oldAncestor` is `connected`):

1. Let *document* be *removedNode*'s node document.
2. If *isSubtreeRoot* is true and *oldAncestor* is [connected](https://dom.spec.whatwg.org/#connected):
   1. Let *focusedAnchor* be *document*'s focused area's [DOM anchor](https://html.spec.whatwg.org/multipage/interaction.html#dom-anchor) if *document*'s focused area is non-null; otherwise null.
   2. If *focusedAnchor* is non-null and *removedNode* is a [shadow-including inclusive ancestor](https://dom.spec.whatwg.org/#concept-shadow-including-inclusive-ancestor) of *focusedAnchor*:
      1. Set *document*'s focused area to *document*'s viewport.
      2. Set *document*'s relevant global object's navigation API's focus changed during ongoing navigation to false.
      3. Set *document*'s **pending `focusreset` target** to *oldAncestor*.
      4. Set *document*'s **pending `focusreset` related target** to *focusedAnchor* if *focusedAnchor* is an `Element`; otherwise null.
   3. Otherwise, if *document*'s **pending `focusreset` target** is non-null and *removedNode* is a [shadow-including inclusive ancestor](https://dom.spec.whatwg.org/#concept-shadow-including-inclusive-ancestor) of *document*'s **pending `focusreset` target**:
      1. If *document*'s **pending `focusreset` related target** is non-null, set *document*'s **pending `focusreset` related target** to the result of **retargeting a `focusreset` related target** given *document*'s **pending `focusreset` related target**, *document*'s **pending `focusreset` target**, and *oldAncestor*.
      2. Set *document*'s **pending `focusreset` target** to *oldAncestor*.

Why crafting `removing steps` at `isSubtreeRoot = true` matters:
- **Immune to synchronous re-entrancy during descendant teardown**: Waiting until DOM's descendant loop (`isSubtreeRoot = false`) visits a deep focused node would run *after* earlier sibling `<iframe>`s destroy their child navigables (whose legacy synchronous `pagehide`/`unload` handlers could detach `oldAncestor` before the focused descendant is visited). Running at `isSubtreeRoot = true` records `oldAncestor` before any descendant `removing steps` run; if a legacy `pagehide` handler then removes `oldAncestor`, that nested `remove()`'s own `isSubtreeRoot = true` step advances **pending `focusreset` target** to `oldAncestor`'s surviving parent.
- **Ignores mutations in already-detached subtrees**: Guarding on `oldAncestor` being `connected` (`true` for connected `Element`s, `ShadowRoot`s of connected hosts, and `Document` itself; `false` for detached subtrees and `DocumentFragment`s) guarantees detached mutations never overwrite **pending `focusreset` target**.
- **Handles non-`Element` focusable areas (`DOM anchor`)**: Checking `focused area`'s `DOM anchor` handles elements, focusable scroll regions, UA subwidgets (`<video>`, `<input type="number">`), and `<img usemap>` anchors uniformly.
- **`O(1)` / `O(depth)` engine cost**: Removing a 10,000-node subtree runs this check once (not 10,000 times)—doing two pointer checks when focus is on the viewport and **pending `focusreset` target** is null, or walking `parent` / `ShadowRoot.host` up from at most one node.

### 2. Automatic Cancellation in `focus update steps`

In HTML's [focus update steps](https://html.spec.whatwg.org/multipage/interaction.html#focus-update-steps) `(old chain, new chain, new focus target)`:
- Because [focusing steps](https://html.spec.whatwg.org/multipage/interaction.html#focusing-steps) bail out early before *focus update steps* when a candidate cannot be focused, reaching *focus update steps* guarantees focus actually changed.
- After popping matching trailing entries from *old chain* and *new chain*: if *new focus target* is non-null and is **not** a `Document`'s viewport, then for each `Document` *doc* in *old chain* or *new chain*, set *doc*'s **pending `focusreset` target** to `null` and *doc*'s **pending `focusreset` related target** to `null`.
- **What cancels `focusreset`**: `replacement.focus()`, `dialog.showModal()`, `dialog.show()`, `popover.showPopover()` (with `autofocus`), or user focus changes before rendering.
- **What does NOT cancel `focusreset`**: No-op calls (`removedElement.focus()`, `disabledElement.focus()`, `inertElement.focus()`, `hiddenElement.focus()`, or `document.body.blur()`), and viewport resets performed by the platform itself.

### 3. Rendering Focus Fixup & Dispatch in *Update the Rendering*

In HTML's [update the rendering](https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering) (which runs *after* `requestAnimationFrame` callbacks and `ResizeObserver`, and *before* `IntersectionObserver` and paint):

1. **In the [focus fixup rule](https://html.spec.whatwg.org/multipage/webappapis.html#focus-fixup-rule) step (`#focus-fixup-rule`):** For each *doc* of *docs*, if the focused area of *doc* is not a focusable area (`disabled`, `hidden`, `inert`, `display: none`, `visibility: hidden`, etc.):
   1. Let *anchor* be *doc*'s focused area's [DOM anchor](https://html.spec.whatwg.org/multipage/interaction.html#dom-anchor).
   2. Set *doc*'s **pending `focusreset` target** to *anchor* if *anchor* is [connected](https://dom.spec.whatwg.org/#connected); otherwise *doc*.
   3. Set *doc*'s **pending `focusreset` related target** to *anchor* if *anchor* is an `Element`; otherwise null.
   4. Run the focusing steps for *doc*'s viewport (firing `blur` and `focusout` as today), and set *doc*'s relevant global object's navigation API's focus changed during ongoing navigation to false.
   *(If a synchronous `blur` or `focusout` listener moves focus to another element, § 2 clears **pending `focusreset` target** and **pending `focusreset` related target** to `null`; if that listener removes *anchor*, § 1.2.3 advances **pending `focusreset` target** to *anchor*'s surviving ancestor.)*

2. **Immediately after `#focus-fixup-rule`:** For each *doc* of *docs* (which, as today, only includes documents that are being rendered; a hidden document keeps its pending fields until it is rendered again):
   1. Let *target* be *doc*'s **pending `focusreset` target**, and let *related* be *doc*'s **pending `focusreset` related target**.
   2. Set *doc*'s **pending `focusreset` target** to `null` and *doc*'s **pending `focusreset` related target** to `null`.
   3. If *target* is non-null:
      1. Assert: *target* is [connected](https://dom.spec.whatwg.org/#connected) and *target*'s node document is *doc*.
      2. Fire an event named `focusreset` at *target*, using `FocusResetEvent`, with `bubbles` set to `true`, `composed` set to `true`, internal `relatedTarget` set to *related*, and internal `contextTarget` set to *target* (where `FocusResetEvent.prototype.relatedTarget`'s getter returns the result of **retargeting a `focusreset` related target** given internal `relatedTarget`, internal `contextTarget`, and `this.currentTarget`).

Why this timing works across both frameworks and low-level DOM code:
- **Framework-friendly**: Synchronous handlers, microtasks (`LitElement.updated`, `Promise.then`), post-commit hooks (`useEffect`), `requestAnimationFrame` callbacks, and `ResizeObserver` callbacks all get a chance to call `nextElement.focus()` and cancel `focusreset` before it fires.
- **Pre-paint (including `rAF` / `ResizeObserver` removals)**: Because the focus fixup step runs *after* `requestAnimationFrame` and `ResizeObserver`, focus losses triggered inside `rAF` or `ResizeObserver` still fire `focusreset` before the frame is painted. A handler's `.focus()` call may scroll the new target into view before paint; that is intended, so the first painted frame already shows the restored focus.
- **Coalesced and loop-safe**: Multiple focus/reset cycles in the same frame coalesce to a single `focusreset`, and clearing both pending fields before dispatch prevents infinite recursion. A detached `relatedTarget` is held for at most one frame in a visible document; a hidden document holds at most one such element until it is rendered again or focus moves.
- **Background documents**: Recording does not depend on system focus, so a reset caused by a timer, push message, or network response in a background tab is delivered before the user sees the tab again, instead of being silently lost.

### 4. Shadow DOM Encapsulation & Propagation Details

- **Why `FocusResetEvent` (like HTML's `ToggleEvent` / `CommandEvent`) rather than `FocusEvent`**: Events that populate DOM `Event`'s internal `relatedTarget` slot (such as `FocusEvent`) trigger DOM `dispatch`'s shadow-boundary suppression rule (`Otherwise, if parent is relatedTarget, then set parent to null`), which stops propagation at `shadowRoot.host` whenever `target` and `relatedTarget` originate inside the same shadow tree. Using `FocusResetEvent` with getter-level retargeting (matching `CommandEvent.prototype.source` and `ToggleEvent.prototype.source`) lets `focusreset` compose all the way to `Document` while keeping `relatedTarget` encapsulated.
- **Shadow host also removed (`shadowBtn.remove(); host.remove();`)**: Per [DOM § 4.2.2 (`shadow-including descendant`)](https://dom.spec.whatwg.org/#concept-shadow-including-descendant), `host` is a shadow-including inclusive ancestor of both internal shadow elements and `ShadowRoot` itself. Removing `host` advances **pending `focusreset` target** to `host`'s surviving light-DOM parent and retargets **pending `focusreset` related target** from `shadowBtn` to `host` via § 1.2.3—so light-DOM listeners see `event.relatedTarget === host` without leaking `shadowBtn`.
- **Inner component handled vs. outer trap**: If an inner shadow listener calls `internalFallback.focus()`, then by the time outer light-DOM bubble listeners run, `document.activeElement` is already `host` (no longer `document.body`), or the inner component can call `event.stopPropagation()`.
- **Relationship to legacy synchronous `blur` on removal ([whatwg/html#12842](https://github.com/whatwg/html/issues/12842))**: `focusreset` is independent of it: it fires the same way whether or not an engine also fires a legacy `blur`. Major UI libraries (`Radix`, `MUI`, `focus-trap`) already maintain fallback paths for engines without `blur` on removal; `focusreset` gives them one standard container-level signal, which makes it possible for engines to retire the legacy behavior on their own schedule.

</details>

---

## Alternatives Considered

- **Firing only on `Document` (`bubbles: false`)**: Requires every modal dialog, custom element, and roving-`tabindex` widget to register and manually clean up `ownerDocument` listeners, cannot distinguish which subtree lost focus when the focused node was removed, and misses local Shadow DOM handling.
- **Firing in the `MutationObserver` microtask instead of *update the rendering***: Fires prematurely before framework post-commit hooks (`useEffect`, `requestAnimationFrame`) can move focus to a replacement element, and cannot unify DOM removal with layout-dependent focus fixups (`display: none`, `visibility: hidden`). Running right after *update the rendering*'s focus fixup step still runs *after* `rAF` and `ResizeObserver` and *before* paint.
- **Firing `blur` / `focusout` synchronously on removal**: Unsafe—runs script mid-removal on partially updated DOM trees.
- **Firing `blur` / `focusout` asynchronously on the nearest surviving ancestor**: Semantically inaccurate on removal (the ancestor itself was never focused), and `focusout` cannot distinguish *"focus dropped to the viewport"* from ordinary focus movement into closed Shadow DOM or non-focusable click targets.
- **Reusing `FocusEvent` directly**: `FocusEvent` binds to DOM `Event`'s internal `relatedTarget` slot, which triggers DOM `dispatch`'s shadow-boundary suppression rule (`parent === relatedTarget`) and would swallow `focusreset` at `shadowRoot.host` whenever focus was lost inside a shadow tree; in addition, plain DOM `retarget()` does not retarget a node already detached from its `ShadowRoot`. Defining `FocusResetEvent` with getter retargeting (like `CommandEvent` and `ToggleEvent`) avoids both issues.
- **Firing unconditionally even when another element gained focus**: Would cause fallback handlers and focus traps to overwrite intentional `nextElement.focus()` calls.
- **Firing only while the document has system focus**: An earlier draft checked [has focus steps](https://html.spec.whatwg.org/multipage/interaction.html#has-focus-steps) when recording and when dispatching. Resets in background tabs (timers, push messages, network responses) were then silently dropped, and a returning user found the dialog or widget already broken.

### Naming

The Navigation API has a `focusReset` option (`navigation.navigate(url, { focusReset: "after-transition" })`). The overlap is mostly harmless, since both describe focus going back to the viewport, but the name `focusreset` could suggest the event fires for navigation focus resets, which it does not. **`focuscancel`** is a candidate alternative name, and we're open to renaming.

---

## Security and Privacy

`focusreset` exposes no new cross-origin or encapsulated information:
- It fires only when the platform resets focus to the viewport within the same `Document`—state already observable via `document.activeElement` and `MutationObserver`. Firing in background documents (when they are next rendered) exposes nothing new: the same script could already poll `document.activeElement` on `visibilitychange`.
- Both `event.target` and `event.relatedTarget` retarget across shadow boundaries (even when `relatedTarget` was detached from a `ShadowRoot` before dispatch), so closed Shadow DOM internals are never leaked to outer trees; any detached `relatedTarget` reference on `Document` is cleared on the next focus transfer or when the event fires.
- Cross-origin `<iframe>`s remain isolated per `Document` and per agent, and BFCache / Private Browsing behave identically to normal browsing.

---

## Stakeholder Feedback & References

- **Chromium**: Positive ([ChromeStatus](https://chromestatus.com/feature/5068521917775872), [crbug.com/572677865](https://issues.chromium.org/issues/572677865))
- **Gecko**: Pending ([mozilla/standards-positions#1469](https://github.com/mozilla/standards-positions/issues/1469), [Bug 2081188](https://bugzilla.mozilla.org/show_bug.cgi?id=2081188))
- **WebKit**: Pending ([WebKit/standards-positions#732](https://github.com/WebKit/standards-positions/issues/732), [Bug 327387](https://bugs.webkit.org/show_bug.cgi?id=327387))

Many thanks for feedback and advice from Dominic Farolino, Mason Freed, Ryosuke Niwa, Scott O'Hara, Simon Pieters, and Steve Orvell.

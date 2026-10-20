# Explainer for the `focusreset` Event

This proposal is an early design sketch by Chrome Web Platform to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Noam Rosenthal (Google)

## Participate

- WHATWG HTML issue: [whatwg/html#12842](https://github.com/whatwg/html/issues/12842)
- WHATWG HTML specification PR: [whatwg/html#13050](https://github.com/whatwg/html/pull/13050)
- WHATWG DOM specification PR: [whatwg/dom#1512](https://github.com/whatwg/dom/pull/1512)
- Web Platform Tests PR: [web-platform-tests/wpt#63595](https://github.com/web-platform-tests/wpt/pull/63595)

## Table of Contents

<!-- Update this table of contents by running `npx doctoc focusreset-explainer.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User research](#user-research)
- [Use cases](#use-cases)
  - [Use case 1: Maintaining focus traps in modal dialogs and overlays](#use-case-1-maintaining-focus-traps-in-modal-dialogs-and-overlays)
  - [Use case 2: Preserving keyboard position in dynamic lists, grids, and roving tabindex widgets](#use-case-2-preserving-keyboard-position-in-dynamic-lists-grids-and-roving-tabindex-widgets)
  - [Use case 3: Restoring focus after partial DOM updates or template re-renders](#use-case-3-restoring-focus-after-partial-dom-updates-or-template-re-renders)
  - [Use case 4: Assistive keyboard navigation tools and browser extensions](#use-case-4-assistive-keyboard-navigation-tools-and-browser-extensions)
- [Proposed Solution](#proposed-solution)
  - [The `focusreset` event on `Document`](#the-focusreset-event-on-document)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [Use case 1: Maintaining a focus trap without polling or `MutationObserver`](#use-case-1-maintaining-a-focus-trap-without-polling-or-mutationobserver)
    - [Use case 2: Recovering focus in a dynamic list or composite widget](#use-case-2-recovering-focus-in-a-dynamic-list-or-composite-widget)
    - [Use case 3: Focus recovery across declarative or framework partial DOM updates](#use-case-3-focus-recovery-across-declarative-or-framework-partial-dom-updates)
    - [Use case 4: Keyboard navigation tools](#use-case-4-keyboard-navigation-tools)
- [Detailed design discussion](#detailed-design-discussion)
  - [Microtask timing in `notify mutation observers`](#microtask-timing-in-notify-mutation-observers)
  - [Automatic cancellation when focus is explicitly moved](#automatic-cancellation-when-focus-is-explicitly-moved)
  - [Focus relevance and nested documents (`<iframe>`s)](#focus-relevance-and-nested-documents-iframes)
  - [Interaction with other platform features](#interaction-with-other-platform-features)
- [Considered alternatives](#considered-alternatives)
  - [Firing `blur` or `focusout` synchronously on node removal](#firing-blur-or-focusout-synchronously-on-node-removal)
  - [Firing `blur` or `focusout` asynchronously on the nearest still-connected ancestor](#firing-blur-or-focusout-asynchronously-on-the-nearest-still-connected-ancestor)
  - [Firing `focusreset` during the rendering update instead of a microtask](#firing-focusreset-during-the-rendering-update-instead-of-a-microtask)
  - [Exposing the removed element on the event (e.g., `relatedTarget`)](#exposing-the-removed-element-on-the-event-eg-relatedtarget)
  - [Firing `focusreset` unconditionally even when focus was subsequently moved](#firing-focusreset-unconditionally-even-when-focus-was-subsequently-moved)
  - [Firing `focusreset` in documents that do not have system focus](#firing-focusreset-in-documents-that-do-not-have-system-focus)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
  - [Self-Review Questionnaire: Security and Privacy](#self-review-questionnaire-security-and-privacy)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Users who navigate the web with a keyboard, switch device, or screen reader depend on focus staying in a predictable place as they interact with a page. In modern web applications, however, the currently focused element is frequently removed from the DOM as a normal part of using the interface—for example, when a user deletes an item from a list, clicks a "Next" button that swaps out a step inside a modal dialog, filters a table, or receives a live DOM update that re-renders part of the page.

When the focused element (or an ancestor containing it) is removed from the DOM, the browser synchronously resets focus to the viewport (`document.body` / `document.documentElement`) via the HTML Standard's [node-remove focus fixup rule](https://html.spec.whatwg.org/multipage/infrastructure.html#node-remove-focus-fixup)—**without firing `blur` or `focusout`**. When an application or component library does not realize that focus was reset:

- **Users lose their place on the page**: Keyboard focus is silently dropped back to the top of the document (`document.body`), forcing the user to `Tab` through the entire page header, navigation bar, and sidebar just to get back to the list or toolbar they were using.
- **Focus escapes modal dialogs and overlays**: In a modal dialog or drawer, removing a focused button (such as a "Confirm" or "Next step" button) silently moves focus to `document.body` outside the dialog, allowing the user's next `Tab` press to wander into background content that should be inert or hidden behind a backdrop.
- **Assistive technologies lose context**: Screen-reader users and keyboard-navigation tool users suddenly lose their active cursor without warning, with no announcement or visual focus ring indicating where focus went.

Because the platform fires no event when focus is lost on DOM removal, UI component libraries and accessibility tools have been forced to invent expensive, fragile workarounds in practice just to protect users from losing focus:

- **Observing every DOM mutation in the document**: [Radix UI's `FocusScope`](https://github.com/radix-ui/primitives/blob/main/packages/react/focus-scope/src/focus-scope.tsx#L122-L165) attaches a `MutationObserver({ childList: true, subtree: true })` to `document.body` whenever a dialog or focus scope is open, running a callback on *every* DOM mutation on the page solely to check whether `document.activeElement` silently became `document.body` so it can pull focus back into the dialog. Similarly, [`focus-trap`](https://github.com/focus-trap/focus-trap/blob/master/index.js#L981-L1035) attaches a subtree `MutationObserver` and walks every `removedNodes` list to check whether the removed node held focus.
- **Polling `document.activeElement` on a timer**: [Material UI's `FocusTrap`](https://github.com/mui/material-ui/blob/master/packages/mui-material/src/Unstable_TrapFocus/FocusTrap.tsx#L360-L377) runs a recurring `setInterval` timer that polls `doc.activeElement?.tagName === 'BODY'` to detect when the focused element has disappeared and restore focus into the trap. Browser extensions like [Tridactyl](https://github.com/tridactyl/tridactyl/blob/f49e1c94bcd481fdc86fb219335bafc4ce0f25ab/src/content.ts#L180-L187) similarly poll `document.activeElement` every 200ms in every tab because `blur` and `focusout` do not fire when the active element is removed—waking up the CPU continuously and still leaving a window where focus is broken between timer ticks.
- **Inconsistent behavior across browsers**: While Gecko and WebKit follow the HTML Standard by not firing `blur` on removal, Blink has historically been unable to remove its legacy synchronous `blur`-on-removal behavior without breaking sites and accessibility tools that have no other signal ([whatwg/html#12842](https://github.com/whatwg/html/issues/12842)).

This proposal introduces a `focusreset` event (and a corresponding `onfocusreset` event handler IDL attribute) on `Document`. Whenever a focused element is removed from a focused document—resetting focus to the viewport—the document is marked as having a pending focus reset. In the mutation observer microtask checkpoint (immediately after `MutationObserver` callbacks and `slotchange` events, and before the next rendering update), if focus has not been explicitly moved to another element and the document still has focus, a `focusreset` event is fired at `Document`. This gives applications and UI libraries a direct, zero-overhead, interoperable way to restore a user's focus before the next frame is even painted.

## Goals

- **Preserve keyboard and screen-reader usability when UI updates remove focused elements**: Give web authors and UI libraries a reliable, low-overhead signal when a user's focus is dropped to the viewport due to DOM removal, so they can immediately place focus on an appropriate fallback element.
- **Repair focus before rendering**: Fire `focusreset` at microtask timing (`notify mutation observers`) so that focus restoration happens before `requestAnimationFrame` callbacks and before the next frame is painted, preventing even a single frame of lost focus.
- **Avoid false positives when focus is already managed**: Automatically cancel a pending `focusreset` if script explicitly moves focus to another element (for example, `item.remove(); nextItem.focus();`) before the microtask runs.
- **Avoid synchronous re-entrancy hazards**: Never run script synchronously in the middle of DOM node removal steps.
- **Enable cross-browser interoperability**: Provide a safe, standard alternative to legacy non-standard synchronous `blur`-on-removal behavior ([whatwg/html#12842](https://github.com/whatwg/html/issues/12842)).

## Non-goals

- **Firing `blur` or `focusout` on node removal**: Firing synchronous events during DOM removal was intentionally removed from the HTML Standard because it allows script to observe and mutate partially updated DOM trees mid-removal, leading to re-entrancy bugs and security vulnerabilities.
- **Replacing rendering-time focus fixup for still-connected elements**: When a connected element becomes unfocusable without being removed from the DOM (for example, by setting `disabled`, `hidden`, `inert`, or `display: none`), the HTML Standard's [rendering-step focus fixup rule](https://html.spec.whatwg.org/multipage/webappapis.html#focus-fixup-rule) runs the focusing steps for the viewport, which already fires `blur` and `focusout`. `focusreset` is specifically scoped to the DOM removal gap.
- **Exposing a reference to the removed element on the event**: Retaining a reference to a detached node on the event object would risk keeping detached DOM subtrees alive in memory and could leak internal nodes across closed Shadow DOM boundaries.

## User research

Investigation in [whatwg/html#12842](https://github.com/whatwg/html/issues/12842) examined how widely used UI component libraries and keyboard-navigation tools handle focus loss when the active element is removed from the DOM, and why their current workarounds fall short in practice:

1. **Document-wide `MutationObserver` watching `document.activeElement` ([Radix UI `FocusScope`](https://github.com/radix-ui/primitives/blob/main/packages/react/focus-scope/src/focus-scope.tsx#L122-L165))**:
   ```js
   // Simplified from Radix UI FocusScope:
   // If the focused element gets removed from the DOM, browsers will move focus
   // back to the document.body. In this case, we move focus to the container.
   const handleMutations = (mutations) => {
     const focusedElement = document.activeElement;
     if (focusedElement !== document.body) return;
     for (const mutation of mutations) {
       if (mutation.removedNodes.length > 0) focus(container);
     }
   };
   const mutationObserver = new MutationObserver(handleMutations);
   mutationObserver.observe(document.body, { childList: true, subtree: true });
   ```
   - **Limitations**: Forces the browser to record and deliver mutation records for *every* DOM insertion and removal anywhere in `document.body` while a modal is open, fails to observe removals inside Shadow DOM trees, and can misfire if an unrelated node is removed while `document.activeElement` already happened to be `document.body`.

2. **Subtree `MutationObserver` scanning `removedNodes` ([`focus-trap`](https://github.com/focus-trap/focus-trap/blob/master/index.js#L981-L1035))**:
   - Watches the trap container with `{ childList: true, subtree: true }` and checks whether each removed node contains the most recently focused node (`removedNode.contains(mostRecentlyFocusedNode)`).
   - **Limitations**: Does not cross Shadow DOM boundaries, requires manual bookkeeping of the last focused node, and runs even if the application code that removed the node already moved focus to a valid replacement element.

3. **Timer polling (`setInterval`) ([Material UI `FocusTrap`](https://github.com/mui/material-ui/blob/master/packages/mui-material/src/Unstable_TrapFocus/FocusTrap.tsx#L360-L377) and [Tridactyl](https://github.com/tridactyl/tridactyl/blob/f49e1c94bcd481fdc86fb219335bafc4ce0f25ab/src/content.ts#L180-L187))**:
   ```js
   // Simplified from Tridactyl src/content.ts:
   // On top of blur/focusout events, we need to periodically check the activeElement
   // is the one we want because blur/focusout events aren't always triggered when
   // document.activeElement changes.
   setInterval(() => {
     if (document.activeElement !== lastActiveElement) {
       onFocusChanged();
     }
   }, 200);
   ```
   - **Limitations**: Causes continuous background CPU wakeups, and leaves a multi-frame gap (hundreds of milliseconds) where keyboard input and screen-reader announcements see lost focus before the timer fires.

## Use cases

### Use case 1: Maintaining focus traps in modal dialogs and overlays

A keyboard or screen-reader user opens a modal dialog or checkout drawer (such as a multi-step wizard or confirmation prompt). When the user activates a "Next" or "Delete" button inside the modal, the application removes that step's controls from the DOM and renders the next step. If the removed button held focus and the new step does not immediately autofocus a control, focus silently escapes the dialog and resets to `document.body`, allowing the user's next `Tab` press to navigate behind the modal overlay. The dialog component needs to know that focus was reset so it can keep focus inside the dialog container.

### Use case 2: Preserving keyboard position in dynamic lists, grids, and roving tabindex widgets

A user navigates a todo list, inbox message list, or tab strip using the keyboard. Either the user deletes the currently focused item, or a real-time background update removes that item (for example, a collaborator archives the message or a live-filtered row disappears). Instead of dropping the user's focus back to the top of the document—forcing them to `Tab` through the entire page header and navigation bar again—the list widget should immediately move focus to the adjacent item or to the list container.

### Use case 3: Restoring focus after partial DOM updates or template re-renders

When a portion of a page is replaced by a client-side router, an HTML-over-the-wire library (such as htmx or Turbo), or declarative partial DOM updates (`innerHTML`, `replaceChildren()`, or streaming `<template>` patches), a focused control inside the replaced subtree is detached and replaced by a newly parsed element with the same logical identity (such as an `id` or `data-focus-key`). The application or framework needs a hook that runs immediately after the DOM replacement—before the next paint—to restore focus to the newly inserted counterpart only if focus was lost during the update.

### Use case 4: Assistive keyboard navigation tools and browser extensions

Users who rely on keyboard-driven browsing extensions or custom focus indicators need their tooling to track when the active element disappears so that focus rings, spatial navigation cursors, or custom command modes remain synchronized without polling `document.activeElement` on a timer.

## Proposed Solution

### The `focusreset` event on `Document`

When the focused area of a document that currently has focus is removed from the DOM, the browser synchronously resets the document's focused area to the viewport (as it does today) and signals a pending focus reset for that `Document`.

At the end of the next [mutation observer microtask](https://dom.spec.whatwg.org/#notify-mutation-observers) (after all `MutationObserver` callbacks and `slotchange` events have been dispatched), if:
1. focus has **not** been explicitly moved to another focusable area or document in the meantime, and
2. the document still **has focus** (per HTML's [has focus steps](https://html.spec.whatwg.org/multipage/interaction.html#has-focus-steps)),

the user agent fires an `Event` named `focusreset` (`bubbles: false`, `cancelable: false`, `composed: false`) at the `Document` object.

`Document` also exposes the corresponding `onfocusreset` event handler IDL attribute:

```webidl
partial interface Document {
  attribute EventHandler onfocusreset;
};
```

Basic usage:

```js
document.addEventListener("focusreset", () => {
  // document.activeElement is currently document.body (or document.documentElement)
  // because the previously focused element was removed and nothing else was focused.
  fallbackElement.focus();
});
```

### How this solution would solve the use cases

#### Use case 1: Maintaining a focus trap without polling or `MutationObserver`

Instead of observing every DOM mutation on `document.body` or polling `document.activeElement` in a `setInterval`, a focus-trap component can listen for `focusreset` on `document` alongside `focusin`:

```js
class FocusTrap {
  #container;
  #abortController = new AbortController();

  constructor(container) {
    this.#container = container;
    const { signal } = this.#abortController;

    document.addEventListener("focusreset", () => {
      // Focus was dropped to the viewport because a focused node inside
      // the trap was removed. Keep focus within the trap container.
      const firstFocusable = this.#container.querySelector(
        "button, [href], input, select, textarea, [tabindex]:not([tabindex='-1'])"
      );
      (firstFocusable ?? this.#container).focus();
    }, { signal });
  }

  disconnect() {
    this.#abortController.abort();
  }
}
```

#### Use case 2: Recovering focus in a dynamic list or composite widget

A custom list component can track the index of the active item on `focusin` and restore focus to the nearest surviving item if `focusreset` fires—without interfering if the code that removed the item already focused a specific button:

```js
class TodoList extends HTMLElement {
  #lastFocusedIndex = -1;

  connectedCallback() {
    this.addEventListener("focusin", (e) => {
      const items = [...this.querySelectorAll("li[tabindex]")];
      this.#lastFocusedIndex = items.indexOf(e.target.closest("li[tabindex]"));
    });

    this.ownerDocument.addEventListener("focusreset", () => {
      if (this.#lastFocusedIndex === -1) return;
      const items = [...this.querySelectorAll("li[tabindex]")];
      const nextItem = items[Math.min(this.#lastFocusedIndex, items.length - 1)];
      this.#lastFocusedIndex = -1;
      (nextItem ?? this).focus();
    });
  }
}
```

If a delete handler explicitly runs `item.remove(); undoButton.focus();`, the explicit `undoButton.focus()` call cancels the pending `focusreset`, so the `focusreset` listener above will not override `undoButton`.

#### Use case 3: Focus recovery across declarative or framework partial DOM updates

When a subtree is replaced via `innerHTML` or a streaming patch, the application can record the focused element's selector or `id` on `focusin` and restore focus on `focusreset` if a matching element exists in the new DOM:

```js
let lastFocusedId = null;

document.addEventListener("focusin", (e) => {
  lastFocusedId = e.target.id || null;
});

document.addEventListener("focusreset", () => {
  if (!lastFocusedId) return;
  const replacement = document.getElementById(lastFocusedId);
  if (replacement) {
    replacement.focus();
  }
});
```

Because `focusreset` fires in the mutation observer microtask, `replacement.focus()` executes before the next rendering update—so `:focus` / `:focus-visible` styles and screen-reader focus remain continuous across the DOM swap.

#### Use case 4: Keyboard navigation tools

Tools like Tridactyl can replace their 200ms `setInterval` loop with a single `document.addEventListener("focusreset", ...)` listener alongside `focusout`, eliminating continuous background timer wakeups across every open tab.

## Detailed design discussion

### Microtask timing in `notify mutation observers`

When a focused node is removed, `focusreset` is **not** dispatched synchronously during the DOM removal algorithm. Instead, HTML's [removing steps](https://html.spec.whatwg.org/multipage/infrastructure.html#node-remove-focus-fixup) call DOM's [**signal a focus reset**](https://github.com/whatwg/dom/pull/1512) algorithm, which adds the `Document` to the surrounding agent's **pending focus reset documents** set and queues a mutation observer microtask.

Inside DOM's [**notify mutation observers**](https://dom.spec.whatwg.org/#notify-mutation-observers) algorithm, operations run in the following order:
1. All pending `MutationObserver` callbacks are invoked.
2. All pending `slotchange` events are fired.
3. The agent's **pending focus reset documents** set is cloned and cleared, and for each document in the clone that still passes the **focus reset relevance steps** (i.e., still has focus), a `focusreset` event is fired at that `Document`.

This timing has several important properties:

- **Safe from synchronous re-entrancy**: The DOM mutation that removed the focused subtree completes in full before any script is invoked.
- **Coalesced per document**: If multiple focus/remove cycles happen synchronously in the same task (e.g., `a.focus(); a.remove(); b.focus(); b.remove();`), only a single `focusreset` event is fired at the microtask checkpoint.
- **Coordinatable with `MutationObserver` and `slotchange`**: Because `pending focus reset documents` is cloned *after* `MutationObserver` callbacks and `slotchange` listeners run:
  - If a `MutationObserver` callback or `slotchange` listener repairs focus by calling `otherElement.focus()`, the pending `focusreset` is cancelled and does not fire.
  - Conversely, if a `MutationObserver` callback or `slotchange` listener itself removes the focused element, `focusreset` is fired in the same microtask checkpoint immediately afterward.
- **Runs before rendering**: Even if the focused element is removed inside a `requestAnimationFrame` callback or a `ResizeObserver` callback, the microtask checkpoint runs immediately after that callback returns—ensuring that any DOM or focus changes made by the `focusreset` handler are picked up in the current or next rendering update.
- **Non-reentrant**: Because `pending focus reset documents` is cloned and emptied before firing `focusreset`, if a `focusreset` listener focuses and removes another element (`b.focus(); b.remove()`), the second `focusreset` is queued for a subsequent microtask rather than firing re-entrantly.

### Automatic cancellation when focus is explicitly moved

A key ergonomic requirement is that `focusreset` should only notify the document when focus was *left* in its reset state, not when the code performing the removal already transferred focus elsewhere.

In HTML's [**focus update steps**](https://html.spec.whatwg.org/multipage/interaction.html#focus-update-steps), whenever focus moves such that the old or new focus chain (after removing common ancestor entries) contains an `Element` or `Document`—meaning an actual focus change occurred that will fire `blur` or `focus`—every `Document` involved in the old or new focus chain is removed from its agent's **pending focus reset documents**.

Consequently:
- Calling `replacement.focus()` synchronously after `focusedElement.remove()`, in an earlier `queueMicrotask()`, or inside a `MutationObserver` callback cancels the pending `focusreset`.
- Opening a `<dialog>` via `dialog.showModal()` or `dialog.show()`, or opening a popover with an `autofocus` descendant via `popover.showPopover()`, runs the focusing steps and therefore cancels a pending `focusreset`. (Opening a popover *without* an `autofocus` descendant does not move focus, so `focusreset` still fires.)
- Conversely, a **no-op** focus call—such as calling `removedElement.focus()` while `removedElement` is disconnected, calling `inertElement.focus()`, or calling `document.body.blur()` when focus is already on the viewport—does not update the focus chain and therefore **does not** cancel a pending `focusreset`.

### Focus relevance and nested documents (`<iframe>`s)

`focusreset` is gated on HTML's [**has focus steps**](https://html.spec.whatwg.org/multipage/interaction.html#has-focus-steps) at two points:
1. **When the node is removed**: `signal a focus reset` is only called if the `has focus steps` given `document` return `true`.
2. **When the microtask runs**: DOM's `notify mutation observers` checks the exported **focus reset relevance steps** hook (which HTML defines as running the `has focus steps` given `document`) before firing `focusreset`.

This two-stage check handles several subtle multi-document scenarios cleanly:

- **Background windows/tabs and unfocused frames**: If a node is removed in a background tab that lacks system focus, or inside a child `<iframe>` when focus is currently in the parent document, no `focusreset` is fired (just as the removed element did not have active system/frame focus).
- **Removing a focused element inside an `<iframe>` and then removing the `<iframe>` itself in the same task**:
  Suppose a button inside a child `<iframe>` is focused, and script runs:
  ```js
  iframeButton.remove();
  iframe.remove();
  ```
  When `iframeButton.remove()` runs, the child document has focus, so a focus reset is signaled for the child `Document`. Next, when `iframe.remove()` runs, `iframe` is the focused area of the parent `Document`, so the parent `Document`'s focused area is reset to its viewport and a focus reset is signaled for the parent `Document`.
  When the mutation observer microtask runs, `iframe` is no longer connected, so the child document's `has focus steps` return `false`, while the parent document's `has focus steps` return `true`. As a result, `focusreset` fires **only at the parent `Document`**, avoiding a redundant event on the detached child document.
- **Cross-origin `<iframe>`s**: Focus resets are tracked per agent and per `Document`. If a focused element inside a cross-origin `<iframe>` is removed, `focusreset` fires only at the child `Document` inside its own agent; the parent `Document` receives no event. Conversely, if the parent removes a focused cross-origin `<iframe>`, `focusreset` fires at the parent `Document`.

### Interaction with other platform features

- **State-preserving atomic move (`moveBefore()`)**: Moving a focused element (or an ancestor of the focused element) within the same document using `ParentNode.prototype.moveBefore()` runs DOM's state-preserving *moving steps* rather than the *removing steps*. Focus remains on the moved element and `focusreset` does **not** fire.
- **All DOM removal APIs**: Because `focusreset` hooks into HTML's `removing steps`, it works uniformly across every mechanism that removes a focused node from a document: `element.remove()`, `parent.removeChild()`, `replaceChild()`, `replaceWith()`, `replaceChildren()`, `innerHTML = ""`, `outerHTML = ""`, `textContent = ""`, `Range.prototype.deleteContents()`, `Range.prototype.extractContents()`, `document.adoptNode()`, `document.open()`, removing a shadow host whose shadow tree contains the focused element, or removing a slotted focused element.
- **Non-removal focus fixups (`disabled`, `hidden`, `display: none`, `inert`)**: Making a connected focused element unfocusable without removing it from the DOM triggers the asynchronous [focus fixup rule](https://html.spec.whatwg.org/multipage/webappapis.html#focus-fixup-rule) during the rendering update, which runs the focusing steps for the viewport and fires `blur` / `focusout`. Because `blur` already fires in those cases, `focusreset` does not fire.
- **Navigation API's `focusReset` option**: `NavigateEvent.prototype.intercept({ focusReset: "after-transition" })` resets focus on navigation completion via the [potentially reset the focus](https://html.spec.whatwg.org/multipage/nav-history-apis.html#potentially-reset-the-focus) algorithm, which runs the focusing steps (firing `blur` and `focus` events as appropriate) rather than firing `focusreset`.

## Considered alternatives

### Firing `blur` or `focusout` synchronously on node removal

Historically, some browsers fired `blur` and `focusout` synchronously while a focused node was being removed from the DOM. This behavior was removed from the HTML Standard because synchronous script execution in the middle of DOM tree removal (like legacy DOM Mutation Events) allows event listeners to observe inconsistent intermediate tree states and mutate the DOM while a removal is still in progress—a longstanding source of crashes and security vulnerabilities.

### Firing `blur` or `focusout` asynchronously on the nearest still-connected ancestor

In [whatwg/html#12842](https://github.com/whatwg/html/issues/12842), we initially explored recording the removed element's ancestor chain (or a live `Range` rooted at its parent) and firing `blur` or `focusout` asynchronously on the lowest still-connected ancestor. We moved away from this approach because:
- Firing `blur` on an ancestor element that was never itself focused or blurred is semantically inaccurate, and `FocusEvent.relatedTarget` has no natural value.
- If ancestors are simultaneously reparented, moved across shadow trees, or adopted into another document before the asynchronous step runs, determining which ancestor should receive the event becomes complex and arbitrary.
- What actually occurred is a document-level focus fixup to the viewport, which is most directly and accurately represented by an event on `Document`.

### Firing `focusreset` during the rendering update instead of a microtask

We considered firing `focusreset` in the event loop's *update the rendering* steps alongside the rendering-time focus fixup rule. However, microtask timing (in `notify mutation observers`, alongside `slotchange`) is strictly better for two reasons:
1. If a focused node is removed inside a `ResizeObserver` callback or a `requestAnimationFrame` callback, firing `focusreset` in a microtask ensures that any focus or DOM updates performed by the `focusreset` handler take effect in the *current* rendering update rather than being deferred to the next frame.
2. Tasks and `requestAnimationFrame` callbacks that run after the removal see focus already repaired by the `focusreset` handler rather than observing a transient state where `document.activeElement` is `document.body`.

### Exposing the removed element on the event (e.g., `relatedTarget`)

We considered using `FocusEvent` and populating `relatedTarget` (or a custom property) with the removed element. We decided to use a plain `Event` without a node reference because:
- Holding a reference to the removed element until the microtask runs (and potentially longer if the event object is retained by author script) can keep large detached DOM subtrees alive in memory.
- If the focused element was inside a closed shadow root when its shadow host or internal subtree was removed, exposing the removed element on a `Document`-level event would require retargeting or could leak encapsulated shadow nodes.

### Firing `focusreset` unconditionally even when focus was subsequently moved

If `focusreset` fired whenever a focused element was removed—even if script immediately called `nextElement.focus()` on the next line—then every `focusreset` listener (such as a document-level focus trap or fallback handler) would fire and overwrite `nextElement.focus()`. Cancelling the pending `focusreset` in the `focus update steps` makes `focusreset` a true "unhandled focus loss" notification.

### Firing `focusreset` in documents that do not have system focus

We considered firing `focusreset` whenever a document's focused area is reset on node removal, regardless of whether the document currently has focus (per the `has focus steps`). However, if a document is in a background window/tab without system focus, or in an unfocused sibling `<iframe>`, its active element did not actually hold the user's focus (and removing it would not have fired `blur` even if it were blurred normally). Gating both signaling and firing on the `has focus steps` also naturally prevents double-firing when a focused button inside an `<iframe>` and the `<iframe>` itself are both removed in the same task.

## Security and Privacy Considerations

`focusreset` exposes no new cross-origin information, user data, or device capabilities:
- It only fires on a `Document` as a direct consequence of a same-document DOM mutation (or removal of a child navigable container in that document) when that document already had focus (`document.hasFocus() === true`), both of which are already observable to scripts in that document via `MutationObserver` and `document.activeElement` / `document.hasFocus()`.
- It does not cross origin boundaries: a focus reset inside a cross-origin `<iframe>` only fires inside that `<iframe>`'s `Document`, and removing a focused `<iframe>` in the parent only fires `focusreset` on the parent `Document`.
- Because it fires asynchronously in the mutation observer microtask, it introduces no synchronous re-entrancy during DOM tree mutations.

### [Self-Review Questionnaire: Security and Privacy](https://w3c.github.io/security-questionnaire/)

1. **What information does this feature expose, and for what purposes?**
   It notifies a `Document` that its focused area was reset to the viewport due to DOM removal of the focused node while the document had focus, and was not subsequently updated before the mutation observer microtask checkpoint. All of this state (`document.activeElement`, `document.hasFocus()`, and DOM removals) is already synchronously observable to script in that document.

2. **Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?**
   Yes. The event is a plain `Event` (`bubbles: false`, `cancelable: false`, `composed: false`) fired at `Document`. It intentionally omits any reference to the detached node so that closed Shadow DOM encapsulation is preserved and detached DOM trees are not retained in memory.

3. **Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?**
   No.

4. **How do the features in your specification deal with sensitive information?**
   N/A.

5. **Does data exposed by your specification carry related but distinct information that may not be obvious to users?**
   No.

6. **Do the features in your specification introduce state that persists across browsing sessions?**
   No. The `pending focus reset documents` set is transient, in-memory microtask state cleared at the next mutation observer microtask checkpoint.

7. **Do the features in your specification expose information about the underlying platform to origins?**
   No. Whether the top-level traversable has system focus is already exposed via `document.hasFocus()`.

8. **Does this specification allow an origin to send data to the underlying platform?**
   No.

9. **Do features in this specification enable access to device sensors?**
   No.

10. **Do features in this specification enable new script execution/loading mechanisms?**
    No. It adds a standard DOM event (`focusreset`) and event handler IDL attribute (`onfocusreset`) on `Document` dispatched at the standard mutation observer microtask checkpoint.

11. **Do features in this specification allow an origin to access other devices?**
    No.

12. **Do features in this specification allow an origin some measure of control over a user agent's native UI?**
    No.

13. **What temporary identifiers do the features in this specification create or expose to the web?**
    N/A.

14. **How does this specification distinguish between behavior in first-party and third-party contexts?**
    Each `Document` only receives `focusreset` for focus resets of its own focused area while it has focus. A cross-origin `<iframe>` only receives `focusreset` when its own focused element is removed while that `<iframe>` has focus; it cannot observe focus resets in its embedding document, and the embedding document does not receive `focusreset` when an element inside the cross-origin `<iframe>` is removed.

15. **How do the features in this specification work in the context of a browser's Private Browsing or Incognito mode?**
    Identically to normal browsing mode; no state is persisted.

16. **Does this specification have both "Security Considerations" and "Privacy Considerations" sections?**
    Yes, provided in this explainer, and the normative changes are integrated into the WHATWG HTML and DOM Standards.

17. **Do features in your specification enable origins to downgrade default security protections?**
    No.

18. **What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?**
    A document in BFCache is not fully active and does not have focus (`has focus steps` return `false`), so `focusreset` is neither signaled nor fired while in BFCache.

19. **What happens when a document that uses your feature gets disconnected?**
    When a child navigable container (such as an `<iframe>`) is disconnected from its parent document, the child document no longer satisfies the `has focus steps` (checked by DOM's `focus reset relevance steps` before firing). Thus, any pending `focusreset` on the disconnected child document is suppressed, and `focusreset` fires only on the still-connected parent document if the removed `<iframe>` held focus.

20. **Does your spec define when and how new kinds of errors should be raised?**
    No new errors or exceptions are introduced.

21. **Does your feature allow sites to learn about the user's use of assistive technology?**
    No. `focusreset` is triggered solely by DOM node removal of `document`'s focused area, regardless of whether assistive technology is active.

22. **What should this questionnaire have asked?**
    Whether the event can be dispatched synchronously during DOM tree mutations (re-entrancy risk). As detailed above, `focusreset` is strictly dispatched in the mutation observer microtask checkpoint, never synchronously during node removal.

## Stakeholder Feedback / Opposition

- **Chromium**: Positive (proponent; implementation tracking bug: [crbug.com/572677865](https://issues.chromium.org/issues/572677865))
- **Gecko**: Standards position / implementation bug filed ([Mozilla Bug 2081188](https://bugzilla.mozilla.org/show_bug.cgi?id=2081188))
- **WebKit**: Standards position / implementation bug filed ([WebKit Bug 327387](https://bugs.webkit.org/show_bug.cgi?id=327387))
- **Web developers & accessibility community**: Positive feedback in [whatwg/html#12842](https://github.com/whatwg/html/issues/12842) and WHATNOT meetings.

## References & acknowledgements

- [WHATWG HTML Issue #12842: Alternative to `blur` and `focusout` on removal](https://github.com/whatwg/html/issues/12842)
- [WHATWG HTML PR #13050: Add the `focusreset` event](https://github.com/whatwg/html/pull/13050)
- [WHATWG DOM PR #1512: Add `signal a focus reset` and `focusreset` event](https://github.com/whatwg/dom/pull/1512)
- [Web Platform Tests PR #63595: `focusreset` event tests](https://github.com/web-platform-tests/wpt/pull/63595)
- [MDN Issue #876: Document `focusreset` event](https://github.com/mdn/mdn/issues/876)

Many thanks for valuable feedback and advice from:

- Dominic Farolino
- Mason Freed
- Ryosuke Niwa
- Scott O'Hara
- Simon Pieters
- Steve Orvell

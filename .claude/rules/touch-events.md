# Touch Events & Long-Press Rules

iOS Safari has aggressive native gesture handling that conflicts with custom touch interactions. Follow these rules for any long-press, drag, or swipe feature.

---

## 1. Never call `e.preventDefault()` on touchstart of a container

Calling `e.preventDefault()` in the touchstart of a card/container blocks click events on ALL child elements (buttons, links, inputs). This is the root cause of the "can't tap child button" bug.

```js
// WRONG — blocks child buttons
function startLongPress(e) {
  e.preventDefault(); // ← kills all child taps
  timer = setTimeout(doLongPress, 500);
}

// RIGHT — only prevent after threshold fires
function startLongPress(e) {
  timer = setTimeout(() => {
    // prevent scrolling only once long press is confirmed
    doLongPress();
  }, 500);
}
```

For child interactive elements (buttons, links) inside a long-press container, add all four event stoppers:
```html
ontouchstart="event.stopPropagation()"
ontouchend="event.stopPropagation()"
onmousedown="event.stopPropagation()"
onmouseup="event.stopPropagation()"
```

---

## 2. Drag handles use `touch-action: none`, not `preventDefault` on touchstart

For drag-to-reorder handles, set CSS `touch-action: none` on the handle element itself. Then call `e.preventDefault()` in the handle's own touchstart (not the container's).

```css
.drag-handle {
  touch-action: none;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}
```

```js
handle.ontouchstart = (e) => {
  e.preventDefault();   // OK here — this IS the drag target
  e.stopPropagation();
  startDrag(e);
};
```

---

## 3. Prevent scroll during drag in touchmove, not touchstart

Add `e.preventDefault()` inside the `touchmove` handler (registered with `{ passive: false }`), not in touchstart. This allows the initial tap to register before locking scroll.

```js
document.addEventListener('touchmove', (e) => {
  e.preventDefault(); // stops scroll during drag
  moveDrag(e);
}, { passive: false });
```

---

## 4. Suppress iOS system callout menus on custom interactive elements

Any element with custom long-press or drag behavior must suppress the iOS text-selection/callout menu:

```css
-webkit-touch-callout: none;
-webkit-user-select: none;
user-select: none;
```

Apply to: drag handles, long-press cards, reorder rows, custom context menu triggers.

---

## 5. Ghost/clone elements during drag

- Use `position: fixed` + `pointer-events: none` so the ghost never intercepts touch events
- Remove the ghost in both `touchend` and `touchcancel` handlers
- Register `touchcancel` as a fallback alongside `touchend`

---

## 6. Haptic feedback pattern

```js
navigator.vibrate(10);          // drag start
navigator.vibrate([4, 4, 4]);   // drop / confirm
```

Always guard: `if (navigator.vibrate) navigator.vibrate(...)` — not supported on iOS Safari (silent fail is fine).

---

## 7. Stale `startX` bug — swipe-back fires during drag

**Root cause**: A drag handle calls `stopPropagation()` on its `touchstart`, so the page-level swipe-back `touchstart` handler never fires. `startX` stays stale from the previous touch. The first `touchmove` event fires before `dy` grows large enough to fail the `Math.abs(dx) > Math.abs(dy) * 1.5` guard, so `dragging` is set to `true` based on a stale `dx`. The subsequent `touchend` then calls `close()`.

**Fix**: In every page-level swipe-back handler, check the drag state variable and bail out early:

```js
// In initSwipeBack, gate all three handlers:
page.addEventListener('touchstart', e => {
  if (_subDrag) return;           // ← drag in progress — skip
  startX = ...; startY = ...; dragging = false;
}, { passive: true });

page.addEventListener('touchmove', e => {
  if (_subDrag) { dragging = false; return; }  // ← reset stale dragging flag
  ...
}, { passive: false });

page.addEventListener('touchend', e => {
  if (_subDrag) { dragging = false; return; }  // ← prevent accidental close
  ...
});
```

**Rule**: Any page that has BOTH a swipe-to-close gesture AND draggable children must gate the swipe handlers on the drag state variable being null.

---

## 8. Reorder drag implementation pattern

See `_startSubDrag()` in demo.html for the reference implementation:
- `data-sub-id` on each draggable row
- Handle `ontouchstart` → `e.preventDefault()` + `e.stopPropagation()` → create fixed ghost clone
- `touchmove` on document (passive: false) → move ghost top
- `touchend` / `touchcancel` (once: true) → calculate new index from ghost center vs item centers → splice array → save + re-render

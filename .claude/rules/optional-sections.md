# Optional Section Pattern

Use this pattern whenever a form section is optional (user may or may not need it). Do **not** use click-to-expand rows, accordion arrows, or custom disclosure widgets.

---

## The pattern

```html
<div class="section-label">区块名称</div>
<div class="card">
  <div class="recur-check-row" style="border-top:none">
    <label>开启XXX</label>
    <button class="toggle-switch" id="xxxToggle" onclick="toggleXxx()"></button>
  </div>
  <div id="xxxSection" style="display:none">
    <!-- content rows, first one gets border-top:0.5px solid var(--border) -->
    <div class="settings-row" style="border-top:0.5px solid var(--border)">
      ...
    </div>
  </div>
</div>
```

```js
function toggleXxx() {
  const btn = document.getElementById('xxxToggle');
  const sec = document.getElementById('xxxSection');
  if (!btn || !sec) return;
  const open = !btn.classList.contains('on');
  btn.classList.toggle('on', open);
  sec.style.display = open ? 'block' : 'none';
  if (!open) { /* clear inputs when collapsing */ }
}
```

**Reset on sheet/page open:** always call `btn.classList.remove('on')` and `sec.style.display = 'none'` to restore collapsed state.

---

## Rules

- Toggle state class is **`.on`** — never `.active` or `.checked`.
- Collapsing always clears the section's inputs (so re-opening starts fresh).
- The content div gets `display:none` initially (not `height:0` or `visibility:hidden`).
- Never use `section-label` as the clickable trigger — the label stays static, the toggle button is the only interactive element.

---

## Existing uses in demo.html

| Feature | Toggle ID | Section ID |
|---------|-----------|------------|
| 组事项模式 | `groupToggle` | `groupSubSection` |
| 循环 | `recurToggle` | `recurOptions` |
| 截止日期 | `deadlineToggle` | `deadlineSection` |

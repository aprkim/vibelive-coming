# VibeLive -- Safe Design & Color Change Prompt

You are acting as a **senior frontend engineer** with strong design judgment.

---

## Goal

Apply **design and color improvements only**, while guaranteeing that **no existing functionality breaks**.

> This is a **non-functional change**.
> Behavior, logic, and interaction flow must remain **100% identical**.

---

## Hard Rules (MANDATORY)

### Do NOT change JavaScript logic

- No edits to event handlers
- No changes to function names
- No changes to async flows
- No reordering of logic

### Do NOT change or remove any existing:

- `id` attributes
- Class names referenced by JS, including but not limited to:
  - `.placeholder`
  - `.badge`
  - `.media-state`
  - `.video-tile`
  - `.video-grid`
- Inline handlers (`onclick="..."`)
- Element creation order relied on by JS

### Do NOT introduce CSS that blocks JS control

- No `display: none !important`
- No `pointer-events: none` on interactive elements
- No `position: fixed/absolute` overlays that can cover buttons
- No forced `z-index` unless required and safe

### Structure must remain stable

- You may NOT move elements that JS queries with:
  - `getElementById`
  - `querySelector`
- Wrapping elements is allowed **only if** it does not change selector behavior

---

## What You ARE Allowed To Change

- Colors
- CSS variables (`:root` tokens only)
- Borders, radius, shadows
- Spacing (padding / margin)
- Font weight / size / line-height
- Backgrounds using existing tokens
- Hover / focus styles using tokens
- Placeholder visuals without renaming classes

---

## Color System Rules

- All colors must come from **existing design tokens**
- Use `var(--bg)`, `var(--card)`, `var(--soft)`, `var(--accent)`, etc.
- Do not introduce raw hex colors in component styles
- Remove any hard-coded gradients that conflict with theme tokens
- Placeholder backgrounds must use:
  - `--soft`, `--accentSoft`, or neutral surfaces
- Accent color is used **only** for primary actions and highlights

---

## Display & Visibility Rules

- If JS toggles visibility with:
  ```javascript
  el.style.display = 'none' | 'block' | 'flex'
  ```
  then CSS **must not** override `display` with `!important`
- Default display states may be defined in CSS, but must remain override-able by JS

---

## Required Safety Checks Before Output

Before finishing, verify and confirm:

- [ ] No JS files were modified
- [ ] No IDs were renamed or removed
- [ ] No JS-referenced class names were changed
- [ ] No `display` or `visibility` rules block JS toggling
- [ ] All buttons remain clickable
- [ ] Video placeholders still toggle correctly
- [ ] Media state indicators still update visually
- [ ] Theme switching affects only CSS variables

---

## Output Requirements

Provide only:

- Updated CSS (or token overrides), and/or
- A clear list of CSS changes

Do **not** rewrite or restate unchanged code.

If a requested design change **risks breaking behavior**:

1. **STOP**
2. Explain the risk
3. Propose a safe alternative

---

## Success Definition

The UI should:

- Look more refined and consistent
- Respect the VibeLive color system
- Preserve all existing interaction behavior
- Pass all join / leave / toggle flows without regression

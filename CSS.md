# CSS

### Box model
```
┌──────────── margin (outside space) ────────────┐
│ ┌──────────── border ──────────────────────┐ │
│ │ ┌────────── padding ──────────────────┐ │ │
│ │ │          content (width×height)     │ │ │
`box-sizing: border-box;`  → width includes padding+border (always set globally)
```

### Selectors & specificity
element (1) < class/attr/pseudo-class (10) < id (100) < inline (1000); `!important` overrides (avoid). Later rule wins on tie.

### Flexbox (1-dimensional layout) ⭐
```css
.row { display: flex; justify-content: space-between; align-items: center; gap: 16px; flex-wrap: wrap; }
.item { flex: 1; }          /* share space equally */
```
`justify-content` = main axis, `align-items` = cross axis.

### Grid (2-dimensional)
```css
.cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); gap: 16px; }
```

### Responsive
```css
@media (max-width: 768px) { .sidebar { display: none; } }   /* mobile-first: base = mobile, add min-width for bigger */
```
Units: `px` fixed, `rem` relative to root font, `%`, `vw/vh`, `em`. `position`: static, relative, absolute (to nearest positioned ancestor), fixed, sticky. `z-index` needs a positioned element. Center a div: `display:grid; place-items:center;`. CSS variables: `--brand: #0a66c2; color: var(--brand);`. BEM naming (`card__title--active`), CSS Modules / Tailwind in React.

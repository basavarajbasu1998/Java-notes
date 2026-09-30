# CSS — Interview Deep Dive (modern CSS, 2024+ baseline)

> For a Java full-stack developer: CSS is "secondary" but interviews still probe it — cascade/specificity, box model, flex/grid, positioning + stacking, responsive design, performance, and "build this layout" tasks. Feature notes reflect broadly available browser support (`:has()`, container queries, `dvh`, `@layer`, native nesting, `subgrid`, `aspect-ratio`, `gap` in flex). Arithmetic in the flex examples was checked with a script; CSS snippets are written to be copy-paste correct (they are not executable in Node).
> Sections: 1 Mental model · 2 Cascade · 3 Box & flow · 4 Positioning & stacking · 5 Flexbox · 6 Grid · 7 Flex vs Grid · 8 Responsive · 9 Variables & theming · 10 Selectors · 11 Motion & performance · 12 Accessibility · 13 Architecture · 14 Twenty layout tasks · 15 Bugs & fixes · 16 Interview Q&A (34) · 17 Cheat sheet

---

## 1. 60-second mental model

**CSS = a rules engine that (1) picks which declaration wins per property per element (the cascade), (2) computes values (inheritance, `em/rem/%` → px), then (3) the browser lays out boxes (layout), draws them (paint) and stacks layers (composite).** Every element is a rectangular **box**; layout modes (`block/inline` flow, `flex`, `grid`, `position`) decide where boxes go.

**Analogy — a hotel with house rules:** many rule books apply to the same room (browser defaults, hotel policy, guest requests). When two rules disagree the front desk follows a strict ranking: *importance → origin/layer → specificity → order* (the cascade). Unspecified things are inherited from the parent room (font, colour). The room's furniture layout is then chosen by the layout mode: **flex** = furniture along one wall (a row/column), **grid** = a floor plan with numbered cells.

```
 Author CSS ─► CASCADE (who wins?) ─► COMPUTED STYLE ─► LAYOUT (geometry) ─► PAINT (pixels) ─► COMPOSITE (layers on GPU)
                 importance,layers,      inheritance,      reflow: expensive     repaint          transform/opacity live here
                 specificity, order      units → px
```
The three questions behind 90% of CSS bugs: **Which rule wins? (cascade) · What box am I in / who is my containing block? (layout) · Did I create a stacking context or BFC? (z-index, margins, floats).**

---

## 2. The cascade: origin, layers, specificity, order, inheritance

**Precedence (highest wins), simplified:** 1) transitions 2) `!important` UA 3) `!important` user 4) **`!important` author** 5) animations 6) **normal author** 7) normal user 8) normal UA (browser default). Inside author styles: **`@layer` order** (unlayered beats layered for normal rules; for `!important` the order **reverses**: earlier layers win) → **specificity** → **source order** (later wins).

**Specificity = (inline, IDs, classes/attributes/pseudo-classes, elements/pseudo-elements)** compared column by column, not as a decimal number (11 classes never beat 1 id).

| Selector | Score (a,b,c) |
|---|---|
| `*`, `>`, `+`, `~`, `:where(...)` | 0,0,0 |
| `p`, `::before` | 0,0,1 |
| `.card`, `[type=text]`, `:hover`, `:nth-child()` | 0,1,0 |
| `#nav` | 1,0,0 |
| `style="..."` | 1,0,0,0 (beats any selector) |
| `:is(#a, .b)`, `:not(#a)`, `:has(#a)` | takes the **most specific argument** (`:is(#a,.b)` = 1,0,0) |
| `#nav .item a:hover` | 1,2,1 |
| `ul li.active > a` | 0,1,3 |

```css
a { color: blue }                 /* 0,0,1 */
.link { color: green }            /* 0,1,0  wins over a */
nav a.link { color: red }         /* 0,1,2  wins over .link */
#menu a { color: black }          /* 1,0,1  wins over all class-based */
a { color: purple !important }    /* beats everything author-normal, incl. inline (unless inline is !important) */
```
Tie → **later rule wins**. `!important` is for utilities/overriding third-party CSS; overusing it starts "important wars" (fix with `@layer` or lower-specificity selectors).

**`@layer` (cascade layers)** control priority independent of specificity — great for resets, third-party CSS and design systems:
```css
@layer reset, base, components, utilities;      /* declared order = priority, later layer wins */
@layer reset { * { margin: 0 } }
@layer components { .btn { padding: .5rem 1rem; background: royalblue } }
@layer utilities { .p-0 { padding: 0 } }        /* beats .btn regardless of specificity */
.legacy { color: red }                          /* UNLAYERED styles beat all layered ones (normal declarations) */
@import url("bootstrap.css") layer(vendor);     /* tame third-party specificity */
```

**Inheritance:** some properties inherit by default — text-related: `color`, `font-*`, `line-height`, `letter-spacing`, `text-align`, `visibility`, `cursor`, `list-style`; most box-related do **not**: `margin`, `padding`, `border`, `background`, `width`, `display`, `position`. Control with `inherit`, `initial`, `unset`, `revert`, `revert-layer`. Value pipeline: *specified → computed → used → actual*. `currentColor` reuses `color` (borders/SVG icons).

**Reset strategy:** `*, *::before, *::after { box-sizing: border-box }` + `body { margin: 0; line-height: 1.5 }` + `img, video { max-width: 100%; height: auto; display: block }`.

---

## 3. Box model, display types, normal flow, margin collapsing

```
 ┌──────────────────────── margin (transparent, collapses vertically) ──────────────────────┐
 │ ┌────────────────────────── border ────────────────────────────────────────────────────┐ │
 │ │ ┌──────────────────────── padding (background shows here) ─────────────────────────┐ │ │
 │ │ │ ┌──────────────────── content  (width × height) ──────────────────────────────┐ │ │ │
 │ │ │ └───────────────────────────────────────────────────────────────────────────────┘ │ │ │
 │ │ └───────────────────────────────────────────────────────────────────────────────────┘ │ │
 │ └──────────────────────────────────────────────────────────────────────────────────────┘ │
 └──────────────────────────────────────────────────────────────────────────────────────────┘
 content-box (default): width = content only     → total = width + padding + border   (200 + 2*20 + 2*5 = 250px)
 border-box  (use it): width = content+padding+border → total = width                  (stays 200px; content = 150px)
```
**Percentages:** `width:%` is relative to the containing block's width; `padding/margin: %` are *always relative to the containing block's **width*** (even top/bottom → the aspect-ratio padding hack); `height: %` only works if the parent has a **definite** height.

**`display` types**

| Value | Behaviour |
|---|---|
| `block` | new line, fills available width, honours width/height/margin/padding on all sides |
| `inline` | flows in text; **ignores width/height and vertical margin**; vertical padding overflows visually; sits on the text baseline (source of "gap under images") |
| `inline-block` | flows inline but is a block inside: width/height/margins work; whitespace between elements creates gaps |
| `flex` / `inline-flex` | flex container (children = flex items) |
| `grid` / `inline-grid` | grid container |
| `flow-root` | block that creates a **BFC** (contains floats, stops margin collapse) |
| `contents` | element's box disappears, children participate in parent's layout (a11y caveats) |
| `none` | removed from layout **and** accessibility tree; `visibility:hidden` keeps space, still occupies layout, hidden from a11y; `opacity:0` invisible but focusable/clickable |
| `table*` | table layout (`display: table-cell` no longer needed for centering) |

**Normal flow:** blocks stack vertically, inline content wraps in line boxes. Leaving flow: `float` (text wrapping only), `position: absolute/fixed` (out of flow; parent doesn't grow), flex/grid children (participate in the parent's flex/grid formatting context).

**Margin collapsing** — vertical margins of *adjacent block-level boxes in the same block formatting context* merge into **one margin = the larger** (positive+negative → their sum: largest positive + most negative). Happens between: (1) adjacent siblings; (2) parent and first/last child when the parent has no border, padding, inline content, height or BFC separating them; (3) an empty block's own top & bottom. Never happens: horizontally, for floats, `absolute/fixed`, `inline-block`, **flex or grid items**, or across a BFC boundary.
```css
h2 { margin-bottom: 30px } p { margin-top: 20px }     /* gap = 30px, NOT 50px */
.parent { }  .parent > h1 { margin-top: 40px }        /* the 40px "escapes" the parent (parent moves down) */
/* fixes: .parent { display: flow-root } | padding-top: 1px | border | overflow: auto | use gap on a flex/grid parent */
```
**BFC** (block formatting context) is created by: root element, `float`, `position: absolute/fixed`, `display: flow-root | inline-block | table-cell | flex | grid` (for their contents), `overflow` ≠ `visible`, `contain: layout|paint`. A BFC contains its floats, prevents margin collapsing with children, and doesn't overlap floats.

---

## 4. Positioning, containing blocks, stacking contexts

| `position` | In flow? | Offsets (`top/left...`) relative to | Notes |
|---|---|---|---|
| `static` (default) | yes | ignored | `z-index` ignored |
| `relative` | yes (space kept) | its **own normal position** | creates a containing block for absolute children |
| `absolute` | **no** | **nearest positioned ancestor** (`position` ≠ static), else initial containing block | shrink-to-fit width; `inset: 0` fills the container |
| `fixed` | no | **viewport** (unless an ancestor has `transform/filter/perspective/contain: paint`, which becomes the containing block!) | header bars, modals |
| `sticky` | yes | scroll container: acts `relative` until the threshold (`top: 0`), then sticks **within its parent's box** | needs a threshold; **breaks if an ancestor has `overflow: hidden/auto`** (that ancestor becomes the scroll container) or the parent has no room |

```css
.card   { position: relative }                       /* anchor */
.badge  { position: absolute; top: -8px; right: -8px }   /* relative to .card */
.header { position: sticky; top: 0; z-index: 10 }
.cover  { position: absolute; inset: 0 }             /* = top/right/bottom/left: 0 */
```

**Stacking context & `z-index`.** Elements paint in a context in this order: context root background → negative z-index children → in-flow blocks → floats → inline content → `z-index: 0/auto` positioned → positive z-index. `z-index` only works on **positioned** elements and **flex/grid items**. A **stacking context** is an isolated group: children's z-index only compete *inside* it, and the whole context is then ordered as one unit against its siblings.

Created by: root `<html>`; `position: relative|absolute` **with** z-index ≠ auto; `position: fixed|sticky` (always); flex/grid item with z-index ≠ auto; `opacity < 1`; `transform`, `filter`, `perspective`, `clip-path`, `mask`; `will-change` of those; `isolation: isolate`; `mix-blend-mode ≠ normal`; `contain: layout|paint`.

```
 <header style="position:relative; z-index:1">      ── context A (z 1)
    <div dropdown style="position:absolute; z-index:9999">   ← trapped inside A
 <main style="position:relative; z-index:2">        ── context B (z 2) paints ABOVE the whole header incl. its 9999 dropdown
```
**Classic bug:** "my modal's `z-index: 9999` is under the sidebar" → an ancestor created a stacking context with a lower value (often via `transform`, `opacity`, `filter`). Fixes: render modals at `<body>` level (portals; `<dialog>` uses the top layer), remove the accidental context, or `isolation: isolate` on components so their internal z-indexes can't leak. Keep a z-index scale in variables (`--z-dropdown: 100; --z-modal: 1000`).

---

## 5. Flexbox in depth (one-dimensional layout)

```
 flex-direction: row (default)                              flex-direction: column
 main axis  ────────────────────────►                       main axis ↓ (top→bottom)
 ┌ container ───────────────────────────────┐ cross axis    justify-content = MAIN axis
 │ [item1] [item2] [item3]                  │      │        align-items     = CROSS axis (per line)
 └──────────────────────────────────────────┘      ▼        align-content   = CROSS axis, distributes LINES (only if wrap & >1 line)
                                                            align-self      = one item's cross alignment
```
Container props: `display:flex`, `flex-direction`, `flex-wrap`, `justify-content` (start, center, space-between, space-around, space-evenly), `align-items` (stretch default, center, flex-start, baseline), `align-content`, `gap`. Item props: `flex-grow`, `flex-shrink`, `flex-basis`, `align-self`, `order`, `margin: auto`.

**Shorthands:** `flex: 1` = `1 1 0%` (equal shares of *all* space, basis 0) · `flex: auto` = `1 1 auto` (grow from content size) · `flex: none` = `0 0 auto` (rigid) · `flex: 0 1 auto` = default (can shrink, doesn't grow). `flex-basis` beats `width` for the main size (unless `auto`, then `width` is the basis).

**The algorithm (what the browser does):**
1. **Flex base size** = `flex-basis` (or `width`/content if `auto`); **hypothetical main size** = base size clamped by `min-width/max-width`.
2. Collect items into lines (`nowrap` → one line; `wrap` → break when the hypothetical sizes exceed the container).
3. Compare the sum of hypothetical sizes with the container's inner main size: **less → free space is positive → grow; more → negative → shrink.**
4. **Resolve flexible lengths (loop):** distribute free space — *grow*: proportional to `flex-grow`; *shrink*: proportional to `flex-shrink × flex-base-size` (bigger items lose more). If an item hits its min/max it is **frozen at that size** ("violation") and the remaining space is redistributed among the others; repeat until no violations.
5. Cross size of each line, then `align-*` and `justify-content` on leftover space.

**Worked example A — shrink.** Container 500px, `nowrap`; items with `flex-basis: 200px` and `flex-shrink` 1, 1, 2. Sum = 600 → overflow 100px. Weights = basis × shrink = 200, 200, 400 (total 800). Each loses `100 × weight/800` → **25px, 25px, 50px** → final widths **175, 175, 150** (= 500). *(Intuition: `flex-shrink: 2` = "shrink twice as fast".)*

**Worked example B — grow.** Container 600px; `flex-basis: 100px`, `flex-grow` 1, 2, 3. Free = 600 − 300 = 300, split 1:2:3 → +50, +100, +150 → **150, 200, 250**. With `flex: 1 1 0%` on all three the basis is 0, so *all 600px* is shared 1:2:3 → **100, 200, 300** (that's why `flex: 1` gives equal columns even with different content).

**Worked example C — max clamp & redistribution.** Container 500px, three items `flex: 1 1 100px`, the 2nd has `max-width: 120px`. Free = 200 → tentative +66.67 each → item 2 would be 166.7 > 120 → **frozen at 120**; remaining free = 500 − 120 − 100 − 100 = 180 → +90 each → **190, 120, 190**.

**The `min-width: auto` overflow trap.** A flex item's automatic minimum size is its **min-content** size, so it refuses to shrink below its longest unbreakable word/child — long text, URLs, `<pre>`, tables or nested flex containers overflow instead of truncating. Fix on the *flex item*: `min-width: 0` (row) / `min-height: 0` (column), plus `overflow: hidden` or `overflow-wrap: anywhere`. The same applies to grid tracks: `1fr` is `minmax(auto, 1fr)` → use `minmax(0, 1fr)`.
```css
.row  { display: flex; gap: 12px }
.text { flex: 1; min-width: 0 }                                      /* allow shrinking */
.text > p { overflow: hidden; text-overflow: ellipsis; white-space: nowrap }
```

**Other facts:** `align-content` only matters with `flex-wrap: wrap` and multiple lines (with one line use `align-items`); auto margins absorb free space before `justify-content` (`margin-left: auto` pushes an item right — nav "logo left, links right"); `gap` replaces margin hacks; `order` changes visual not DOM/tab order (a11y issue); `flex-wrap: wrap` items with `flex: 1 1 200px` make simple responsive rows; `percentage height` on flex children needs a definite container height; a flex item's `margin` doesn't collapse; `z-index` works on flex items without `position`; `position: absolute` children leave the flex flow; `flex-basis: 0` vs `auto` differences appear when items have different content sizes.

---

## 6. Grid in depth (two-dimensional layout)

```
 grid-template-columns: 200px 1fr 2fr;     ── tracks = columns; lines are numbered 1…n+1
 grid-template-rows: auto 1fr auto;
  line1     line2            line3           line4
   │  200px  │   1fr (1/3)    │   2fr (2/3)   │      fr = share of LEFTOVER space after fixed tracks and gaps
 ──┼─────────┼────────────────┼───────────────┼──
   │ header  spanning all columns: grid-column: 1 / -1     (−1 = last line)
```
```css
.layout {
  display: grid;
  grid-template-columns: 240px minmax(0, 1fr);       /* fixed sidebar + fluid content that can shrink */
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header  header"
    "sidebar main"
    "footer  footer";
  gap: 16px;                                         /* row-gap column-gap */
  min-height: 100dvh;
}
.header { grid-area: header } .sidebar { grid-area: sidebar } .main { grid-area: main } .footer { grid-area: footer }
```
- **Track sizing:** `px/%`, `fr`, `auto` (content, stretches), `min-content/max-content`, `fit-content(300px)`, `minmax(min, max)`, `repeat(n, ...)`.
- **Implicit grid:** items beyond your template create rows/cols sized by `grid-auto-rows/columns`; `grid-auto-flow: row | column | dense`. **`dense`** back-fills holes with later smaller items (visual order ≠ DOM order — a11y caveat).
- **Placement:** `grid-column: 2 / 4` (lines), `grid-column: span 2`, `grid-row: 1 / -1`, named lines `[full-start]`, `grid-area`.
- **Alignment:** `justify-items / align-items` (items inside their cell), `justify-content / align-content` (whole grid inside the container when tracks don't fill it), `place-items: center` (shorthand), `place-self`.

**`auto-fit` vs `auto-fill`** (with `repeat(auto-…, minmax(200px, 1fr))`): both create as many ≥200px columns as fit. If there are *fewer items than columns*: **`auto-fill` keeps the empty tracks** (items stay ~200px, blank space on the right); **`auto-fit` collapses the empty tracks to 0** so `1fr` lets the existing items **stretch** to fill the row. With enough items both behave identically.
```css
.cards { display: grid; gap: 1rem;
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 240px), 1fr)); }   /* min(100%, …) prevents overflow on screens < 240px */
```
- **`subgrid`:** a nested grid uses its parent's tracks (`grid-template-columns: subgrid`) → align card internals (title/body/footer) across sibling cards.
- **Grid vs table:** grid items can overlap (`grid-area: 1 / 1` for stacking), have explicit gaps, and separate source from visual order.
- **Blowout:** long content inflates `1fr` columns → `minmax(0, 1fr)` and `min-width: 0` on items.

---

## 7. Flexbox vs Grid — when to use which

| Question | Flexbox | Grid |
|---|---|---|
| Dimension | one axis (row **or** column), content-driven | two axes at once, layout-driven |
| Sizing is driven by | items' content ("distribute what I have") | container's tracks ("place items into this plan") |
| Best for | nav bars, toolbars, button groups, centering, media object, tag lists, rows that wrap | page shells, dashboards, card grids, forms with aligned labels, overlapping/stacked items |
| Wrapping | wrapped lines don't align columns with each other | rows *and* columns stay aligned |
| Rule of thumb | "**content out**" | "**layout in**" |

They compose: grid for the page skeleton, flex inside components. If you find yourself giving flex items fixed percentage widths to fake columns, use grid.

---

## 8. Responsive design

**Setup:** `<meta name="viewport" content="width=device-width, initial-scale=1">` (without it mobile browsers emulate a ~980px viewport).

**Mobile-first:** base CSS = smallest screen, enhance with `min-width` queries (less overriding, ships less CSS to phones, forces content priority). Use content-driven breakpoints (where the layout breaks), not device names.
```css
.grid { display: grid; gap: 1rem }                                   /* 1 column by default */
@media (min-width: 40rem) { .grid { grid-template-columns: repeat(2, 1fr) } }
@media (min-width: 64rem) { .grid { grid-template-columns: repeat(4, 1fr) } }
@media (max-width: 40rem), (prefers-reduced-motion: reduce) { /* comma = OR */ }
@media (min-width: 40rem) and (orientation: landscape) { }
@media (hover: hover) and (pointer: fine) { .btn:hover { filter: brightness(1.1) } }   /* hover only where it exists */
@media (400px <= width <= 800px) { }                                 /* range syntax */
```
**Container queries** — components respond to *their container's* width, not the viewport (reusable card in sidebar vs main):
```css
.card-wrap { container-type: inline-size; container-name: card }
@container card (min-width: 420px) { .card { display: grid; grid-template-columns: 160px 1fr } }
/* container query units: cqw, cqi */
```
**Fluid sizing:** `clamp(min, preferred, max)` — `font-size: clamp(1rem, 0.9rem + 1vw, 1.5rem)`; `padding-inline: clamp(1rem, 4vw, 3rem)`; `min()/max()`: `width: min(100% - 2rem, 70rem)`. Include a `rem` term in the preferred value so browser zoom/font settings still work (pure `vw` text fails WCAG 1.4.4).

**Units**

| Unit | Relative to | Use |
|---|---|---|
| `px` | CSS pixel (not device pixel) | borders, shadows, hairlines |
| `rem` | root `font-size` (16px default) | font sizes, spacing, media queries — respects user settings |
| `em` | **this element's** font-size (for `font-size` itself: parent's); compounds when nested | padding/margins that should scale with a component's text |
| `%` | property-specific parent value | fluid widths |
| `vw/vh` | 1% viewport width/height | full-screen sections |
| `svh/lvh/dvh` | small / large / **dynamic** viewport height | **mobile URL bar fix** |
| `ch` | width of "0" in current font | `max-width: 65ch` readable line length |
| `lh`, `ex`, `cqw`, `fr` | line-height, x-height, container width, grid share | |

**`100vh` on mobile:** `vh` equals the *largest* viewport (URL bar hidden), so a `100vh` hero is taller than the visible area when the bar is shown (bottom buttons cut off). Use `min-height: 100dvh` (dynamic; follows the bar, may cause resize jank) or `100svh` (always fits). Provide a fallback line first: `min-height: 100vh; min-height: 100dvh;`.

**Images:** `img { max-width: 100%; height: auto }`; set `width`/`height` attributes (browser reserves space via aspect ratio → no CLS); `object-fit: cover|contain` + `object-position`; `aspect-ratio`.
```html
<img src="hero-800.jpg" srcset="hero-400.jpg 400w, hero-800.jpg 800w, hero-1600.jpg 1600w"
     sizes="(min-width: 64rem) 50vw, 100vw" width="800" height="450" loading="lazy" alt="…">
<picture>                                         <!-- art direction / format negotiation -->
  <source type="image/avif" srcset="hero.avif"> <source media="(min-width: 40rem)" srcset="hero-wide.webp">
  <img src="hero.jpg" alt="…" width="800" height="450">
</picture>
```
`srcset` + `sizes` lets the browser choose by width & DPR; `picture` lets *you* choose (crop, format). Test with DevTools device mode + real devices; avoid horizontal scroll (usual culprits: `100vw`, fixed widths, long words, wide tables → `overflow-x: auto` wrapper).

---

## 9. Custom properties (CSS variables), theming, dark mode

```css
:root { --brand: #0a66c2; --space: 8px; --radius: .5rem; color-scheme: light dark }
.btn { background: var(--brand); padding: calc(var(--space) * 2); border-radius: var(--radius) }
.btn.danger { --brand: #c0392b }                       /* scoped override: children inherit the new value */
.x { width: var(--w, 200px) }                          /* fallback if --w undefined */
```
Custom properties **cascade and inherit** (unlike Sass variables, which are compile-time); they are resolved at computed-value time, can be changed at runtime (`el.style.setProperty('--x', '10px')`), used inside `calc()`, and scoped to any subtree. `@property --angle { syntax: "<angle>"; inherits: false; initial-value: 0deg }` makes them typed and animatable. Invalid at computed time → the property becomes `unset` (silent failure — check the Computed pane).

**Dark mode**
```css
:root { --bg: #ffffff; --fg: #1a1a1a }
@media (prefers-color-scheme: dark) { :root { --bg: #121212; --fg: #eaeaea } }
:root[data-theme="dark"]  { --bg: #121212; --fg: #eaeaea }           /* manual toggle wins if placed later / more specific */
:root[data-theme="light"] { --bg: #ffffff; --fg: #1a1a1a }
body { background: var(--bg); color: var(--fg) }
/* modern: color: light-dark(#1a1a1a, #eaeaea) with color-scheme: light dark */
```
Persist the choice (localStorage), set `data-theme` **before first paint** (inline script in `<head>`) to avoid a flash; also set `color-scheme` so scrollbars/form controls match; check contrast in both themes; avoid pure `#000` on `#fff` glare in dark mode.

---

## 10. Selectors, pseudo-classes, pseudo-elements

```css
a, b          /* list */        a b     /* descendant */      a > b   /* child */
a + b         /* next sibling */ a ~ b  /* later siblings */   [href^="https"] [type="text" i] [data-x~="y"]
:hover :focus :focus-visible :focus-within :active :disabled :checked :required :invalid :placeholder-shown
:first-child :last-child :nth-child(2n+1) :nth-of-type(2) :only-child :empty :root :target
:not(.x)                      /* negation, specificity of its argument */
:is(h1, h2, h3):hover         /* forgiving list; specificity = most specific arg  → shorten repetitive selectors */
:where(.a, .b) p              /* same matching, ZERO specificity → ideal for resets / overridable defaults */
.card:has(img)                /* parent selector: card that contains an img */
label:has(+ input:invalid)    /* relational: label followed by an invalid input */
form:has(:focus-visible)      /* style the container when a descendant is focused */
```
- **`:has()`** (baseline since Dec 2023) enables parent/previous-sibling styling and quantity queries, e.g. `ul:has(li:nth-child(4))`; it can be expensive with broad subjects — keep it scoped.
- **Pseudo-elements** create/style virtual parts: `::before/::after` (need `content`, are inline by default, decorative; not in the a11y tree reliably), `::first-line`, `::first-letter`, `::marker`, `::placeholder`, `::selection`, `::backdrop` (dialog/fullscreen top layer), `::file-selector-button`. Single colon = pseudo-class (state), double colon = pseudo-element (part).
- **Native nesting** (no Sass needed): `.card { padding: 1rem; &:hover { … } .title { … } @media (min-width: 40rem) { … } }`.
```css
.tooltip { position: relative }
.tooltip::after { content: attr(data-tip); position: absolute; bottom: 100%; left: 50%; translate: -50% -4px; opacity: 0; pointer-events: none }
.tooltip:is(:hover, :focus-visible)::after { opacity: 1 }
.clearfix::after { content: ""; display: block; clear: both }          /* legacy float containment; prefer display: flow-root */
```

---

## 11. Transitions, animations, transforms and rendering performance

```css
.btn { transition: transform .2s ease-out, background-color .2s; }           /* list properties, never `all` in hot paths */
.btn:hover { transform: translateY(-2px) scale(1.03) }
@keyframes spin { to { transform: rotate(360deg) } }
.loader { animation: spin 1s linear infinite }
@media (prefers-reduced-motion: reduce) { *, *::before, *::after { animation-duration: .01ms !important; animation-iteration-count: 1 !important; transition-duration: .01ms !important; scroll-behavior: auto !important } }
```
- **transition** = state A → B when a property changes (needs a start and end value; can't transition `display` or `height: auto` — use `grid-template-rows: 0fr → 1fr`, `max-height`, `@starting-style`, `interpolate-size: allow-keywords`); **animation** = multi-step `@keyframes`, can run without a trigger, loops, fill-mode.
- **transform** functions: `translate`, `scale`, `rotate`, `skew`, `matrix`, 3D (`perspective`, `rotateY`); `transform-origin`; individual properties `translate/scale/rotate`; transforms don't affect layout (siblings don't move), create a **stacking context and a containing block for fixed/absolute descendants**. Order matters (`rotate() translate()` ≠ `translate() rotate()`).

**How the browser renders & why `transform/opacity` are cheap**
```
 style → LAYOUT (geometry: width/height/margin/top/left/font-size/flex…)   ← most expensive, may cascade to descendants/siblings
       → PAINT  (pixels: color, background, box-shadow, border-radius, outline)
       → COMPOSITE (GPU combines layers: transform, opacity, filter*, position of an already-painted layer)  ← cheapest; runs on compositor thread
 Animate width/left/top      → layout + paint + composite every frame (jank)
 Animate background/box-shadow → paint + composite
 Animate transform/opacity   → composite only (element gets its own GPU layer; main thread can be busy and the animation still runs smoothly)
```
**Rules:** animate `transform`/`opacity`, not `top/left/width/height/margin`; `will-change: transform` (or `transform: translateZ(0)`) promotes a layer *ahead* of time — use sparingly, set it just before animating, remove after (each layer costs GPU memory; too many layers hurt); avoid **layout thrashing** (interleaved reads like `offsetHeight` and writes in a loop force synchronous reflow); use `contain: content` / `content-visibility: auto` on offscreen sections to skip their layout/paint; keep DOM small; prefer `position: sticky` over scroll handlers; DevTools → Performance panel + "Rendering → Paint flashing / Layout shift regions", Layers panel.

**Cause of reflow:** DOM insert/remove, size/font/margin/`display` changes, reading layout properties, resizing window. **Repaint only:** colour, visibility, shadow, outline.

---

## 12. Accessibility in CSS/HTML

- **Semantic HTML first** (`button`, `nav`, `main`, `header`, `label`, `ul`, `table`, `dialog`, `details`): free keyboard support, roles, and screen-reader semantics. **First rule of ARIA: don't use ARIA if a native element exists.** ARIA basics: `role`, `aria-label/labelledby/describedby`, `aria-expanded/controls/current/live`, `aria-hidden` (never on focusable elements). ARIA changes semantics, not behaviour — you must add keyboard handling.
- **Focus:** never `outline: none` without a visible replacement. Use `:focus-visible` (shows focus for keyboard, not for mouse clicks): `:focus-visible { outline: 3px solid Highlight; outline-offset: 2px }`. Keep DOM order = visual order (`order`/grid `dense` break tab order). Skip link, `scroll-margin-top` under sticky headers, `:focus-within` for composite widgets.
- **Contrast (WCAG 2.2 AA):** text 4.5:1 (large ≥ 24px, or ≥ 18.66px bold: 3:1); UI components, focus rings and icons: 3:1. Don't rely on colour alone (add icon/text/underline). Test both themes.
- **Motion:** honour `prefers-reduced-motion` (see §11), avoid parallax/autoplay; no flashing > 3/s. Also `prefers-contrast`, `forced-colors` (Windows High Contrast; use system colours, keep borders), `prefers-reduced-transparency`.
- **Sizing:** use `rem` so text scales to 200%; don't disable zoom (`user-scalable=no`); reflow at 320px width without 2D scrolling; touch targets ≥ 24×24 CSS px (WCAG 2.2 minimum), ~44px recommended; `line-height ≥ 1.5`.
- **Hiding:** `display:none`/`hidden`/`visibility:hidden` remove from a11y tree; visually-hidden-but-readable pattern:
```css
.sr-only { position: absolute; width: 1px; height: 1px; margin: -1px; padding: 0; overflow: hidden; clip-path: inset(50%); white-space: nowrap; border: 0 }
```
- Forms: visible `<label for>`, error text linked via `aria-describedby`, `:user-invalid` after interaction, `autocomplete` attributes. Images: meaningful `alt` (empty `alt=""` when decorative). Test: keyboard-only pass, screen reader (NVDA/VoiceOver), axe/Lighthouse, zoom 200%.

---

## 13. CSS architecture

| Approach | Idea | Pros | Cons |
|---|---|---|---|
| **BEM** (`.card`, `.card__title`, `.card--featured`) | naming convention: Block__Element--Modifier, flat single-class selectors | zero tooling, low specificity, readable, works anywhere | verbose names; discipline only (no enforcement); global namespace |
| **CSS Modules** (`styles.card` → `Card_card__x7f3`) | build-time hashed class names per file | true local scope, plain CSS, zero runtime, great with React/Vite | dynamic theming needs variables; sharing across files via `composes`/imports; class-name typos silent (without types) |
| **CSS-in-JS** (styled-components, Emotion) | styles in JS, generated `<style>` at runtime | co-location, props-based dynamic styles, auto scoping, dead-code elimination | **runtime cost** (serialise + inject on render), SSR/streaming and React 18 concurrent-rendering friction, bigger bundles, harder caching; libraries in maintenance mode → trend to **zero-runtime** (vanilla-extract, Linaria, Panda, CSS Modules) |
| **Tailwind (utility-first)** | tiny single-purpose classes (`flex gap-4 p-4 md:p-8`), JIT generates only used ones | fast to build, design tokens enforced (spacing/colour scale), no naming, tiny CSS, consistent, great with component frameworks, no dead CSS | verbose markup, learning curve, needs components/`@apply` for reuse, hard to read long class lists, overriding third-party is awkward, specificity conflicts among utilities (`tailwind-merge`) |
| **Sass/Less + native nesting/variables** | preprocessors | mixins, functions | most features now native (vars, nesting) |

Guidelines: keep specificity low and flat (single class selectors), avoid `#id` and `!important`, use `@layer` to order reset → base → components → utilities, design tokens as custom properties, component-scoped CSS + global tokens, colocate styles with components, lint with Stylelint, `prettier`. **Which would I pick?** Component app with a design system: CSS Modules or Tailwind (+ tokens); avoid runtime CSS-in-JS on performance-sensitive SSR sites.

---

## 14. Twenty classic layout tasks (with solutions)

**1. Center a div — 5 ways (horizontal + vertical)**
```css
/* a) Grid (shortest)   */ .p { display: grid; place-items: center; min-height: 100dvh }
/* b) Flex              */ .p { display: flex; justify-content: center; align-items: center }
/* c) Absolute+transform*/ .p { position: relative } .c { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%) }
/* d) Absolute+inset+margin auto (needs a size) */ .c { position: absolute; inset: 0; margin: auto; width: 300px; height: 200px }
/* e) Horizontal only: block with width → margin-inline: auto; inline content → text-align: center; single line vertical → line-height = height */
/* f) grid on parent, margin: auto on child:  .p { display: grid } .c { margin: auto } */
```
*Note:* `translate(-50%,-50%)` on odd pixel sizes can blur text (subpixel); prefer grid/flex.

**2. Sticky footer (footer at bottom on short pages)**
```css
body { min-height: 100dvh; display: grid; grid-template-rows: auto 1fr auto }   /* header / main grows / footer */
/* flex alternative: body { min-height:100dvh; display:flex; flex-direction:column } main { flex:1 } */
```

**3. Holy grail (header, footer, 3 columns: fixed–fluid–fixed; content first in DOM)**
```css
.hg { display: grid; min-height: 100dvh; gap: 1rem;
  grid-template: "header header header" auto "nav main aside" 1fr "footer footer footer" auto / 200px minmax(0, 1fr) 200px }
header { grid-area: header } nav { grid-area: nav } main { grid-area: main } aside { grid-area: aside } footer { grid-area: footer }
@media (max-width: 48rem) { .hg { grid-template: "header" "main" "nav" "aside" "footer" / minmax(0, 1fr) } }
```

**4. Equal-height cards with the button pinned to the bottom**
```css
.cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(min(100%, 260px), 1fr)); gap: 1rem }   /* rows are equal height by default (align-items: stretch) */
.card { display: flex; flex-direction: column }
.card .btn { margin-top: auto }                                                                                /* pushes to the bottom */
```

**5. Responsive navigation (logo left, links right, collapses on mobile)**
```css
.nav { display: flex; align-items: center; gap: 1rem; flex-wrap: wrap }
.nav .logo { margin-right: auto }
.nav ul { display: flex; gap: 1rem; list-style: none; margin: 0; padding: 0 }
@media (max-width: 40rem) { .nav ul { display: none; flex-direction: column; width: 100%; } .nav.open ul { display: flex } }  /* toggle .open + aria-expanded via a <button> */
```

**6. Fixed sidebar + fluid content (collapses on mobile)**
```css
.app { display: grid; grid-template-columns: 260px minmax(0, 1fr); min-height: 100dvh }
.sidebar { position: sticky; top: 0; height: 100dvh; overflow-y: auto }
@media (max-width: 48rem) { .app { grid-template-columns: 1fr } .sidebar { position: fixed; inset: 0 auto 0 0; width: 260px; translate: -100%; transition: translate .2s } .sidebar.open { translate: 0 } }
```

**7. Masonry-ish (variable-height cards, no JS)**
```css
.masonry { columns: 3 240px; column-gap: 1rem }                       /* CSS multi-column: 3 columns, min 240px each */
.masonry > * { break-inside: avoid; margin-bottom: 1rem }
/* order runs top-to-bottom per column (not left-to-right). Native `grid-template-rows: masonry` is not broadly shipped — use JS/library if row order matters. */
```

**8. Truncate a single line with an ellipsis**
```css
.t { overflow: hidden; text-overflow: ellipsis; white-space: nowrap }    /* needs a constrained width; inside flex items add min-width: 0 */
```

**9. Multi-line clamp (3 lines)**
```css
.clamp { display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 3; line-clamp: 3; overflow: hidden }
```
(The `-webkit-` form is what all browsers support today; the standard `line-clamp` is catching up.)

**10. Aspect-ratio boxes (16:9 video / square avatar)**
```css
.video { aspect-ratio: 16 / 9; width: 100% } .video > iframe { width: 100%; height: 100% }
.avatar { aspect-ratio: 1; border-radius: 50%; object-fit: cover; width: 4rem }
/* legacy: .box { position: relative; padding-top: 56.25% } .box > * { position: absolute; inset: 0 }  (padding % is relative to WIDTH) */
```

**11. Overlay modal**
```css
.backdrop { position: fixed; inset: 0; background: rgb(0 0 0 / .5); display: grid; place-items: center; z-index: 1000 }
.modal { width: min(90vw, 32rem); max-height: 85dvh; overflow: auto; background: #fff; border-radius: .75rem; padding: 1.5rem }
body.modal-open { overflow: hidden }                 /* scroll lock */
```
Prefer native `<dialog>` + `dialog.showModal()` (top layer: no z-index fights, `::backdrop`, focus trapping, Esc to close) and restore focus on close; add `aria-labelledby`.

**12. Sticky header without hiding anchor targets**
```css
header { position: sticky; top: 0; z-index: 50 } html { scroll-padding-top: 4rem; scroll-behavior: smooth } :target { scroll-margin-top: 4rem }
```

**13. Full-bleed image/hero with text overlay**
```css
.hero { position: relative; min-height: 60dvh; display: grid; align-items: end }
.hero > img { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; z-index: -1 }   /* or grid-area: 1/1 on both children to stack without absolute */
```

**14. Form with aligned labels (2-column grid)**
```css
.form { display: grid; grid-template-columns: max-content 1fr; gap: .75rem 1rem; align-items: center }
.form .full { grid-column: 1 / -1 } @media (max-width: 40rem) { .form { grid-template-columns: 1fr } }
```

**15. Responsive card grid without a single media query** → task 4's `repeat(auto-fit, minmax(min(100%, 260px), 1fr))`.

**16. Tooltip / badge anchored to an element** → §10 tooltip (`position: relative` parent, `::after` absolute, `pointer-events: none`).

**17. Horizontal scroll carousel with snap**
```css
.rail { display: grid; grid-auto-flow: column; grid-auto-columns: 80%; gap: 1rem; overflow-x: auto; scroll-snap-type: x mandatory; overscroll-behavior-x: contain }
.rail > * { scroll-snap-align: start }
```

**18. Centered container with a full-bleed child**
```css
.wrap { display: grid; grid-template-columns: 1fr min(70rem, 100% - 2rem) 1fr } .wrap > * { grid-column: 2 } .wrap > .bleed { grid-column: 1 / -1 }
```

**19. Fluid typography & spacing scale**
```css
:root { --step-0: clamp(1rem, .95rem + .3vw, 1.125rem); --step-2: clamp(1.5rem, 1.2rem + 1.5vw, 2.25rem); --gap: clamp(1rem, 2vw, 2rem) } h1 { font-size: var(--step-2); max-width: 20ch; text-wrap: balance }
```

**20. CSS-only accordion / toggle**
```html
<details><summary>Shipping</summary><p>Ships in 2 days.</p></details>   <!-- native, keyboard + a11y for free -->
```
```css
.acc { display: grid; grid-template-rows: 0fr; transition: grid-template-rows .25s } .acc.open { grid-template-rows: 1fr } .acc > div { overflow: hidden }   /* animates to height:auto */
.switch input { position: absolute; opacity: 0 } .switch span { transition: background .2s } .switch input:checked + span { background: #2e7d32 } .switch input:focus-visible + span { outline: 3px solid Highlight }
```
Also common: **holy-grail alt with flex**, **sticky table header** (`th { position: sticky; top: 0 }` — needs a scroll container with fixed height), **equal-gap stack** (`.stack > * + * { margin-block-start: 1rem }` or `display:grid; gap`), **image gallery** (grid `grid-auto-flow: dense` with `.wide { grid-column: span 2 }`).

---

## 15. Common bugs and fixes (production checklist)

| Symptom | Cause | Fix |
|---|---|---|
| `height: 100%` does nothing | parent height is `auto` (not definite) | give parents height (`html, body { height: 100% }`) or use flex/grid `min-height: 100dvh` |
| `z-index: 9999` still hidden | ancestor created a stacking context (transform/opacity/filter/z-index) | move element (portal/`<dialog>`), remove the context, `isolation: isolate` |
| Text overflows a flex/grid child | `min-width: auto` | `min-width: 0` + `overflow: hidden` / `overflow-wrap: anywhere`; `minmax(0, 1fr)` |
| Margin "disappears" / parent moves | margin collapsing | `display: flow-root`, padding/border, or `gap` |
| `position: sticky` doesn't stick | ancestor has `overflow: hidden/auto`, no `top`, or parent too short | remove overflow (or `overflow: clip`), set `top`, check the parent's height |
| Horizontal scrollbar on mobile | `100vw` (includes scrollbar), fixed widths, long words, images without max-width | `width: 100%`, `max-width: 100%`, `overflow-wrap: anywhere`, find culprit with `* { outline: 1px solid red }` |
| Gap under an `<img>` | inline → baseline space for descenders | `display: block` (or `vertical-align: middle`) |
| Mobile bottom bar hidden by browser UI | `100vh` | `100dvh`/`svh`, `env(safe-area-inset-bottom)` for notch |
| `overflow: hidden` clips shadows/focus outlines | clipping at padding box | add padding, use `overflow: clip` + `overflow-clip-margin`, or move overflow to an inner wrapper |
| `position: fixed` element behaves like absolute | ancestor has `transform/filter` | remove it from that ancestor or portal out |
| Transition doesn't run | animating `display`, `height: auto`, or class added in same frame as element insertion | animate `opacity/transform`, `grid-template-rows`, `@starting-style`, force reflow |
| Styles not applied | specificity/order/layer, typo, invalid value dropped silently, `:hover` on touch | DevTools → Styles (struck-through rules), Computed → "Show all", check `@layer`, use `@media (hover: hover)` |
| iOS zooms on input focus | input `font-size < 16px` | set `font-size: 16px` on inputs |
| Blurry text after centering | `translate(-50%,-50%)` lands on half pixels | use grid/flex centering or `will-change: transform` sparingly |
| Grid blows out its container | `1fr` = `minmax(auto,1fr)`, wide child | `minmax(0,1fr)`, `min-width: 0` |
| Flex item won't shrink / image squashed | `flex-shrink` on fixed-size media | `flex: none` on the image, or `min-width` |
| Layout shifts (CLS) | images/fonts/ads without reserved size | `width/height` attrs, `aspect-ratio`, `font-display: optional/swap` + size-adjust, reserve slots |
| `!important` wars | competing specificity | `@layer`, flatten selectors, BEM, remove nested selectors |
| Dark mode flash | theme applied after first paint | inline script in `<head>` setting `data-theme`, `color-scheme` |
| Unwanted scroll chaining / bounce | modal scrolls the page underneath | `overscroll-behavior: contain`, `body { overflow: hidden }` while open |
| Fonts jump (FOUT/FOIT) | web font swap | `font-display: swap`, preload, `size-adjust`/fallback metrics |

---

## 16. Interview questions (34) — graded, with follow-up chains and common wrong answers

**Legend:** [E] easy, [M] medium, [H] hard · *FU* = follow-up chain · *Wrong* = common mistakes.

**1. [E] What is the box model? `content-box` vs `border-box`?** Every element is content + padding + border + margin. `content-box` (default): `width` = content only, so 200px + 20px padding + 5px border = 250px on screen; `border-box`: `width` includes padding and border (total stays 200px). Set `*, *::before, *::after { box-sizing: border-box }`. *FU:* which part shows the background (padding + content, up to border box by default; `background-clip`) → do margins count in width? (no) → what does `%` padding refer to? (containing block's width). *Wrong:* "margin is part of the width".

**2. [E] How is specificity calculated?** (inline, ids, classes/attrs/pseudo-classes, elements/pseudo-elements) compared left to right; ties → later wins; `!important` beats normal. `#a` (1,0,0) > `.a.b.c` (0,3,0) > `div p` (0,0,2). *FU:* specificity of `:is(#a, .b)` (1,0,0) and `:where()` (0) → what do `*` and combinators add (0) → how would you override a third-party rule without `!important` (`@layer`, equal specificity later in source, `:where` in your own defaults). *Wrong:* "10 classes beat 1 id".

**3. [M] Explain the cascade fully.** Origin/importance → `@layer` → specificity → order. Author `!important` beats normal author; for important declarations layer order reverses; unlayered beats layered (normal). *FU:* where do inline styles fit (highest specificity, but `!important` in a stylesheet beats them) → what are transitions/animations in the order → `revert-layer`.

**4. [E] Which properties inherit?** Mostly text ones (`color`, `font`, `line-height`, `text-align`, `visibility`, `cursor`, list styles); box properties don't. Use `inherit`/`initial`/`unset`. *FU:* why does a `<button>` not inherit the page font (UA sets its own; use `font: inherit`) → what does `all: unset` do.

**5. [M] Explain margin collapsing and how to stop it.** §3. *FU:* do flex items collapse (no) → parent/child collapse case → difference between BFC via `overflow:hidden` vs `flow-root` (side effects: clipping). *Wrong:* "margins are added".

**6. [M] `display: none` vs `visibility: hidden` vs `opacity: 0` vs `hidden` attribute.** `none`: no box, not in a11y tree; `visibility:hidden`: keeps space, not focusable/not read; `opacity: 0`: invisible but still interactive/focusable/read (and can be animated); `hidden` attribute = UA `display:none`. *FU:* which animates (opacity/visibility with delay) → hidden but screen-reader accessible (`.sr-only`).

**7. [M] Block vs inline vs inline-block?** §3 table (inline ignores width/height/vertical margins; inline-block honours them; whitespace gap issue). *FU:* why is there a gap under images → why can't I centre an inline `<a>` vertically with margin → replaced elements (`img`) are inline but accept width/height.

**8. [M] Explain `position` values and containing blocks.** §4. `absolute` uses nearest positioned ancestor (or one with `transform/filter/contain`), `fixed` uses the viewport (same exception), `sticky` = relative + fixed within scroll container/parent. *FU:* why isn't my absolute child positioned relative to the parent (parent is `static`) → `sticky` not working (overflow ancestor) → percentages for absolute `top`/`height`.

**9. [H] `z-index` isn't working — why? What is a stacking context?** §4: only on positioned/flex/grid items; a stacking context traps children; created by opacity/transform/filter/`isolation`/etc.; compare contexts not raw numbers. *FU:* how to design a z-index scale → why does `<dialog>` avoid the problem (top layer) → does `z-index: -1` go behind parent background (only if parent isn't a stacking context).

**10. [M] Centre a div — as many ways as you know.** §14 task 1. *FU:* centre with unknown size (grid/flex/translate) → centre text vertically in a button (flex/`line-height`, `padding`) → centre in the viewport with `dvh`. *Wrong:* only `margin: 0 auto` (horizontal only, needs width).

**11. [M] Flexbox: axes, `justify-content` vs `align-items` vs `align-content`.** §5. `align-content` only distributes multiple wrapped lines. *FU:* what does `flex: 1` expand to (`1 1 0%`) and how does it differ from `flex: auto`/`flex: none` → what changes with `flex-direction: column` (axes swap) → `gap` vs margins.

**12. [H] Explain how `flex-grow/shrink/basis` are resolved — compute an example.** Use §5 examples: shrink 500px/200,200,200 with shrink 1,1,2 → 175,175,150; grow 600/100 each with 1:2:3 → 150,200,250; frozen item on max-width → 190,120,190. State the loop: hypothetical size → free space sign → distribute (grow by factor; shrink by factor×basis) → freeze violators → repeat. *FU:* why do items shrink below `width` (default `flex-shrink: 1`) → what stops shrinking (`min-width: auto`, `flex-shrink: 0`) → why is `flex-basis: 0` used in `flex: 1`.

**13. [M] Why does long text overflow my flex/grid child, and how do you fix it?** `min-width: auto` (min-content) → `min-width: 0` (+ `overflow`) / `minmax(0,1fr)`. *FU:* same for `min-height` in column flex scroll areas (chat layouts: parent `min-height: 0` so inner list can scroll).

**14. [M] Flex vs Grid — when do you use which?** §7. *FU:* build a dashboard shell (grid areas) with a toolbar inside (flex) → why not fake columns with `float`/percent widths → `subgrid` use case.

**15. [M] `auto-fit` vs `auto-fill`; what does `minmax(0, 1fr)` solve?** §6. *FU:* what does `fr` mean exactly (share of remaining free space) → `1fr` vs `auto` → `min(100%, 240px)` inside `minmax` (prevents overflow).

**16. [M] Mobile-first vs desktop-first; how do you pick breakpoints?** Base = mobile, `min-width` queries add complexity; content-driven breakpoints; `rem`-based queries; container queries for components. *FU:* what's `viewport` meta for → `hover: hover` media feature → why do `max-width` queries cause overrides.

**17. [M] Explain container queries and when you'd use them over media queries.** Component adapts to its container (`container-type: inline-size` + `@container`); needed for reusable widgets placed in sidebar/main/modal; media queries only know the viewport. *FU:* why can't the container query its own size (no cyclic dependency; query applies to descendants) → container units.

**18. [M] `rem` vs `em` vs `px` vs `%` vs `vw` — when?** §8 table. *FU:* why `rem` for font-size (respects user default) → why `em` compounds → what's wrong with pure `vw` fonts (zoom accessibility; use `clamp` with rem).

**19. [M] Why is `100vh` a problem on mobile? Fix?** Large viewport vs visible area with the URL bar; use `dvh`/`svh`, fallback chain, `env(safe-area-inset-*)`. *FU:* difference between `svh/lvh/dvh` → why avoid `dvh` for animated hero (resizes on scroll → jank).

**20. [M] `srcset`/`sizes` vs `<picture>`; how do you optimise images?** §8. Formats (AVIF/WebP), responsive sizes, lazy loading, dimensions to avoid CLS, `fetchpriority="high"` for LCP image, CDN. *FU:* how does the browser choose (width descriptors + `sizes` + DPR) → `object-fit`.

**21. [M] CSS custom properties vs Sass variables?** Runtime, cascade/inherit, scoped, JS-readable/writable, usable in media/container? (**not inside media query conditions**), themable; Sass are compile-time only. *FU:* how do you implement dark mode → `@property` typed vars → fallback `var(--x, fallback)` → cost of many vars on `:root` (style recalc on change).

**22. [M] How do you implement dark mode properly?** §9: tokens, `prefers-color-scheme`, `data-theme` override, `color-scheme`, no-flash inline script, persisted choice, contrast tested. *FU:* images/shadows in dark mode → `light-dark()`.

**23. [M] `:is()`, `:where()`, `:has()`, `:not()` — differences and uses.** §10: specificity behaviours; `:has` = parent selector; `:where` for zero-specificity defaults. *FU:* performance of `:has` → forgiving selector lists → `:focus-visible` vs `:focus`.

**24. [M] `::before/::after` — how do they work; a11y concerns?** Generated children needing `content`, inline by default, not selectable/in DOM; decorative only — content in pseudo-elements may or may not be announced and can't be translated/copied; icons via CSS `mask` or inline SVG. *FU:* `attr()`, counters (`counter-increment`), clearfix.

**25. [M] Which CSS properties are cheap to animate and why?** `transform` and `opacity` run on the compositor; layout properties trigger reflow → paint → composite. §11 diagram. *FU:* what does `will-change` do and misuse → why does a transformed element create a stacking context → animate `height: auto` alternatives → `prefers-reduced-motion`.

**26. [H] Explain reflow vs repaint vs composite; what causes layout thrashing?** §11: reading layout after writing in a loop forces sync layout; batch reads/writes, use `requestAnimationFrame`, `ResizeObserver`, `contain`/`content-visibility`. *FU:* how do you find it (DevTools Performance: purple "Layout" blocks, forced reflow warnings) → virtualisation for long lists.

**27. [M] Transition vs animation vs transform?** State changes with implicit start/end vs keyframes vs geometry function. *FU:* why doesn't `transition: display .3s` work → `transition: all` downside → `animation-fill-mode`.

**28. [M] CSS accessibility essentials.** Semantic HTML, focus-visible, contrast 4.5:1/3:1, reduced motion, zoom/rem, target size, don't hide focus, ARIA only when needed, visually-hidden utility. §12. *FU:* `display:none` vs `.sr-only` → how to test → `outline: none` fix.

**29. [M] BEM vs CSS Modules vs CSS-in-JS vs Tailwind — trade-offs, and what would you choose?** §13 table; state a decision framework: team size, design system, SSR/perf constraints, tooling. *FU:* runtime cost of styled-components → how does Tailwind avoid huge CSS (JIT scanning) → how do you override third-party component styles (`@layer`, CSS variables, `::part()`).

**30. [H] "Implement this layout": header, sidebar, main, footer, sticky footer, responsive.** Answer with grid areas (§6/§14 tasks 2, 3, 6): `min-height: 100dvh`, `grid-template-rows: auto 1fr auto`, `minmax(0,1fr)`, collapse via media/container query, sticky header with `scroll-padding-top`. *FU:* what if content is dynamic/long (min-width issues, inner scroll `min-height: 0`) → keyboard/tab order with `order` or grid placement.

**31. [M] `position: sticky` — how does it work and why might it fail?** §4. *FU:* sticky table headers (`th` sticky + scroll container height) → sticky inside flex/grid (`align-self: start` needed, otherwise the stretched item has no room to stick).

**32. [M] How do you truncate text (single/multi-line) and what are the gotchas?** §14 tasks 8–9; requires constrained width; flex item needs `min-width: 0`; `-webkit-line-clamp` needs `display: -webkit-box`; add `title`/tooltip since text is lost. *FU:* RTL/`text-overflow` with `direction`.

**33. [H] Your CSS is huge and slow — how do you diagnose and improve?** Coverage tab (unused CSS), critical CSS inline + defer rest (`media="print" onload` pattern / `preload`), split per route, purge with Tailwind/PurgeCSS, avoid expensive selectors (deep descendant, broad `:has`), reduce paint area (`contain`), avoid large blur/box-shadow animations, compress (brotli), cache with hashed names, `content-visibility: auto`. Measure with Lighthouse/Performance panel. *FU:* render-blocking CSS and `media` attribute trick → font loading strategy.

**34. [H] Debug: "This element is not styled/positioned as expected."** Method: inspect element → Styles pane (crossed-out rules, specificity, layer) → Computed (final values, box model diagram) → check containing block/stacking context ancestors (transform, overflow, position) → toggle `outline: 1px solid red` on `*` → check display type of parent (flex/grid item semantics: `float`/`vertical-align` ignored, margins don't collapse) → check invalid value (dropped declaration) and media/container query conditions → test in another browser. *FU:* how do you prevent regressions (Stylelint, visual regression tests with Playwright, Storybook).

---

## 17. One-page cheat sheet

```
CASCADE     importance → @layer (unlayered wins; !important reverses) → specificity (id,class,el) → order   ; inline=1,0,0,0 ; :where=0 ; :is/:not/:has = max arg
INHERIT     text props inherit (color,font,line-height,text-align,visibility) ; box props don't ; inherit|initial|unset|revert
BOX         *{box-sizing:border-box} ; % padding = width of containing block ; height:% needs definite parent ; margins collapse vertically (siblings, parent/child, empty) except flex/grid/BFC
DISPLAY     block | inline (no w/h/vertical margin) | inline-block | flex | grid | flow-root (BFC) | contents | none (removes a11y) ; visibility:hidden keeps space
POSITION    relative(anchor) absolute(nearest positioned; transform/filter also anchor) fixed(viewport) sticky(needs top; overflow ancestor kills it)
STACKING    z-index needs positioned/flex/grid item ; contexts: opacity<1 transform filter isolation fixed/sticky z-index≠auto ; compare contexts, not numbers
FLEX        justify=main align-items=cross align-content=lines(wrap only) ; flex:1 = 1 1 0% ; auto=1 1 auto ; none=0 0 auto
            grow: share free space by grow ; shrink: by shrink×basis ; min-width:auto trap → min-width:0
GRID        fr = share of leftover ; minmax(0,1fr) ; repeat(auto-fit, minmax(min(100%,240px),1fr)) ; auto-fit collapses empty tracks, auto-fill keeps ; areas ; subgrid ; dense reorders visually
FLEX v GRID content-out vs layout-in ; grid for page/2D, flex for components/1D
RESPONSIVE  viewport meta ; mobile-first min-width ; container queries ; clamp(min, rem+vw, max) ; rem for type ; ch for measure ; dvh/svh not vh ; srcset+sizes / picture ; aspect-ratio ; object-fit
VARS        --x cascades & inherits ; var(--x, fallback) ; dark mode: prefers-color-scheme + [data-theme] + color-scheme + no-flash script
SELECTORS   :is :where :not :has :focus-visible :focus-within ; ::before/::after need content ; native nesting &
PERF        layout > paint > composite ; animate transform/opacity ; will-change sparingly ; avoid read/write thrash ; contain/content-visibility ; critical CSS
A11Y        semantic HTML ; focus-visible ; contrast 4.5:1 (3:1 large/UI) ; prefers-reduced-motion ; rem+zoom ; targets ≥24px ; ARIA last resort
ARCH        BEM (flat classes) | CSS Modules (build-scoped) | CSS-in-JS (runtime cost → zero-runtime) | Tailwind (utility, tokens, verbose) ; @layer to order ; low specificity ; no !important
BUG TOP-8   height:100% ; z-index trapped ; flex overflow (min-width:0) ; margin collapse ; sticky+overflow ; 100vw scrollbar ; img baseline gap ; 100vh mobile
```

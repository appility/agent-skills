# CSS and progressive enhancement

CSS degrades well by design: a browser ignores declarations and at-rules it doesn't understand. Use that on purpose.

## Fallbacks in the cascade

Write the widely supported value first, the newer one after. Browsers that don't understand the second keep the first.

```css
.card {
  width: 90%;
  width: min(90%, 40rem);
  background: #f4f1ea;
  background: oklch(95% 0.02 85);
}
```

## Feature queries

Use `@supports` when a whole block depends on a feature, so the baseline styles stay clean.

```css
/* Baseline: single column, works everywhere */
.gallery > * + * { margin-top: 1rem; }

@supports (display: grid) {
  .gallery { display: grid; grid-template-columns: repeat(auto-fill, minmax(14rem, 1fr)); gap: 1rem; }
  .gallery > * + * { margin-top: 0; }
}

@supports selector(:has(input:checked)) {
  .option:has(input:checked) { outline: 2px solid currentColor; }
}
```

In JS, `CSS.supports('display', 'grid')` answers the same question.

Layout: build mobile-first, single column by default. Grid, subgrid and container queries enhance from there; a browser without them still gets a readable column.

## Styling for "JS is running"

Styles that only make sense once a script has wired something up (collapsed panels, hidden fallback buttons, ARIA tabs) need a guard, or people without the script get hidden content with no way to reveal it.

**Best: a class on the component, set when it is wired.** The script adds the class in the same step that attaches its handlers, so the styles can never apply without the behaviour:

```js
tabs.classList.add('is-enhanced'); // after the keyboard handlers are attached
```

```css
.tabs.is-enhanced [role="tabpanel"][hidden] { display: none; }
```

**Weaker: a page-wide flag.** An inline script in `<head>` swaps `no-js` for `js` before the page paints. It only proves JavaScript runs, not that your component's code loaded without errors.

```html
<html class="no-js">
<head>
  <script>document.documentElement.classList.replace('no-js', 'js');</script>
```

**Media query, for hiding JS-only extras.** `@media (scripting: none)` (current browsers) matches when scripting is off, so it's a safe way to hide controls that can't work without it. Don't use `scripting: enabled` to hide content: it is true even when your bundle fails to load.

```css
@media (scripting: none) {
  .needs-js { display: none; }
}
```

## Native behaviour before custom

Reach for elements and attributes that need no JS before styling a div into a widget:

- `<details>`/`<summary>` for disclosure; same `name` on several for an exclusive accordion
- `<dialog>` for modals; `popover` + `popovertarget` for menus and toggletips
- `:focus-visible` for focus rings, `accent-color` for checkboxes and radios
- `scroll-snap-type` for carousels, `scroll-behavior: smooth` (inside a reduced-motion guard)

## Motion

Put animation behind a preference check so it's an enhancement, not a default:

```css
@media (prefers-reduced-motion: no-preference) {
  .panel { transition: transform 200ms ease; }
}
```

View transitions are a progressive enhancement by nature: browsers without them navigate instantly. Cross-document transitions (`@view-transition { navigation: auto; }`) work only where supported, so never rely on them to explain a change.

## Fonts

- `font-display: swap` so text shows in a fallback font while the web font loads.
- A sensible system stack after the web font.
- `size-adjust` and `ascent-override` on an `@font-face` fallback to reduce layout shift when the web font arrives.

## Don't

- Hide content with CSS that only JS can undo.
- Use `display: none` on a no-JS fallback (submit button, link, `<noscript>` alternative) unless scoped to a JS-is-running hook.
- Convey state only through colour or animation.

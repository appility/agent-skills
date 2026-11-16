# Patterns

Each entry: **Baseline** (server HTML, no JS) → **Enhance** (what JS adds) → **Watch for** (common mistakes).

## Contents

1. Forms and mutations
2. Validation
3. Controls that load data on change
4. Search and filters
5. Pagination, load more, infinite scroll
6. Navigation and client-side routing
7. Disclosure and accordions
8. Tabs
9. Dialogs and modals
10. Menus and site navigation
11. Toggles and one-click actions
12. Clipboard, share and other API buttons
13. Inputs: dates, files, autocomplete, rich text
14. Images and media
15. Carousels

---

## 1. Forms and mutations

**Baseline.** `<form method="post" action="/items">` with named fields and a submit button. The server validates, writes, then redirects (POST → redirect → GET) so a refresh doesn't resubmit. On error it re-renders the form with the user's values, an error summary at the top linking to each field, and messages beside the fields. Prefix the page `<title>` with "Error:" so the problem is announced on load.

**Enhance.** Intercept `submit`, send `new FormData(form)` to the same `action`, show a pending state on the button. On success, update in place or navigate client-side. On error, render the same messages and move focus to the error summary.

**Watch for.**
- Disabling the submit button in the server HTML "until JS loads".
- Fields without `name` attributes, so `FormData` and the no-JS POST both send nothing.
- Double submits: block resubmission while pending (set `aria-disabled="true"` and ignore clicks, rather than `disabled`, so the button keeps focus).
- Optimistic UI that never rolls back. If the server rejects the change, revert it and announce why.

## 2. Validation

**Baseline.** Native constraints (`required`, `type="email"`, `pattern`, `min`, `max`, `maxlength`) plus server validation, always. The browser blocks obvious mistakes with no JS; the server catches everything else.

**Enhance.** If you replace the browser's built-in messages with your own inline errors, add `novalidate` to the form from JS, at the moment your validation is wired up. Validate on blur and on submit, not on every keystroke. Connect messages with `aria-describedby` and set `aria-invalid="true"` on failing fields.

**Watch for.**
- `novalidate` written into the server HTML, which throws away the no-JS validation.
- Client rules the server doesn't also enforce.
- Errors shown only by colour or only on hover.

## 3. Controls that load data on change

Dependent or cascading selects, filter dropdowns, anything where changing one control fetches data for another.

**Baseline.** A `method="get"` form with a visible submit button ("Continue", "Apply"). The server renders the dependent options from the query string.

**Enhance.** On `change`, fetch the same data in the background, replace the dependent options in place, and announce the result in a `role="status"` region. Hide the submit button once this is wired.

**Watch for.** `onChange` that reloads the page or moves focus, which fails WCAG 3.2.2 On Input. One that adds a history entry per change isn't a WCAG failure, but it breaks Back. Full detail, pitfalls and test steps: `on-input.md`.

## 4. Search and filters

**Baseline.** `<form method="get" role="search">` with a labelled input and a submit button. Results are rendered by the server from the query string, so searches are bookmarkable and shareable.

**Enhance.** Debounce input (about 250–300 ms), fetch results from the same route, swap the results list, update the URL with `history.replaceState` (not `pushState` per keystroke, or Back becomes useless), and announce the count in a live region ("12 results for 'oak'"). Leave the submit button or keep Enter working.

**Watch for.**
- Stale responses arriving out of order. Abort the previous request.
- Moving focus out of the search box when results update.
- Filtering that only exists in client state, so a reload or shared link loses it.

## 5. Pagination, load more, infinite scroll

**Baseline.** `<a href="?page=2" rel="next">Next page</a>`. Page 2 is a real server-rendered page.

**Enhance.** Intercept the link, fetch page 2, append the items, update the link to page 3, and update the URL. Move focus to the first new item (give it `tabindex="-1"`) or announce "20 more items loaded". Infinite scroll, if you want it at all, sits on top of this: detect `IntersectionObserver` and keep the link as the trigger and fallback.

**Watch for.**
- Infinite scroll with no link: keyboard users can never reach the footer, and no-JS users see page 1 forever.
- Back button losing the user's place. Put the page in the URL so the server can render up to it, or restore scroll.

## 6. Navigation and client-side routing

**Baseline.** Plain `<a href>` links. Every route renders on the server.

**Enhance.** A client router intercepts clicks for faster transitions.

**Watch for.**
- Navigation built from `onClick` + `navigate()` instead of anchors. It breaks open-in-new-tab, modifier clicks, middle-click, copy link and no-JS.
- Silent route changes. After a client transition, update `document.title`, move focus to the new page's `<h1>` or `<main>` (with `tabindex="-1"`), and let screen readers know the page changed. Some routers do this; many don't.
- Scroll not reset on new pages, or not restored on Back.

## 7. Disclosure and accordions

**Baseline.** `<details><summary>Question</summary>Answer</details>`. Works with no JS and is keyboard and screen reader friendly. For "only one open at a time", give each `<details>` the same `name` attribute (exclusive accordions, supported in current browsers; older ones simply allow several open).

**Enhance.** Usually nothing. Animate opening with CSS under `@supports` if you want.

**Watch for.** Custom accordions whose panels are `display: none` in CSS and opened only by JS. Without JS the content is unreachable.

## 8. Tabs

**Baseline.** A short list of in-page links (`<a href="#billing">`) followed by each section with its heading. Everything is visible and linkable.

**Enhance.** JS turns it into ARIA tabs: `role="tablist"`, `role="tab"` with `aria-selected` and `aria-controls`, `role="tabpanel"`, arrow-key movement with a roving `tabindex`, and inactive panels hidden. Optionally sync the active tab to the URL hash.

**Watch for.** Shipping `role="tab"` in the server HTML before JS adds the keyboard behaviour. The roles promise interactions that don't exist yet. Add roles and hide panels in the same step as wiring the keys.

## 9. Dialogs and modals

**Baseline.** A link to a real page that holds the content, such as `/items/42/delete` with a confirmation form. Or show the content inline.

**Enhance.** Intercept the link, load the same content, and show it in a `<dialog>` opened with `showModal()`. That gives you a focus trap, Escape to close and an inert background for free. Close with `<form method="dialog">` and return focus to the element that opened it.

**Watch for.**
- Content or actions that exist only inside a JS modal.
- Hand-rolled modals that don't trap focus or restore it.
- Popovers and non-modal overlays: the `popover` attribute with `popovertarget` on a button opens and closes them with no JS in current browsers. Provide the content inline for older ones.

## 10. Menus and site navigation

**Baseline.** The navigation links are in the HTML. On small screens, put them inside `<details><summary>Menu</summary>…</details>` or simply let the list show.

**Enhance.** A toggle button with `aria-expanded` and `aria-controls`, or the `popover` attribute. Close on Escape and return focus to the button.

**Watch for.** A mobile menu hidden by CSS with only a JS button to open it. Hide the links only under a class the toggle's script sets once it is wired (see `css.md`).

## 11. Toggles and one-click actions

Like, follow, star, archive, delete, mark as read.

**Baseline.** A tiny form: `<form method="post" action="/posts/42/like"><button>Like</button></form>`. The server makes the change and redirects back to the page (with an anchor such as `#post-42` so the user lands where they were). For destructive actions, the baseline is a confirmation page or an undo.

**Enhance.** Submit in the background, flip the state optimistically (`aria-pressed` on a toggle button), roll back and announce if the server refuses.

**Watch for.** Using a GET link to change data. Link prefetchers, crawlers and browser preloading will trigger it.

## 12. Clipboard, share and other API buttons

**Baseline.** Show the thing itself: the URL or code in a read-only input the user can select, or a normal link (`mailto:`, a share URL).

**Enhance.** Render a Copy button only if `navigator.clipboard` exists (it also needs a secure context). Render a Share button only if `navigator.share` exists. Announce "Copied" in a live region.

**Watch for.** Buttons that silently do nothing in browsers without the API, or on `http:` pages.

## 13. Inputs

- **Dates and times.** `<input type="date">` / `type="time"` gives a native picker and keyboard entry. A custom picker is an enhancement; still let people type the date.
- **File upload.** `<input type="file" name="…">` in a `multipart/form-data` form. Drag and drop, previews and progress bars are enhancements. The drop zone still contains the real, labelled input.
- **Autocomplete.** Start with a plain input, a `<select>`, or `<input list>` with a `<datalist>`. Enhance to an ARIA combobox only if you need richer behaviour.
- **Rich text.** A `<textarea>` (Markdown is fine) as the baseline; the editor enhances it and keeps the textarea's `name` in sync so the form still posts.

## 14. Images and media

- `<img>` with `alt`, `width` and `height` (no layout shift), `srcset`/`sizes`, and `loading="lazy"` for below-the-fold images. No JS needed.
- `<picture>` with `<source type="image/avif">` and `type="image/webp"` before a JPEG `<img>` for format fallbacks.
- Avoid `data-src` lazy-loading that needs JS to swap in the real image. If a library insists, include a `<noscript>` copy.
- `<video controls>` with a `poster`; custom controls are an enhancement. Respect `prefers-reduced-motion` for autoplay.

## 15. Carousels

**Baseline.** A horizontal list with `overflow-x: auto` and CSS scroll snap. Every item is in the HTML and reachable by scrolling or tabbing.

**Enhance.** Previous/next buttons (rendered by JS), position indicators, and pause controls for any autoplay. No autoplay under `prefers-reduced-motion: reduce`.

**Watch for.** Slides hidden with `display: none` until JS positions them.

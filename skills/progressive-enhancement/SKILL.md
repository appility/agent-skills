---
name: progressive-enhancement
description: Build or review server-rendered web UI so it works from the HTML alone and JavaScript only enhances it. Use for forms, dependent selects, filters and search, pagination, modals, tabs, menus, like/follow buttons, hydration bugs ("button does nothing until the page loads"), no-JS fallbacks, and accessibility reviews of interactive components on public or content pages — even if nobody says "progressive enhancement".
---

# Progressive enhancement

## The idea

Start with HTML from the server that lets someone finish the task. Add CSS to make it look right, with fallbacks where support is uneven. Add JavaScript to make it faster or smoother. Each layer is optional to the one beneath it.

This is not only for people who switch JavaScript off:

- **Scripts fail.** Flaky mobile networks, a CDN outage, an ad blocker or corporate proxy, a runtime error in an unrelated bundle.
- **Scripts arrive late.** On slow devices the page is visible and clickable seconds before hydration. Anything that needs JS is dead in that window, and clicks in it are lost.
- **Accessibility comes free with native elements.** Links, buttons, forms, `<details>`, `<dialog>` and `popover` bring keyboard and screen reader behaviour with them. Rebuilding them in JS is where most accessibility bugs come from.
- **The server stays the source of truth.** Data and rules live in one place; the client is an optimisation.

## When it's not worth it

Progressive enhancement is a trade-off, not a rule. Weigh it before applying it, and say which way you went.

**Go all in** for anything public or task-critical: marketing and content pages, search and listings, sign-up and checkout, forms people must complete (applications, bookings, government or health services), anything used on phones over mobile data.

**Do less** where JavaScript is the product or the audience is known:

- **Apps that can't exist without script.** Canvas and design editors, maps, real-time collaboration, games, audio and video tools. Don't build a fake no-JS version. Server-render a clear message and whatever can work (the document title, a read-only view, an export link), and fail visibly rather than showing a dead UI.
- **Internal tools behind a login** on managed machines. The no-JS baseline matters less; native elements, accessible markup and not losing clicks before hydration still matter.
- **Offline-first or local-first apps.** The service worker and client store are the baseline; the server HTML is the shell.
- **Prototypes.** Note the gap and move on.

Some parts are cheap enough to keep everywhere, because they cost almost nothing and fix real bugs: real links and buttons instead of clickable divs, forms with `name` attributes, native `<details>`/`<dialog>`, no `onChange` that navigates, a live region for in-place updates, and never hiding content that only JS can reveal.

When the user's context is unclear, ask what the page is for and who uses it, or state your assumption, rather than retrofitting a no-JS path into an app that doesn't need one.

## How to apply it

For any feature, answer three questions before writing code:

1. **Baseline.** What does the server HTML do alone? Usually a link (GET) or a form (GET to read, POST to write) and a full page response.
2. **Enhancement.** What does JS improve? Usually: no reload, keep scroll and focus, show progress, update in place.
3. **Switch.** What flips the page from baseline to enhanced, and when? It should be the moment the enhancement is wired up. If the script never runs, the baseline stays.

Then hold to these rules:

- **One server endpoint, two callers.** The enhanced path calls the same route, loader or action as the baseline, so they can't drift apart.
- **Real URLs.** State worth sharing or reloading (filters, search query, page 2, the selected item) lives in the URL so the server can render it.
- **Real elements.** Links navigate, buttons act, forms submit. A `<div onClick>` has no baseline. A `<button type="button">` does nothing without JS, so either put the action in a form or only render the button after hydration.
- **Remove the fallback only once the enhancement is live.** Don't hide the submit button, ship a spinner, or render an empty shell in the server HTML on the assumption that JS will arrive.
- **Don't hide content behind JS.** Accordion panels, inactive tabs, modal content and "read more" text are in the HTML and reachable without JS.
- **Feature-detect, don't browser-sniff.** `'clipboard' in navigator`, `CSS.supports(...)`, `@supports`, `typeof HTMLDialogElement`.
- **When JS replaces a page load, it inherits the page load's jobs.** A full load gives a screen reader announcement, a focus reset, a history entry and scroll restoration. For each one you remove, decide deliberately: where focus goes (or that it stays), what gets announced (a `role="status"` live region), whether the URL changes (`pushState` for new views, `replaceState` for refinements), and where scroll ends up.

## Patterns and recipes

Read `references/patterns.md` for the component in front of you. Each entry gives the baseline HTML, the enhancement, and what usually goes wrong. It covers:

- Forms, mutations and optimistic UI; validation
- Controls that load data on change (dependent selects, filter dropdowns) → deep dive in `references/on-input.md` (WCAG 3.2.2 On Input)
- Search and filters; pagination, load more, infinite scroll
- Navigation and client-side routing
- Disclosure and accordions, tabs, dialogs, menus, carousels
- Toggles and one-click actions (like, follow, archive, delete)
- Clipboard, share and other API-dependent buttons
- Date, file and autocomplete inputs; images and media

CSS fallbacks, no-JS styling hooks and motion: `references/css.md`.

Framework recipes:

- React Router v7 / Remix → `references/react-router.md`
- Next.js App Router → `references/nextjs.md`
- No framework, or a server framework (Rails, Django, Laravel, Express, HTMX) → `references/vanilla.md`

## Hydration traps (React, Vue, Svelte and friends)

- Controls in the SSR HTML are clickable before hydration. If the handler needs JS, the click is lost. Give the control a baseline (a form or link), or render it absent or disabled until hydrated.
- Gate client-only UI on a hydrated flag, not `typeof window`, which causes hydration mismatches. In React: `useSyncExternalStore(() => () => {}, () => true, () => false)`.
- Don't decide markup from `localStorage`, `matchMedia` or cookies read on the client during render. The server can't see them, so the first client render won't match. Render the server-safe version and enhance in an effect.

## Reviewing

Flag:

- A clickable `div` or `span`; an `<a>` with no `href`, `href="#"` or `javascript:` acting as a button
- An `onClick` that is the only way to do something, with no form or link baseline
- `onSubmit` that calls `preventDefault()` on a form with no `action`/`method` the server handles
- `onChange` or `onInput` that reloads the page or moves focus (fails WCAG 3.2.2; see `references/on-input.md`), or that adds a history entry per change (breaks Back)
- Core content that appears only after a client fetch, with no server render
- CSS that hides content by default and relies on JS to reveal it
- Browser APIs used without detection (`navigator.clipboard.writeText`, `navigator.share`)
- Spinners or skeletons in server HTML that only JS removes
- GET links that change data (prefetchers and crawlers will follow them)

Checklist (paste into the PR):

- [ ] With JavaScript disabled, every task on the page can be completed
- [ ] Before hydration (throttled CPU and network), nothing clickable is dead
- [ ] Links have real `href`s; actions are forms, or buttons inside forms
- [ ] Shareable state is in the URL and rendered by the server
- [ ] Baseline and enhanced paths use the same server endpoint
- [ ] Fallback UI is removed only once the enhancement is wired up
- [ ] No content is hidden in a way only JS can undo
- [ ] Where JS replaces a page load: focus, announcement, URL and scroll are handled
- [ ] Browser APIs and newer CSS are feature-detected with a working fallback
- [ ] The server validates everything; client validation is a convenience

## Testing

- **No JS:** Chrome DevTools → Command menu → "Disable JavaScript". Reload and complete each task with the keyboard alone.
- **Late JS:** throttle to slow 4G with a 6× CPU slowdown and click things the moment they appear.
- **Failed JS:** in the Network panel, right-click the main bundle → "Block request URL". HTML and CSS still load; the script doesn't.
- **Screen reader:** run VoiceOver or NVDA on the enhanced path and check every in-place update is announced and focus lands somewhere sensible.
- **Automated:** run baseline end-to-end tests in a Playwright context created with `javaScriptEnabled: false`, alongside the normal suite.

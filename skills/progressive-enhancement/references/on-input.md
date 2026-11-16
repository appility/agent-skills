# Controls that load data on change (WCAG 3.2.2 On Input)

Dependent or cascading selects, filter and sort dropdowns, auto-submitting forms, jump menus.

## The rule

WCAG 2.x success criterion 3.2.2 **On Input** (Level A): changing the value of a form control must not cause a change of context — a page load, a new window, focus jumping elsewhere — unless the user was told beforehand.

Why it matters:

- On a closed `<select>`, the arrow keys change the value one option at a time. If `onChange` navigates, every arrow press reloads the page before a keyboard user reaches the option they want.
- A screen reader loses its place and starts reading a new page.
- Focus leaves the control and lands at the top of the page.

A submit button the user presses on purpose is fine: pressing it *is* a request for a new page.

## The pattern

1. **Server HTML works alone.** Render the parent control, the dependent control and a visible submit button ("Continue") in a `method="get"` form. Pressing it reloads with `?parent=value` and the server fills in the dependent control.
2. **JS upgrades it.** Once the script is running, hide the button. On `change`, fetch the dependent data in the background from the same loader. Don't push history or move focus. Replace the dependent options in place. For a field inside a larger form, leave the URL alone too. For filters and sort on a results page, updating the URL with `replaceState` (or the framework's `replace` option) is fine and makes the view shareable, because it changes nothing the user can feel.
3. **Announce the result.** A `role="status"` / `aria-live="polite"` region, present and empty from first render, gets a short message after each fetch ("2 descriptions for SUBS"). Focus is still on the parent, so without this a screen reader user won't know the second control changed.
4. **Hide the fallback only once the upgrade is wired.** The same flag that enables the fetch removes the button.

Code: `react-router.md` (dependent select example) or `vanilla.md`.

## Where implementations break

- **Live region text must change every time.** A live region speaks only when its content changes. "2 descriptions available" for two different parents is identical text, so switching between them is silent (in React the state update is skipped entirely). Include the chosen value in the message. `change` only fires when the value differs, so the text always will too.
- **Don't announce on first load.** Render the status region empty and set it only after a background fetch; a full page load is already announced.
- **The live region must exist before its text changes.** Inserting the region and its text in the same render is often not announced.
- **Clear stale child values.** In React, key the child `<select>` on its option list; in vanilla JS, replace its children.
- **Race conditions.** Fast arrow presses fire several requests. Cancel the previous one (React Router fetchers do; otherwise `AbortController`).
- **Slow or failed fetches.** Disable the child while loading if a stale pick matters. On error, say so in the status region and show the fallback button again.
- **GET submits the whole form.** On the no-JS path every other field lands in the URL. Keep the dependent pair in its own small form, or make the loader ignore unrelated params.
- **Validate on the server.** A crafted request can send any parent/child combination.

## Reviewing

Fail 3.2.2:

- `onChange` / `onInput` / `onBlur` on an input, select, radio or checkbox that triggers a full page load (`window.location = …`, `form.submit()`, `requestSubmit()` without interception), opens a window (`window.open()`), or moves focus elsewhere.
Not a 3.2.2 failure on its own, but flag it anyway: client-side `navigate()` / `router.push()` in a change handler. Each change adds a history entry, so Back steps through every option tried, and many routers reset scroll. Use the replace option and keep scroll, or fetch in the background.
- Jump-menu selects that go to a URL when an option is picked.
- Auto-submitting filter or sort forms with no button and no prior warning.

If the page clearly warns beforehand that changing the control loads a new page, 3.2.2 is technically met, but the pattern above is still the better experience.

## Testing

- With JS disabled, complete the form with the keyboard only.
- With JS on, tab to the parent select and press the down arrow repeatedly: the URL must not change and focus must stay put.
- With a screen reader, change the parent and confirm one announcement per change, including between two values that return the same number of options.
- In automated tests, assert that changing the parent doesn't navigate and that the status region's text changes.

# Progressive enhancement

Progressive enhancement gets mentioned a lot and understood very little, and that includes plenty of senior developers. The usual definition is something like "the site should work with JavaScript turned off". Hardly anyone turns JavaScript off, so the idea gets filed under nice-to-have and forgotten.

That's the wrong way to look at it. Progressive enhancement is about the order you build things in. First the server sends HTML that does the job on its own. Then JavaScript makes it nicer to use. Every user goes without JavaScript at some point anyway: on a slow connection before the bundle arrives, when a CDN has a bad minute, when an extension blocks a script, or when an error somewhere else on the page stops the rest from running.

Get the order wrong and things fail quietly. Buttons do nothing for the first few seconds. Forms can't be submitted until hydration finishes. A dropdown reloads the page every time a keyboard user presses an arrow key. You won't see any of it on a fast laptop with everything cached, and that's where most of us test.

## An example

Here's a dependent select. You pick a benefit code, and the Description list shows the options for that code. There are three parts to it:

- **Baseline.** With no JavaScript, there's a Continue button. Pick SUBS, press Continue, and the form does a GET to `?code=SUBS`. The server sends the page back with Description filled in.
- **Enhancement.** With JavaScript, Description fills in as soon as you change the code. The URL doesn't change, focus stays on Benefit code, and a line under the select tells screen readers how many descriptions there are now.
- **Switch.** Continue only goes away once the script has loaded and the background update is set up. If the script never loads, Continue stays and the form still works.

Both use the same loader. The JavaScript only changes how the loader gets called, not what it sends back. There's a bonus too: built this way, the form passes WCAG 3.2.2 On Input, which the usual reload-on-change approach fails. My code uses React Router, but you can do the same in any framework that renders on the server.

## Two paths, one loader

![Both paths call one loader; only Continue reloads the page. Without JavaScript: picks a code, presses Continue, the loader runs, and a new page comes back filled in with focus at the top. With JavaScript: changes the code, fetcher.load runs the same loader, and Description updates while focus and URL stay put.](two-paths-one-loader.svg)

If you press Continue, you've asked for a new page, so getting one is fine. If you only change the code, you haven't asked for anything, so the page shouldn't move.

## Why not just reload on change?

Because the user has only picked something. They haven't asked to go anywhere, and the page moves them anyway. That causes real problems:

- In many browsers, especially on Windows, the arrow keys move through a closed select one option at a time, and each press fires `onChange`. A keyboard user gets a page reload before they reach the option they want.
- A screen reader loses its place and starts reading out a whole new page. The user then has to find their way back to the form.
- Focus jumps to the top of the page, so everyone has to tab back down.

## The code

It's one route module and a small hook. The parts that matter:

1. **One loader.** It reads `code` from the URL and returns the descriptions for it. Continue (a full page load) and `fetcher.load` (a background load) both call it, so they can't disagree.
2. **`useFetcher`, not navigation.** `fetcher.load('?code=SUBS')` runs the loader without touching the URL, the history or focus. If you press the arrow keys quickly, each new load cancels the one before, so old results don't turn up late.
3. **`useHydrated` decides the mode.** It's `false` in the server HTML and `true` once React has hydrated. The same flag hides Continue and turns on the fetch, so you never get both, or neither.
4. **Fetched data wins.** `fetcher.data?.descriptions ?? data.descriptions` uses the background result if there is one, and the server-rendered list if not.
5. **A `key` on Description.** When the options change, the select is rebuilt, so a description picked for the old code doesn't hang around.

```tsx
import { useEffect, useState } from 'react';

import { Form, useFetcher, useLoaderData } from 'react-router';

import useHydrated from '~/hooks/useHydrated';

const descriptionsByCode: Record<string, string[]> = {
  EDUC: ['Educational Assistance', 'Educational Assistance CL1A'],
  SUBS: ['Subs and Professional Fees', 'Subscriptions and Fees']
};

// Name the code, so two codes with the same count still produce different text.
const announcementFor = (code: string, count: number) => {
  if (!code) return 'No benefit code chosen';
  return `${count} ${count === 1 ? 'description' : 'descriptions'} for ${code}`;
};

// Same loader for Continue (full page) and for fetcher.load (page stays put).
export const loader = ({ request }: { request: Request }) => {
  const code = new URL(request.url).searchParams.get('code') ?? '';

  return { code, descriptions: descriptionsByCode[code] ?? [] };
};

export default function AddBenefit() {
  const data = useLoaderData<typeof loader>();
  const fetcher = useFetcher<typeof loader>();
  const isHydrated = useHydrated();
  const descriptions = fetcher.data?.descriptions ?? data.descriptions;
  const [announcement, setAnnouncement] = useState('');

  // Only after a background fetch. A full page load is already announced.
  useEffect(() => {
    if (!fetcher.data) return;
    setAnnouncement(announcementFor(fetcher.data.code, fetcher.data.descriptions.length));
  }, [fetcher.data]);

  return (
    <Form method="get">
      <label htmlFor="code">Benefit code</label>
      <select
        defaultValue={data.code}
        id="code"
        name="code"
        onChange={(event) => {
          if (!isHydrated) return;
          const nextCode = event.currentTarget.value;
          fetcher.load(nextCode ? `?code=${encodeURIComponent(nextCode)}` : '?');
        }}>
        <option value="">Choose a benefit code</option>
        <option value="EDUC">EDUC — Education</option>
        <option value="SUBS">SUBS — Subscriptions</option>
      </select>

      {isHydrated ? null : <button type="submit">Continue</button>}

      <p aria-live="polite" role="status">
        {announcement}
      </p>

      <label htmlFor="description">Description</label>
      <select id="description" key={descriptions.join('|')} name="description">
        {descriptions.map((description) => (
          <option key={description} value={description}>
            {description}
          </option>
        ))}
      </select>
    </Form>
  );
}
```


`useHydrated` is only a few lines. During hydration `useSyncExternalStore` returns the server value, so the server HTML and the first client render match:

```ts
import { useSyncExternalStore } from 'react';

const subscribe = () => () => {};

export default function useHydrated() {
  return useSyncExternalStore(subscribe, () => true, () => false);
}
```


## Telling screen readers

Focus is still on Benefit code, so a screen reader won't notice that Description has changed. A status region tells it.

- `role="status"` with `aria-live="polite"` gets read out when its text changes, once the screen reader has finished what it was saying.
- Leave it empty in the server HTML and only fill it after a background fetch. A full page load is announced anyway, so you'd hear it twice.
- Put the region in the page from the start. If you add the region and its text at the same moment, it often isn't read out.

**Watch out for identical messages.** A live region only speaks when its text changes. EDUC and SUBS both have two descriptions, so "2 descriptions available" is the same text for both. Switch from one to the other and React doesn't even update the DOM, so nothing is read out. Putting the code in the message ("2 descriptions for SUBS") fixes it, because `onChange` only fires when the value actually changes.

## Things that catch people out

- **Index routes.** In React Router, a fetcher pointed at an index route needs `?index` in the query. Without it you get the parent route's loader.
- **Other fields.** Continue submits the whole form as a GET, so every other field ends up in the URL. Put the two selects in their own small form, or have the loader ignore anything it doesn't need.
- **Slow responses.** Description shows the old options until the fetch comes back. If that might take a while, disable it while `fetcher.state` is `loading` so nobody picks a stale value.
- **Errors.** If the fetch fails, say so in the status region and bring Continue back, so the form still works.
- **Validation.** The final submit still needs checking on the server. Someone can send any code with any description.

A checklist for code review:

- [ ] No `onChange`, `onInput` or `onBlur` on a form control navigates, submits or opens a new window
- [ ] With JavaScript disabled, the form can be completed using a visible submit button
- [ ] With JavaScript on, changing the parent control keeps focus and the URL where they are
- [ ] Both paths call the same loader
- [ ] The no-JS button is only hidden once the background update is wired up
- [ ] A `role="status"` region exists from first render, starts empty, and gets new text on every change
- [ ] Stale child values are cleared when the parent changes
- [ ] Tested with arrow keys on a closed select and with a screen reader

## The same idea elsewhere

The dropdown is one case. Most interactive features split the same way: a link or a form that works, and JavaScript that saves the reload.

| Feature | Baseline | Enhancement |
| --- | --- | --- |
| Like button | A tiny `<form method="post">` around the button. The server records the like and redirects back to the post | Submit in the background, flip the button straight away, put it back if the server says no |
| Pagination | `<a href="?page=2">Next page</a>` to a real page 2 | Fetch page 2, append the items, update the URL, move focus to the first new item |
| Search | A GET form with a search button. Results are rendered from `?q=` | Search as you type, update the URL with `replaceState`, announce the number of results |
| Delete confirmation | A link to a confirmation page with a form on it | Load the same content into a `<dialog>`, return focus to the button when it closes |

In each case the baseline is the thing you'd build if JavaScript didn't exist, and the enhancement calls the same server route.

## Where to start

Next time you build something interactive, write the link or form version first and check it works with JavaScript turned off in DevTools. Then add the script. It's usually less work than it sounds, and you find out early whether the server can actually do the job on its own.

It isn't always worth it. A canvas editor, a multiplayer game or an internal tool on managed laptops can skip the no-JS version. Even then, real links and buttons, native elements and not losing clicks before hydration still pay off.

I've turned all this into an agent skill, so your AI coding assistant applies these patterns when it writes or reviews your code. It works with Claude and GitHub Copilot. It covers about fifteen common components, has recipes for React Router, Next.js and plain JavaScript, includes the review checklist above, and knows when to hold back. It's on GitHub: [agent-skills](https://github.com/YOUR-USERNAME/agent-skills).
